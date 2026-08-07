# Upgrade path reliability and drift detection

Read-only review. Scope: `llz upgrade`, the copier update contract, the
managed/owned classification and its guards, `llz render --check` / `llz drift`,
the umbrella-tag pinning contract, Argo chart pins, and apl-core version pinning.

Every claim below cites a file and line read in full this session. Where a
tradeoff is already recorded in an ADR, a design doc, or the manifest's own
prose, it is labelled **[documented tradeoff]** and is not counted as a new
finding.

---

## 0. The mechanism, stated once (so the findings below are readable)

`llz upgrade` (`tools/cmd/llz/commands.go:747-886`) is a five-stage local
command. Nothing in it touches a cloud API:

1. `requireCopier` (commands.go:750) — copier must be on PATH.
2. `snapshotUpgradeOwned` (commands.go:779 → `upgrade_policy.go:22-50`) — copy
   every `owned`-classified worktree file into a temp dir.
3. `copier update --trust --defaults --vcs-ref <ref> --data llz_version=<ref>`
   (commands.go:784, argv at commands.go:92-98).
4. `applyUpgradeManifestPolicy` (commands.go:788 → `upgrade_policy.go:154-175`) —
   restore the `owned` snapshot, then render a **second, clean copier scaffold**
   of the target ref into a temp dir and copy every `managed` file from it over
   the instance (`upgrade_policy.go:177-244`).
5. `applyTemplateRemovals` (commands.go:797), then four gates and a re-render:
   answer-regression (commands.go:821), conflict markers (commands.go:835),
   `renderAfterUpgrade` (commands.go:856), CI-image skew warning
   (commands.go:869), optional `--commit` (commands.go:881).

The ownership map is `instance-template/.template-manifest`; the class → action
table is `ci_template_manifest.go:63-70`:

| class | action | copier-fenced | digest-locked |
|---|---|---|---|
| `managed` | `upgradeOverwrite` — re-copied from a clean render | no | **yes** |
| `merge` | `upgradeMerge` — copier's 3-way merge stands | no | no |
| `owned` | `upgradeRestore` — instance bytes put back | yes | no |

---

## Q1 — What does `llz upgrade` do to a live instance repo?

**Answer.** It rewrites the whole scaffold in place on the local working tree.
`managed` files are overwritten byte-for-byte from a clean render of the target
tag (`upgrade_policy.go:218-244`); `merge` files keep copier's 3-way merge
result; `owned` files are restored to their pre-update bytes
(`upgrade_policy.go:58-73`). `.template-removals` then deletes/untracks paths the
new template dropped (`instance-template/.template-removals`, applied at
commands.go:797 — copier itself never deletes). Then `llz render` rewrites every
committed `apl-values/<env>` kustomization because their `?ref=` is the pin that
just moved (commands.go:899-910).

An adopter's local edit to a `managed` file is **silently discarded** — the
overwrite pass does not diff or warn (`upgrade_policy.go:228-242` copies
unconditionally). That is the intended contract, and it is guarded, imperfectly:

### F1 — The digest lock covers ~half the surface it is described as covering; the rest is declared-and-unmitigated

- Evidence: `ci_managed_lock.go:142-165` (`lockableFiles`) skips any
  `digestLocked` file containing the copier token `<@`
  (`ci_managed_lock.go:156-159`). Those are written into the lock **header as
  comments** (`ci_managed_lock.go:239-249`), not as digests.
- Evidence: the lock is a projection of `digestLocked` classes only
  (`ci_template_manifest.go:64-69`) — `merge` is not digest-locked, so nothing
  detects a hand-edit to a `merge` file. `.template-manifest` says this out loud
  for the three token-free caller stubs: *"a hand-edit there survives upgrades
  silently"* — **[documented tradeoff]**, with the stated migration path
  (reclassify to `managed` and the lock picks them up).
- Why it matters: a `merge` file that an adopter hand-edits will collide on a
  future release, and a tokenful `managed` file that an adopter hand-edits is
  overwritten with no CI signal at all. The lock's own header calls this "the gap
  … auditable rather than invisible", which is honest but is not detection.
- Recommendation: the honest fix is not more digests — it is to make the
  tokenful-`managed` set empty. Add a lint step that fails when a `managed` file
  gains a `<@ … @>` token (the class contract says the template owns it
  outright; a per-instance token means it should be `merge`). Cheap, and it
  converts a growing declared gap into a bounded one.

### F2 — On a `copier update` failure the `owned` snapshot is discarded without being restored

- Evidence: commands.go:784-786 returns on copier error; the restore lives in
  `applyUpgradeManifestPolicy`, only reached at commands.go:788 when copier
  succeeded. `defer owned.cleanup()` (commands.go:782 →
  `upgrade_policy.go:52-56`) then `RemoveAll`s the temp copy.
- Bound: the practical blast radius is small. `checkCopierFencing`
  (`ci_template_manifest.go:215-265`) fails CI unless every *shipped* `owned`
  file is in copier's `_skip_if_exists`/`_exclude`, and `copier.yml:73-83` lists
  them. So copier should not have written those files at all. The exposure is
  the residue: partially-written `managed`/`merge` files from a copier run that
  died mid-write, with the tree left at an indeterminate ref.
- Why it matters: the code's own comment (commands.go:748-749) says "a failed
  `llz upgrade` is mid-flight" — but nothing acts on that. There is no cleanup,
  no marker, and no instruction.
- Recommendation: move the restore into a `defer` that runs on the error path
  too, and on any failure after step 3 print the exact recovery command
  (`git checkout -- . && git clean -fd` when the tree was clean going in). One
  branch and three lines of prose.

### F3 — The managed overwrite is not transactional

- Evidence: `overwriteManagedFromScaffold` (`upgrade_policy.go:218-244`) returns
  `(count, err)` mid-loop on the first copy failure; the caller wraps it and
  returns (commands.go:788-790). Files copied before the error stay copied.
- Why it matters: a disk-full or permission error partway through leaves the
  instance with an arbitrary mix of old and new `managed` files, and the pin
  already advanced (copier rewrote `.copier-answers.yml` in step 3).
- Recommendation: write to a staging dir and rename in, or at minimum record the
  files already overwritten and name them in the error so a human knows the mix.

---

## Q2 — Is the upgrade atomic / reversible? Dry-run? Rollback? Does it reach the LIVE cluster?

**Atomic:** no (F3). **Reversible:** no — see F4. **Dry-run:** exists, but is
close to vacuous — see F5. **Reaches the live cluster:** yes, indirectly but
reliably — see F6, which is the finding to lead with.

### F4 — There is no rollback path, in code or in docs

- Evidence: `grep -rni 'rollback|downgrade|revert' tools/cmd/llz/*.go` (non-test)
  returns only unrelated hits — Argo self-heal comments, an mTLS ADR note, an
  `assert-adopter-pin` phrase. There is no `llz downgrade`, no `--to`, no saved
  pre-upgrade state.
- Evidence: `docs/adopter-guide.md:222-252` documents the upgrade in detail and
  names no rollback. `terraform-modules/RELEASING.md:39` states tags are
  immutable and "to release a change, cut a new one" — i.e. the intended recovery
  is roll-*forward*.
- Evidence: the one rollback that IS documented is at the cluster layer and is
  destructive: `docs/apl-core-migration-runbook.md:201-210` — "the rollback path
  is **recreate the LKE-E cluster**".
- Why it matters: `llz upgrade --ref <older-tag>` is not a supported downgrade —
  copier's 3-way merge against an older template is untested (see F11), and
  `.template-removals` is only ever additive (files deleted on the way up are not
  restored on the way down). An operator who upgrades into a broken release has
  `git revert` of the upgrade commit as their only lever, and that lever does not
  undo the GitHub repo *variables* the upgrade told them to re-pin (F8).
- Recommendation: state the supported recovery explicitly in
  `docs/adopter-guide.md` §4 — "recovery from a bad upgrade is `git revert` of
  the upgrade commit plus re-pinning TF_IMAGE/KUBE_IMAGE to the previous
  commit; downgrading the template ref is not supported" — and add a gate only
  if downgrade is meant to work.

### F5 — `--dry-run` cannot preview the upgrade diff

- Evidence: `run()` returns before exec on dry-run (commands.go:196-202), so
  `copier update` never runs. `applyUpgradeManifestPolicy` then prints two
  counts and returns (`upgrade_policy.go:154-159`): "would restore N owned
  file(s)", "would overwrite managed files". `runUpgrade` returns at
  commands.go:800-802 before the answer gate, the conflict gate, the re-render
  and the diffstat.
- Why it matters: the flag reads as "show me what this upgrade will change" and
  delivers a file count with no filenames and no diff. `printUpgradeSummary`
  (commands.go:957-971) — the one place the churn is visible — never runs on
  dry-run. For an upgrade whose main risk is unreviewable churn, this is the
  wrong thing to be cheap about.
- Recommendation: make `--dry-run` render the target scaffold into a temp dir
  (`renderUpgradeScaffold` already does exactly this,
  `upgrade_policy.go:177-194`) and print a real diffstat of clean-render vs
  instance for the `managed` set. No copier *update* needed; the machinery is
  already there.

### F6 — The upgrade DOES change what the live cluster runs, with no plan and no approval — and the Terraform half of the same upgrade is gated

This is the asymmetry worth acting on.

- The pin the upgrade moves is read at render time into every committed
  `apl-values/<env>` kustomization as a **kustomize remote base**:
  `clusterspec.RemoteBase` builds
  `github.com/akamai-consulting/lke-landing-zone//platform-apl/<path>?ref=<pin>`
  (`tools/internal/clusterspec/kustomize.go:36,51-52,101`).
- `renderAfterUpgrade` (commands.go:856, 899-910) rewrites those files on every
  upgrade — deliberately, because skipping it "left a live instance running
  three releases behind on what ArgoCD DEPLOYS" (commands.go:843-852).
- Argo syncs them **automatically, with prune and selfHeal**:
  `platformBootstrapApplicationManifest` sets
  `syncPolicy.automated{prune:true, selfHeal:true}`
  (`ci_bootstrap_cluster_manifests.go:215-219`) on `apl-values/<env>/manifest`
  at `targetRevision: o.appsRepoRevision` (`:208`, default spec-then-`main`,
  `ci_bootstrap_cluster.go:148`). The carved Apps do the same
  (`kustomize.go:454-459`), as does the `instance-custom` ApplicationSet
  (`kustomize.go:342-347`), and `llz-secret-store` is pinned directly to the
  template ref (`ci_bootstrap_cluster_manifests.go:242`) — which is why
  `ci_bootstrap_cluster.go:386` notes its targetRevision "changes every upgrade".
- Therefore: **`llz upgrade --commit` + push to the default branch is a
  production deploy of a new platform-apl manifest tree.** No plan, no diff
  review step, no approval — the only gate is whatever branch protection the
  adopter configured. `kustomize.go:225-228` names this for the escape hatch and
  accepts it — "there is no pin to roll back to, and a commit to the default
  branch deploys itself. The gate is that branch's review" — **[documented
  tradeoff]**, but recorded for `kubernetes-custom/`, not for the
  upgrade-moves-the-platform-manifests case.
- Meanwhile the Terraform half of the *same* upgrade is gated three ways: applies
  are dispatch-only (`llz-terraform.yml:265-271, 421-427`), run under
  `environment: infra-${{ inputs.region }}` (`:277, :431`), and PRs get a plan
  posted (`:142-201`).
- Why it matters: an operator reasonably reads "environment approval gates infra
  changes" (`docs/workflows/llz-terraform.md:246-250`,
  `docs/playbooks/operator-onboarding.md:245`) and concludes the whole upgrade is
  gated. Half of it is. The Argo half deploys on merge.
- Recommendation: (a) say this in `docs/adopter-guide.md` §4 in one sentence —
  "merging an `llz upgrade` commit deploys the new platform manifests; Argo syncs
  them automatically"; (b) add `llz upgrade --diff-apl-values` (or fold it into
  F5's dry-run) so the ArgoCD-visible delta is reviewable *before* commit, which
  is the only place a human can see it.

### F7 — `commitUpgrade` runs `git add -A` with no clean-tree precondition

- Evidence: commands.go:1012-1030 — `git add -A` then commit with message
  `chore(template): upgrade X → Y`. The only guard is "is the tree empty"
  (commands.go:1013).
- Why it matters: an operator with unrelated in-flight edits gets them folded
  into a commit labelled as a template upgrade — the exact commit whose whole
  purpose (commands.go:878-880) is "the operator reviews ONE diff".
- Recommendation: capture `git status --porcelain` before step 3 and refuse
  `--commit` if the tree was dirty going in (or stage only the paths the upgrade
  touched). Roughly ten lines.

---

## Q3 — What can a version bump break on a live instance?

### F8 — Three things move on a template bump; one of them the upgrade cannot reach

Enumerated from the code:

1. **Terraform module `?ref=`** — the generated TF roots carry
   `git::…?ref=<pin>` (`render.go:36-54`, `tfrootTokens`). The roots are
   gitignored build artifacts regenerated before every terraform op
   (`render.go:6-11`, `.template-manifest` `terraform-iac-bootstrap/.gitignore`
   entry), so a bump silently changes the module source consumed by the next
   apply. See F12 for what stops that from being destructive: nothing
   structural.
2. **The kustomize remote base** — F6.
3. **`TF_IMAGE` / `KUBE_IMAGE`** — derived from the pin, but stored as GitHub
   repo *variables*. `llz upgrade` explicitly does not push them
   (`reportCIImageSkew`, commands.go:973-1007) and warns instead. Left unfixed,
   the first pipeline run after every upgrade fails `assert-image-fresh`
   (commands.go:1006). **[documented tradeoff]** — the rationale is stated at
   commands.go:976-982 (a local command should not need credentials), and
   `docs/adopter-guide.md:233-241` repeats the remediation.
- Recommendation for (3): this is the one skew that is deterministic on *every*
  upgrade and is not self-correcting. It is worth a `llz upgrade --push-vars`
  opt-in that runs the two `gh variable set` calls it already prints, gated on
  `--yes` like every other cloud-mutating verb (`runGated`, commands.go:207).

### F9 — The two pins that record the same fact still can and do disagree

- Evidence: `pin_coherence.go:14-20` documents a **live** instance at
  `_commit: v0.0.33` / `llz_version: v0.0.34`, "DEPLOYING v0.0.34 manifests from a
  v0.0.33 scaffold", silent because both values are individually well-formed.
- Guard: `assertPinCoherence` (`pin_coherence.go:43-60`) now fails `llz lint`,
  but only when **both** pins are exact release tags and differ
  (`pin_coherence.go:49`) — a deliberately narrow scope, argued at
  `pin_coherence.go:22-25`.
- Why it matters: this is the concrete precedent for F6's blast radius. It is
  fixed; it is cited here because it is the empirical proof that a repo-only
  pin change reached the cluster.
- Status: **[documented tradeoff]** for the narrowness; no action needed.

### F10 — Terraform-side destructive-change scars are documented, not gated

- Evidence: `docs/lessons-learned.md:188-191` — "**LKE pool inline-drift destroys
  the pool.** … without `ignore_changes = [pool]` a refresh plans to null the
  pool and Linode interprets that as 'delete the pool.'"
- Evidence: `docs/adr/0005-managed-app-platform.md:56` — `apl_enabled` is
  `ForceNew`: "existing clusters must be recreated". Any change that touches it
  is a cluster replacement. **[documented tradeoff]**.
- Evidence: `docs/lessons-learned.md:211-218` — the VPC/`vpc_id` binding bug that
  leaked a VPC per cycle and hung cluster creation at quota.
- Why it matters: these are exactly the shapes a module-ref bump can reintroduce,
  and see F12 — nothing inspects the plan for them.

### F11 — apl-core: the version pin in the spec is **check-only and never deployed**, and the migration runbook still tells operators to set it

- Evidence: `EffectiveAplChartVersion` (`clusterspec/aplversion.go:43-48`) has
  **no non-test consumer** — `grep -rn 'EffectiveAplChartVersion' tools/
  --include='*.go'` returns only its own definition site. The pin is read by
  `resolveAplChartVersion` → `assertAplVersion` (`ci_assert_apl_version.go:73-97`)
  and by `llz ci validate-apl-values --chart-version` for a `helm template`
  schema check (`ci_apl_schema.go:81, 117-121, 169`). Both are *checks*.
- Evidence: nothing helm-installs apl-core.
  `ci_bootstrap_cluster.go:794-796`: "A managed cluster has no
  customer-`helm`-upgradeable `apl` release (helm upgrade → 'apl has no deployed
  releases')". ADR 0005's status line is explicit: "**managed is the ONLY mode.**
  The pivot is complete: LLZ no longer self-installs apl-core"
  (`docs/adr/0005-managed-app-platform.md:3-5`), and its trade-off table records
  "Linode owns apl-core version/lifecycle — LLZ loses 'pin our own apl-core'"
  (`:56`) — **[documented tradeoff]**.
- Evidence: `docs/adopter-guide.md` states the shipped behaviour correctly —
  `aplChartVersion` "optional | **Omit it.** … bootstrap does not consume this
  field, so a pin deploys nothing."
- **Conflict.** `docs/apl-core-migration-runbook.md:32-35` instructs the operator
  to "update `spec.cluster.bootstrap.aplChartVersion` in each
  `environments/<env>.yaml` to match" the chart they found in the helm repo — i.e.
  to treat it as a deploy lever. The same document, `:82-88`, says
  `llz ci bootstrap-cluster` "**Helm-installs apl-core** and blocks until the
  `apl-operator` deployment is Ready", contradicting `ci_bootstrap_cluster.go:794`
  and ADR 0005. `:68-78` also points at
  `instance-template/terraform-iac-bootstrap/cluster` and tells the operator to
  edit `<env>.tfvars` and run `terraform apply` there — a *template* path that
  does not exist in a rendered instance, where the TF roots and tfvars are
  gitignored generated artifacts (`render.go:6-11`).
- Why it matters: this is the document an operator opens to perform an apl-core
  upgrade. Following it produces a spec edit that deploys nothing, a shell
  command against a path that does not exist, and a mental model in which LLZ
  controls the apl-core version — which it does not. The v6 migration design
  already caught the same class of staleness in itself
  (`docs/designs/apl-core-v6-migration.md:257-265`: "**This step is
  historical.** … following this step would have edited a file that feeds
  nothing"). The runbook did not get that treatment.
- Recommendation: apply the v6-design's own remedy to
  `docs/apl-core-migration-runbook.md` — mark Phase 0 bullet 5 and Phase 1
  steps 1-3 historical, replace with the managed reality (Linode owns the
  version; `aplChartVersion` only moves the assert floor and the schema check;
  the operator-side actions are `llz ci prepare-apl-upgrade` and the lab
  checklist). Note there is no gate that would have caught this — see the
  recommendation under F16.

---

## Q4 — Is there a gate that vX → vY is tested?

### F12 — `llz ci upgrade-test` gates copier, not `llz upgrade`

- Evidence: `ci_upgrade_test_gate.go:253-425` scaffolds at the previous release
  tag and runs `copierUpdateArgv(to)` directly (`:353`) with stdin closed
  (`runCopier`, `:181-186`). It asserts four things: non-interactive update
  (`:364`), answers preserved (`:381`), pin advanced (`:392`), clean merge —
  markers and `.rej`/`.orig` (`:399`). It runs in CI at
  `.github/workflows/lint.yml:462` and via `make upgrade-test` (`Makefile:949-951`).
- Evidence: it deliberately reuses `copierUpdateArgv` rather than composing its
  own, and says why (`ci_upgrade_test_gate.go:160-166`) — a genuinely good
  design choice.
- **But**: `runUpgrade` is called from exactly one place — `main.go:303` — and
  from nothing else (`grep -rn 'runUpgrade(' tools/cmd/llz/*.go`). So the
  manifest snapshot/restore, the managed-overwrite pass, `.template-removals`,
  the answer-regression gate, the conflict gate, `renderAfterUpgrade` and
  `commitUpgrade` are **unit-tested in isolation and never exercised end-to-end
  against a real copier run.** The unit tests exist and are real —
  `upgrade_policy_test.go:11,44,79,93`, `upgrade_render_test.go:46,70,82`,
  `template_removals_test.go:9,45,94`, `upgrade_test.go:9,33` — but each stubs
  its own inputs.
- Why it matters: the ordering constraints in `runUpgrade` are load-bearing and
  argued at length in comments (restore-before-overwrite,
  removals-after-policy at commands.go:792-796, conflict-gate-before-render at
  commands.go:854-855). Ordering is precisely what per-function unit tests
  cannot check.
- Recommendation: extend `llz ci upgrade-test` to invoke `llz upgrade` (the
  binary under test) in the built instance instead of `copier update` — it
  already has a real scaffold at a real prior tag in a real git repo, which is
  all `runUpgrade` needs. That converts four copier-layer assertions into a full
  day-2 gate for a small diff.

### F13 — `release-e2e` is greenfield-only; no release exercises an upgrade of a live cluster

- Evidence: `release-e2e-lane.yml` jobs are `llz-functional`, `instantiate`,
  `provision`, `teardown`, `gate` (`:149, 227, 247, 282, 335`). The lane header
  describes the flow (`:16-35`): instantiate force-pushes a fresh scaffold,
  provision applies `module=all` from zero, teardown destroys. There is no
  "instantiate at N-1, upgrade to N" step.
- Evidence: the release gate is convention, not mechanism —
  `terraform-modules/RELEASING.md:87-91`: "e2e can't *mechanically* block
  promotion (the gate is convention + the promote click)". **[documented
  tradeoff]**.
- Evidence: `ci_upgrade_test_gate.go:1-11` names this gap in its own header for
  the copier layer — "the SCAFFOLD path was gated and the UPGRADE path, which
  every adopter takes on day 2 … was exercised by nobody" — and closes it *for
  copier*. The cluster-level equivalent is still open.
- Why it matters: the failure modes that matter most on an upgrade — a module
  ref bump that forces resource replacement, a platform-apl manifest that no
  longer applies over the previous release's objects, an Argo App that goes
  OutOfSync-and-selfHeals-in-a-loop (the exact shape at
  `ci_bootstrap_cluster_manifests.go:196-199`) — are all invisible to a
  greenfield run by construction.
- Recommendation: a dispatch-only (not per-release, given quota:
  `release-e2e.yml:57-60` allows exactly one live cluster) `upgrade` lane that
  provisions at the previous release tag, runs `llz upgrade` to the candidate,
  pushes, waits for converge, and tears down. Even quarterly, it is the only
  thing that can see this class.

### F14 — Both upgrade gates skip silently rather than failing

- Evidence: `ci_upgrade_test_gate.go:276-279` prints "SKIPPED — copier not
  installed" and returns nil; `:301-307` prints "SKIPPED — no vX.Y.Z tag to
  upgrade from (shallow clone?)" and returns nil.
- Why it matters: both skips are individually justified (a shallow clone cannot
  invent a prior release). But a green `make lint` on a runner without copier, or
  a shallow checkout, means "the upgrade path was not checked", and reads as
  "the upgrade path is fine".
- Recommendation: make the skips fail when `CI=true` and the reason is
  fixable (copier absent in a job that is supposed to have it; shallow clone when
  `fetch-depth: 0` was intended). Or emit a `::warning::` so the skip is visible
  in the run summary rather than in step logs.

---

## Q5 — Drift detection: what is actually compared?

### F15 — `llz render --check` is repo-vs-repo; nothing compares the repo to the live cluster

- Evidence: `runRender(..., check=true)` computes `renderTargets` from the spec
  (`render.go:179-182`, `renderTargets` at `:255-300`) and calls `reportDrift`
  against the on-disk files (`render.go:195-203`), filtering out everything under
  `tfDir` because those are gitignored build artifacts (`render.go:191-198`). So
  it answers exactly one question: *do the committed `apl-values/<env>`
  artifacts match what the current spec + current pin would render?*
- Evidence: `llz drift` (`drift.go:17-85`) answers a different question again —
  it compares the instance's `.copier-answers.yml` SHA against the template
  repo's **branch head** via `git ls-remote` (`drift.go:38`). Nothing about the
  cluster.
- Evidence: there is no repo↔cluster reconciler check. Grepping
  `OutOfSync|Synced` across `tools/cmd/llz/*.go` returns assertion helpers used
  at bootstrap/converge time (`ci_assert_argo_app.go:75,129,158`,
  `ci_assert_instance_custom.go:70-105`) and diagnostics
  (`ci_diagnose_argocd.go:144`) — all invoked from the provisioning path, none
  from a scheduled job. The scheduled matrix
  (`llz-scheduled-checks.yml:74,157,222,343,443`) is: weekly cluster checks
  (OpenBao seal, cert readiness, VAP binding, Keycloak client, secret age),
  lke-admin rotation SLA, credential single-pane, PrometheusRule health, and
  monthly template drift. None asserts "every Argo Application is Synced".
- Why it matters: given F6 — Argo runs `automated + selfHeal` on everything — a
  wedged Application is the failure mode that matters, and it is exactly the one
  the repo cannot see. `ci_bootstrap_cluster_manifests.go:196-199` records a real
  instance where `platform-bootstrap` sat permanently OutOfSync with selfHeal
  re-applying in a loop (`autoHealAttemptsCount 7 on lke638381`) — found by hand.
- Recommendation: add an `assert-argo-synced`-style job to the weekly matrix in
  `llz-scheduled-checks.yml` (the assertion primitives already exist in
  `ci_assert_argo_app.go`). This is the single highest-value addition in this
  review: it is the only check that would observe a bad upgrade *after* it
  deployed.

### F16 — The template-drift job measures the wrong distance, runs monthly, and alerts nobody

- Evidence: `drift.go:38` resolves `refs/heads/<branch>` — the **moving branch
  head**, default `main` (`drift.go:18`, workflow input at
  `llz-scheduled-checks.yml:466-468`). An instance pinned to the newest released
  tag is reported as "behind N commits" for every unreleased commit on main.
- Evidence: report-only by default — `--strict` is opt-in
  (`drift.go:81-84`, workflow input `drift_strict` at
  `llz-scheduled-checks.yml:15-16`, defaulted false at
  `instance-template/.github/workflows/scheduled-checks.yml:31`).
- Evidence: monthly cron only — `if: … github.event.schedule == '0 7 1 * *'`
  (`llz-scheduled-checks.yml:452`).
- Evidence: the output is a `::warning::` annotation (`drift.go:77`) and a
  `GITHUB_STEP_SUMMARY` table (`drift.go:108-134`). There is no issue created, no
  Slack/webhook, no alert rule — unlike the credential lane, which routes through
  Prometheus (`llz-scheduled-checks.yml:322-324`, `llz ci alert-eval --strict`).
- Why it matters: an adopter three releases behind gets a green monthly job with
  a warning annotation nobody reads. Compare `assert-adopter-pin`
  (`ci_assert_adopter_pin.go:6-17`), which exists precisely because "green e2e
  the whole time" hid a real adopter break.
- Recommendation: (a) compare against the latest **release tag** rather than the
  branch head — `latestReleaseFn` (commands.go:47) already implements the right
  rule and would make "behind" mean "there is a release you have not taken";
  (b) raise the cadence to weekly, it is one `git ls-remote`; (c) give it an
  output an operator sees — the cheapest is `gh issue create` on first detection,
  idempotent by title.

---

## Q6 — Terraform module ref bumps and destructive plans

### F17 — Nothing inspects a plan for destroys or replacements; the only human gate is an instance-configured GitHub Environment setting that `llz` neither sets nor verifies

- Evidence: `llz ci tf-plan` (`ci_tfplan.go:58-99`) tees plan output to a file,
  retries once on a control-plane API flake (`:72-77`), and appends the last 80
  lines to the step summary (`:85, 91-99`). It parses nothing. There is no
  destroy count, no "forces replacement" detection, no threshold.
- Evidence: on the apply path, plan and apply are consecutive steps in **one
  job** — `llz-terraform.yml:505-509` (plan → `tfplan.bin`) then `:511-519`
  (`llz ci tf-apply --plan tfplan.bin`). Nothing between them.
  `ci.go:492-517` confirms `tf-apply` applies a saved plan with self-heal retries;
  it has no destructive-change guard either.
- Evidence: the asserted control is GitHub Environment required reviewers on
  `infra-<region>` (`llz-terraform.yml:277, 431`;
  `docs/workflows/llz-terraform.md:246-250`, "each have required reviewers;
  approval is logged … in the GitHub Deployments API audit trail";
  `docs/workflows/llz-breakglass-openbao.md:49-50`).
- **But `llz` only configures the deployment-branch policy, never reviewers.**
  `lockInfraEnvBranchPolicy` (`branchpolicy.go:36-128`) restricts the environment
  to `main`, and deliberately sends `reviewers` **only if already present**
  (`:91-93`) to avoid a 422 on repos without a paid plan (`:79-84, 143-147`). On
  a private repo without GitHub Pro/Team/Enterprise the whole protection call
  fails and is downgraded to a warning (`errEnvProtectionUnsupported`, `:31,
  106, 120`). Nothing anywhere asserts that reviewers exist.
- Why it matters: for a module-ref bump — the case where a plan can be
  destructive (F10: the `ignore_changes = [pool]` scar, `apl_enabled` being
  `ForceNew`) — the review depends entirely on a human reading an 80-line plan
  tail in a step summary, under an approval gate that may not be configured and
  is never checked.
- Recommendation: two cheap, independent additions. (1) Have `tf-plan` run
  `tofu show -json tfplan.bin`, count `delete` and `create-then-delete` actions,
  and fail (or require an explicit `-f allow_destroy=true` dispatch input) above
  zero for the cluster root — the plan file is already saved, so this is parsing,
  not re-planning. (2) Add an `llz doctor` check that `infra-<env>` has at least
  one required reviewer, warning when the plan does not support it — so the
  documented control is at least *observed*, which is the standard
  `assert-rotation-health` and `assert-adopter-pin` already set in this repo.

### F18 — Chart `targetRevision` is out of the upgrade's scope by design, and is guarded

- Evidence: `terraform-modules/RELEASING.md:28-31` — charts version independently
  via `Chart.yaml`, publish from `publish-charts.yml`, and "an Argo
  `targetRevision:` chart pin is left untouched by the release flow."
  **[documented tradeoff]**, restated in `docs/adopter-guide.md:75-88` (Renovate
  owns chart bumps; first-party LLZ pins are `enabled: false` in `renovate.json`
  so Renovate never races `llz upgrade`).
- Guards that do exist: `llz ci chart-pin-guard` cross-checks every Argo
  `targetRevision`/`version` against the chart's own `Chart.yaml`
  (`ci_chart_pin_guard.go:11, 47-52, 138-156`), `chart-publish-check` verifies the
  pinned version is actually published (`ci_chart_publish_check.go:90, 293-304`),
  and `.github/workflows/chart-version-guard.yml:8-13` blocks an overwritten tag.
- Residual risk: those guards check *coherence and publication*, not
  *upgrade-safety*. A chart bump that changes an immutable StatefulSet field is
  not detectable statically, and with `automated + selfHeal`
  (`platform-apl/components/openbao/openbao.yaml:62-65`,
  `argoWorkflows/argo-workflows.yaml:43-46`,
  `argoEvents/argo-events.yaml:31-34`) it deploys on merge. `Chart.yaml:82` in
  `llz-openbao-platform` already notes a coupling that "must ship together with
  the openbao.yaml Argo targetRevision".
- Recommendation: no new mechanism — but the chart-bump PR template (or
  `CONTRIBUTING`) should require naming which immutable fields the bump touches,
  the same "name the gate" discipline `AGENTS.md` already imposes elsewhere.

---

## Q7 — Is a major apl-core upgrade on a live cluster documented or tested?

**Documented: partially, and partly wrongly (F11). Tested: no.**

### F19 — apl-core upgrades are Linode-triggered, and LLZ's only lever is applied eagerly and best-effort

- Evidence: `ci_prepare_apl_upgrade.go:19-25` — "On the managed App Platform
  Linode owns the apl-core version and picks the moment it rolls (ADR 0005) — LLZ
  never runs `helm upgrade` on apl-core. That means LLZ has no upgrade hook to
  hang this on, and an operator cannot reliably win a race against a managed
  rollout by hand." So the 6.1.0 prerequisite annotation is applied on **every**
  bootstrap (`prepareAplUpgradeBestEffort`, `:116-126`), warn-not-fail.
- Evidence: the version floor is enforced (`minSupportedAplChartVersion = "6.0.0"`,
  `ci_assert_apl_version.go:50`) and major drift blocks in both directions
  (`clusterspec/aplversion.go:128-151`), with a time-boxed override env var
  (`AllowMajorDriftEnv`, `:36`). Minor drift only warns (`:153-169`) — argued at
  `:121-127`. These are good, and they are **preflight checks on the spec**, not
  on the cluster.
- Evidence: the 6.0→6.1 note is explicit that its verification was a **source
  diff, not a run** — `docs/designs/apl-core-v61-upgrade.md:3-4` ("**Status:**
  Partial … Validate in lab before any non-lab promotion"), `:27-28` ("Each row
  was checked against the `v6.0.0…v6.1.0` source diff, not inferred from the
  release notes"), and a manual lab checklist at `:115-127` naming the Loki
  community-chart switch as "the highest-risk item".
- Evidence: the 5.x→6.x design's downstream section is already annotated
  historical (`docs/designs/apl-core-v6-migration.md:255-265`).
- Why it matters: the honest position is "we cannot test this — Linode picks the
  moment — so we front-load spec checks and hand the operator a lab checklist."
  That is defensible. What is not defensible is `apl-core-migration-runbook.md`
  still reading as though LLZ drives the upgrade (F11).
- Recommendation: fold F11's doc correction and add one sentence to
  `docs/apl-core-migration-runbook.md` stating the managed reality up front:
  Linode owns the version and the timing; the operator's levers are
  `llz ci prepare-apl-upgrade`, the lab checklist, and the `assert-apl-version`
  floor.

---

## Priority

| # | Finding | Severity | Cost |
|---|---|---|---|
| F6 | Upgrade deploys to the live cluster with no plan/approval, while the TF half is gated | High | Doc sentence + optional diff flag |
| F15 | No repo↔cluster drift check anywhere; no scheduled Argo-Synced assertion | High | One job, primitives exist |
| F12 | `runUpgrade` itself is never exercised end-to-end | High | Small diff to an existing gate |
| F11 | apl-core runbook contradicts shipped behaviour on the one lever operators reach for | High | Doc correction |
| F17 | No destructive-plan gate; the documented approval control is never verified | High | `tofu show -json` parse + a doctor check |
| F13 | release-e2e is greenfield-only | Medium | New dispatch-only lane |
| F16 | Drift job measures branch head, monthly, alerts nobody | Medium | Small |
| F4 | No rollback path, and none documented | Medium | Doc |
| F5 | `--dry-run` cannot preview the diff | Medium | Reuses existing render |
| F2 | Owned snapshot discarded without restore on copier failure | Medium | Small |
| F1 | Digest lock covers only token-free `managed` | Medium | New lint step |
| F7 | `commitUpgrade` does `git add -A` on a dirty tree | Medium | Small |
| F14 | Upgrade gates skip silently | Low | Small |
| F3 | Managed overwrite not transactional | Low | Small |
| F8 | CI image vars go stale on every upgrade | Low | Opt-in flag |
| F18 | Chart bumps out of scope by design; coherence guarded, safety not | Low | Process |
| F19 | apl-core upgrade untestable by construction | Low | Doc |
| F9 | Pin skew — fixed, narrow by design | — | None |
| F10 | TF destructive scars — documented | — | See F17 |

**19 findings** (F9/F10 recorded as context; 4 items carry a
**[documented tradeoff]** label where the repo already accepted them).
