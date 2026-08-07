# CI/CD Robustness Review — LKE Landing Zone

**Scope:** all 13 workflows in `.github/workflows/`, all 16 in `instance-template/.github/workflows/`, both composite-action trees (`.github/actions/`, `instance-template/.github/actions/`), `.github/dependabot.yml`, `.github/scripts/dispatch-and-watch.sh`, and the `Makefile` lint groups. Read-only; no repo file was modified.

---

## Executive summary

This is a well-above-average Actions estate. Every `uses:` in both trees is SHA-pinned with a `# vN` comment — 100% compliance with the directory's own doctrine, verified exhaustively, not sampled. There is no `pull_request_target`, no `issue_comment`, and no `workflow_run` trigger anywhere; every one of the seven places untrusted event data is consumed passes it through `env:` rather than interpolating into a `run:` body, with `release-e2e-lane.yml:203-207` carrying an inline explanation of why. Container images get multi-arch builds, SLSA provenance, SBOMs, and digest-bound keyless cosign signatures — including the deprecated alias, for the correct reason. The scars-as-defaults convention is genuinely load-bearing: `dispatch-and-watch.sh` re-reads a run's conclusion because `gh run watch --exit-status` was observed lying in both directions.

The weaknesses are not in the mechanics; they are in **enforcement boundaries that live outside the YAML**. Three findings share one shape — a control that is documented, tested, and believed to be in force, but that nothing actually executes: the pre-release→promote gate is pure human convention (`llz-release.yml` never consults the e2e conclusion), `version-pins-check` runs under neither the default local gate nor CI, and `instance-test.sh`'s actionlint pass silently skips when the binary is absent. Each is the vacuous pass `docs/e2e-gates.md:129-137` refuses, one level up.

Two findings carry real money or credential risk: a failed teardown is *detected and then ignored* (no scheduled reaper exists anywhere in the repo, and the e2e-triage skill states the leaked backlog is what wedges the *next* cluster-create), and the PR-time Terraform plan job reads production Linode and state credentials at repo scope, bypassing every `infra-<region>` Environment protection rule the apply path relies on.

On pipeline efficiency there is no finding to report, and that is the honest answer rather than a gap: the ~40-minute Provision job is structurally sequential (apply-cluster → bootstrap-cluster → bootstrap-openbao → converge, `release-e2e-lane.yml:22-33`), nothing in it is parallelisable without standing up a second cluster the Linode quota forbids (`release-e2e.yml:6-12`), and the cacheable half already has a documented fast path in `release-e2e-warm.yml` (~8-12m by reusing a live cluster). Caching elsewhere is in good shape — `--cache-from/--cache-to type=gha` per image scope (`build-images.yml:187-188`), `actions/setup-go` with `cache-dependency-path` (`release-e2e-lane.yml:162-165`), and a single shared `render-charts` prerequisite so charts render once per `lint-k8s` run (`Makefile:592-593`).

**Counts:** Critical 0 · High 5 · Medium 10 · Low 4.

---

# Findings

## HIGH

### H1. The pre-release → promote gate is unenforceable; nothing consults the e2e result
**Impact: High | LoE to fix: Low**

**Evidence**
- `.github/workflows/llz-release.yml:19-22` — `on: release: types: [released]`, plus `workflow_dispatch:`. No `if:` and no first-step check reads the e2e conclusion.
- `.github/workflows/release-e2e.yml:22-24` — e2e fires on `release: types: [prereleased]`, a *separate* event with no linkage to the later one.
- `.github/workflows/release-e2e-lane.yml:335-359` — the `gate` job computes the verdict and then exits; nothing downstream reads it.
- `AGENTS.md:100-107` and `.github/workflows/AGENTS.md:143-152` both describe the promote click as "the approval".

**Why it matters**
A human can uncheck "pre-release" on a candidate whose e2e run is red, cancelled, or never started, and `llz-release.yml` will build binaries, attach `SHA256SUMS`, and retag `ghcr.io/<owner>/llz:vX.Y.Z` — the artifacts adopters `curl` and pin. The whole two-step design exists to make e2e blocking, and the blocking is a social convention. `AGENTS.md:108` also states tags are immutable, so a bad promotion cannot be withdrawn by moving the tag; it requires a new version. The repo's own doctrine (`docs/e2e-gates.md:211-213`) says "a gate that isn't in the lane table does not exist" — this is that failure at the release layer.

**Recommendation**
Add a first job to `llz-release.yml` that hard-fails unless `release-e2e.yml`'s most recent run for `github.sha` concluded `success`:
```yaml
verify-e2e:
  runs-on: ubuntu-latest
  permissions: { contents: read, actions: read }
  steps:
    - env: { GH_TOKEN: "${{ secrets.GITHUB_TOKEN }}" }
      run: |
        set -euo pipefail
        c=$(gh run list --workflow=release-e2e.yml --repo "$GITHUB_REPOSITORY" \
              --commit "$GITHUB_SHA" --limit 1 --json conclusion --jq '.[0].conclusion // "none"')
        [ "$c" = success ] || { echo "::error::release-e2e for $GITHUB_SHA concluded '$c'"; exit 1; }
```
Put the decision logic in `llz ci` (as `chart-version-guard` and `assert-adopter-pin` already are) so it is unit-tested rather than inline bash, and make `build`/`release`/`image-tag` all `needs: verify-e2e`. Provide a documented `--force` override that requires an explicit dispatch input, so the bypass is auditable rather than invisible.

---

### H2. Teardown failure is detected but never remediated; no scheduled sweeper exists
**Impact: High | LoE to fix: Medium**

**Evidence**
- `.github/workflows/release-e2e-lane.yml:356-358` — the gate prints `::error::teardown $T — a cluster may have leaked; check ${INSTANCE_REPO} state` and exits non-zero. That is the entire remediation.
- `.github/scripts/dispatch-and-watch.sh:80-97` — on timeout the script *cancels* the instance-side run and exits 124. A cancelled destroy leaves partially-deleted infra and no sweep.
- `instance-template/.github/workflows/llz-terraform.yml:840,858,879` — `reap-volumes`, `reap-nodebalancers`, `reap-objkeys` and the VPC delete (`:861`) all live **inside** the destroy job. If the destroy dispatch never completes, none of them run.
- No workflow in either tree runs a scheduled sweep. `.claude/skills/e2e-triage/SKILL.md:159-171` confirms the manual-only path (`make reap-orphans`, dry-run by default, needs `CONFIRM=yes`).
- `.claude/skills/e2e-triage/SKILL.md:161-163`: "Failed and cancelled cycles leak Linode resources, and the backlog is what makes the **next** cluster-create hang." `SKILL.md:103-106` names VPC-quota exhaustion as a *confirmed* root cause of the silent cluster-create hang.

**Note on what is already handled.** The teardown job's gating is correct and should not be changed: `release-e2e-lane.yml:287` is `if: always() && needs.provision.result != 'skipped' && (!inputs.dry_run)`, and `always()` does run on an ordinary run cancellation, so a normal cancel still dispatches the destroy. The gap is not the trigger — it is what happens when the destroy dispatch itself does not finish, and the fact that a force-cancel (a second cancel click) kills `always()` jobs too.

**Why it matters**
The unswept path is concrete: teardown's destroy dispatch hits the `dispatch-and-watch.sh` timeout, the script cancels the instance-side run *mid-destroy* and exits 124, and every sweep at `llz-terraform.yml:840,858,861,879` is downstream of a destroy that never completed. The gate then prints a message and the run ends. Nothing re-attempts, nothing alerts, nothing sweeps later. This is a documented, already-realised failure loop with a compounding cost: leak → next run hangs → operator cancels → leaks more. Because `release-e2e.yml:50-52` uses `concurrency: release-e2e` with `cancel-in-progress: false`, a wedged run also blocks every subsequent e2e for up to its 200 + 120 minute budget (`release-e2e-lane.yml:251,289`). The cleanup capability is fully built and unit-tested — it simply has no trigger other than a human.

**Recommendation**
Add a scheduled sweeper workflow in this repo (daily) that runs the existing reaper in dry-run, scoped to the e2e cluster label, and *fails the job* when it finds anything. That turns every path in this class — timed-out destroy, force-cancelled run, `keep_cluster` left behind (M9), a warm cluster nobody destroyed — into a red check with a name, instead of a bill nobody is looking at. Deliberately report-only, not auto-delete: `.claude/skills/e2e-triage/SKILL.md:174-178` is emphatic that sweeping on a hunch is its own hazard, and a failing check is enough to get a human to run the confirm step.

---

### H3. Dependabot auto-merge of the `github-actions` ecosystem rewrites SHA pins without human review
**Impact: High | LoE to fix: Low**

**Evidence**
- `.github/dependabot.yml:4-13` — `package-ecosystem: "github-actions"`, `directory: "/"`, grouped `patterns: ["*"]` into one PR.
- `.github/workflows/dependabot-automerge.yml:37-51` — greps the PR body; anything not tagged `semver-major` gets `gh pr merge --auto --squash`.
- `.github/workflows/AGENTS.md:163` — "SHA-pin all `uses:` references… use the full commit SHA with a `# vN` comment." Dependabot's github-actions updater rewrites exactly those SHAs.
- The publishing surface reachable from `main`: `build-images.yml:106-109` (`packages: write` + `id-token: write`, cosign signing), `publish-charts.yml:45-49` (same), `llz-release.yml:74-75` (`contents: write`).
- `.github/CODEOWNERS` exists, but CODEOWNERS review is only enforced if branch protection requires it — repo config not visible in the tree, therefore unverifiable at review time.

**Why it matters**
A grouped minor/patch action bump lands on `main` with zero human eyes on the new SHAs. Dependabot resolves the SHA from the upstream tag at PR time; if an upstream action's tag has been moved or its repo compromised, the new pin is the compromised commit and it merges automatically. The next `push: main` fires `build-images.yml`, which runs that code with `packages: write` and a Sigstore identity — the compromised step can exfiltrate `GITHUB_TOKEN` or publish a signed image. SHA-pinning is the repo's stated supply-chain control; automerging the pin rewrites is the one change class that dissolves it.

The same automerge path covers `gomod` at `/tools` (`dependabot.yml:16-25`), which ships the `llz` binary that performs destructive cloud operations (`reap`, `credentials … revoke-old`) in adopter accounts. Neither ecosystem's PRs are e2e-validated — e2e only runs on a pre-release tag, so bumps sit on `main` unexercised until someone cuts a candidate.

**Recommendation**
Exclude the `github-actions` ecosystem from automerge — the fix is one condition in `dependabot-automerge.yml`, keyed on the PR's `github-actions` label (`dependabot.yml:10`) or on the changed paths. Keep automerge for `gomod` patch updates only (drop minor), and require `go-vuln-audit` to have run on the head SHA (see M5). If automerge for actions is retained deliberately, it must at minimum be gated on a required review from `.github/CODEOWNERS` enforced by branch protection — and that dependency should be written down, because today nothing in the repo records it.

---

### H4. The PAT-create step writes an unscrubbed credential record to the job summary — while the adjacent branch scrubs
**Impact: High | LoE to fix: Low**

**Evidence**
- `instance-template/.github/actions/linode-credentials/action.yml:173` — `NEW_TOKEN=$(jq -r '.new_token // empty' < "$OUT_FILE")`. The raw Linode PAT is in `$OUT_FILE`.
- `…/action.yml:197-203` — the PAT-create branch writes `jq '.' < "$OUT_FILE"` straight into `$GITHUB_STEP_SUMMARY`, unmodified, including `new_token`.
- `…/action.yml:279-289` — the obj-key-create branch *does* scrub, replacing `new_access_key` / `new_secret_key` with `"(masked)"` before the summary, with the comment: *"`::add-mask::` would redact them, but a clean record reads better."*
- `…/action.yml:206-237, 291-323` — the two `revoke-old` branches also dump `jq '.'` unmodified (lower value: IDs only).
- Scope of the credential: `…/action.yml:21-23` — needs `account:read_write`, i.e. can create and revoke PATs across the whole Linode account.

**Why it matters**
The finding rests on the **asymmetry**, not on any claim about GitHub's masking behaviour. Two sibling branches in one file handle equivalent-value secrets with different care: the obj-key path builds a scrubbed record before writing the summary, and its comment shows the author actively considered whether `::add-mask::` was sufficient and chose not to depend on it. The PAT path — carrying a credential with `account:read_write` over the entire Linode account — depends on exactly that. Whichever way the masking question resolves, one of these two branches is wrong, and the file gives no reason why the higher-value secret got the weaker treatment. This is the internally-consistent-and-wrong shape `docs/e2e-gates.md:26-33` describes: both halves look complete from their own angle.

> Scope note: whether `::add-mask::` covers `$GITHUB_STEP_SUMMARY` was **not verified** in this review — no primary source was consulted. The recommendation does not depend on the answer. If someone establishes that summaries are masked, the correct response is still to make the two branches agree (and then simplify the obj-key scrub), not to remove it from the PAT branch.

**Recommendation**
Apply the obj-key branch's scrub to the PAT branch: build a `SCRUBBED` record with `new_token` replaced before writing the summary. Better, move the redaction into `llz credentials` itself so it emits a summary-safe record and a separate secret-bearing stream — a single tested seam instead of a scrub duplicated per call site (the split-contract archetype in `docs/e2e-gates.md:108-122`). Add a unit test asserting the summary-bound record never contains a token field.

---

### H5. The PR-time Terraform plan reads production credentials at repo scope, bypassing `infra-<region>` Environment protection
**Impact: High | LoE to fix: Medium**

**Evidence**
- `instance-template/.github/workflows/llz-terraform.yml:142-151` — `plan-cluster-pr` declares **no** `environment:`.
- `…:168-173, 183-196` — it nonetheless consumes `secrets.TF_STATE_ACCESS_KEY`, `secrets.TF_STATE_SECRET_KEY`, `secrets.TF_STATE_ENCRYPTION_PASSPHRASE`, and `secrets.LINODE_API_TOKEN`. Without an `environment:`, these resolve at **repo** scope only.
- Contrast: every apply/destroy job declares `environment: infra-${{ inputs.region }}` — `…:277, 431, 605, 675, 732, 913, 990, 1040, 1212, 1270, 1326`.
- `…:152-154` — the PR plan matrixes over **every** discovered deployment, so it plans against production alongside lab.
- `…:180-187` — the "plan" job runs `llz ci tf-import`, which **writes** to Terraform state.
- `docs/workflows/llz-breakglass-openbao.md:49-50` and `instance-template/.github/workflows/promote.yml:25,34,101` document required reviewers on `infra-*` Environments as the approval control.

**Why it matters**
For `plan-cluster-pr` to function at all, repo-level copies of those four secrets must exist. Every protection rule on `infra-<region>` — required reviewers, wait timers, deployment-branch restrictions — is therefore bypassable: open a PR touching `terraform-iac-bootstrap/**` and a job runs with the production Linode token and state credentials, under no approval gate. The fork guard at `…:145` is correct and well-commented, but it only excludes forks; anyone with push access to the instance repo qualifies. And because `tf-import` mutates state, this is not a read-only preview — a PR can write to production Terraform state before any human approves anything.

**Recommendation**
Give `plan-cluster-pr` an `environment: infra-${{ matrix.region }}` so it draws environment-scoped secrets and inherits the protection rules, then delete the repo-level duplicates. If PR-time plans against production are wanted without an approval prompt, use a dedicated read-only Environment (`plan-<region>`) holding a Linode token scoped to `read_only` and state credentials scoped to read — that gets the preview without granting write. Separately, move `tf-import` out of the PR path; a state mutation does not belong in a job named "plan". Add an `llz ci` guard asserting no job that consumes `LINODE_API_TOKEN` lacks an `environment:` declaration — this is statically decidable and belongs in `make lint` per `docs/e2e-gates.md:165-169`.

---

## MEDIUM

### M1. `version-pins-check` runs in neither the default local gate nor CI
**Impact: Medium | LoE to fix: Low**

**Evidence**
- `Makefile:640-642` — the target exists and is `LLZ_FORCE_SOURCE`-correct.
- `Makefile:631-635` — its rationale: "The Go constants sat on Terraform 1.9.8 after the other two moved to OpenTofu 1.12.5 — caught by hand then, caught here now."
- `Makefile:670` — invoked **only** under `LINT_ALL=1`.
- `Makefile:674-711` — the change-aware default `lint` path never invokes it, including the `.github/workflows/*.yml` branch at `:703-705`.
- `.github/workflows/lint.yml` — no job runs it. Verified with `rg 'version-pins' .github/workflows/ template-scripts/ Makefile` from the repo root: the only hits are `Makefile:8` (the `.PHONY` list), `Makefile:659` (a comment), and `Makefile:670`.

**Why it matters**
The guard was written *because* the drift it detects had already shipped and been caught by hand. It now runs only when someone types `make LINT_ALL=1 lint`, which nothing requires. The drift class it covers — `TF_IMAGE`/`KUBE_IMAGE` constants versus the Dockerfile ARG block — is exactly what breaks a release-pinned adopter, and `release-e2e-lane.yml:182-199` documents that adopter-shape image-pin drift has shipped before with every e2e run green.

**Recommendation**
Add `version-pins-check` to the `Makefile:703-705` branch (it should fire on any workflow, Dockerfile, or `tools/` change) and to `lint.yml`'s `go-tests` job beside `untestable-loc-check` at `lint.yml:559`.

### M2. `instance-test.sh`'s actionlint pass skips silently when actionlint is absent
**Impact: Medium | LoE to fix: Low**

**Evidence**
- `template-scripts/ci/instance-test.sh:220-227` — `if command -v actionlint …; then … else step "actionlint (SKIPPED — not installed)"; fi`. No non-zero exit.
- `Makefile:575` — `actions-lint` covers only `.github/workflows/*.yml`; the 4,760 lines under `instance-template/.github/workflows/` are actionlinted **only** through this path.
- CI installs it (`lint.yml:443-447`), so CI is covered; the local gate is the hole.
- `docs/e2e-gates.md:131-137` — "A gate that reports success having examined nothing is worse than no gate."

**Why it matters**
A contributor editing `llz-terraform.yml` or `llz-secret-rotation.yml` on a machine without actionlint gets a green `make lint` that examined none of it. Since `make lint` is declared "the authoritative final gate" (`AGENTS.md:156`), that green is load-bearing and wrong. This is the repo's own fail-closed doctrine violated inside the repo's own tooling.

**Recommendation**
Fail the step when actionlint is missing, or auto-install it the way `Makefile:740-742` self-bootstraps gitleaks and `Makefile:125-129` already handles actionlint for the `tools` target. Prefer auto-install — a warning the contributor must read is the thing the owning-layer-fixes doctrine says to convert into a task.

### M3. `llz` release binaries carry checksums but no signature or attestation
**Impact: Medium | LoE to fix: Low**

**Evidence**
- `llz-release.yml:83-95` — `sha256sum llz-* > SHA256SUMS`, uploaded to the same release as the binaries. No cosign, no `actions/attest-build-provenance`.
- `llz-release.yml:24-25, 74-75` — no `id-token: write` on the `build` or `release` job (contrast `build-images.yml:109`, `publish-charts.yml:49`).
- `build-images.yml:181-182, 199-210` — images get provenance, SBOM, and digest-bound keyless signatures.
- `publish-charts.yml:66-67, 99-105` — charts get keyless signatures.

**Why it matters**
Both other artifact classes are signed; the binaries — the ones adopters `curl` and execute with cloud credentials — are not. An unsigned `SHA256SUMS` sitting beside the assets it describes protects against transport corruption, not against anyone who can write to the release. `llz self-update` (`llz-release.yml:152-162`) consumes this channel.

**Recommendation**
Add `id-token: write` to the `release` job and either `cosign sign-blob` each asset (matching the estate's existing keyless pattern) or `actions/attest-build-provenance`, then teach `llz self-update`'s verify step to check it. The verification half matters more than the signing half: a signature nothing verifies is decoration.

### M4. `publish-charts.yml` has no `concurrency:` block — the immutability check is TOCTOU
**Impact: Medium | LoE to fix: Low**

**Evidence**
- `publish-charts.yml:17-23` — `on: push: branches: [main, master]` plus dispatch. No `concurrency:` anywhere in the file.
- `publish-charts.yml:99-105` — the check-then-push is inside `llz ci publish-charts`.
- `AGENTS.md:124-126` — "never overwrite an existing tag (Argo Applications pin `targetRevision: X.Y.Z`)."
- `build-images.yml:36-49` — the sibling publisher adds `concurrency` with a long comment about exactly this race against a mutable tag.

**Why it matters**
Two main pushes touching `kubernetes-charts/**` in quick succession — or a push racing a `workflow_dispatch` — both read "version X not published", then both push it. The immutability invariant is enforced by a read that another run can invalidate between check and write. `chart-version-guard.yml:1-19` documents that a chart version silently not reaching the registry already caused a real outage (the runner-acl RBAC grant); the inverse — an existing version silently overwritten — mutates every consumer pinned to it.

**Recommendation**
Add `concurrency: { group: publish-charts, cancel-in-progress: false }`, mirroring `build-images.yml:47-49`. `cancel-in-progress: false` matters for the same reason stated there — a cancelled publish leaves a partial set.

### M5. `go-vuln-audit.yml` is weekly-only, never runs on a PR, and has no failure routing
**Impact: Medium | LoE to fix: Low**

**Evidence**
- `go-vuln-audit.yml:10-13` — `schedule: '0 2 * * 0'` and `workflow_dispatch` only. No `pull_request`.
- `go-vuln-audit.yml:19-22` — the `audit` job has no notification, no issue creation, no `needs`-based fan-out.
- `.github/dependabot.yml:16-25` — `gomod` bumps are opened weekly and auto-merged (H3).

**Why it matters**
Two gaps compound. A dependency bump that *introduces* a vulnerable version can automerge and go unexamined for up to seven days. And when the scheduled run does go red, GitHub's only default notification is an email to whoever last edited the cron — a routing nobody owns. `docs/e2e-gates.md:236-254` is explicit that a gate's job is not finished until its failure says what to do next; here the failure may not reach anyone at all.

**Recommendation**
Add `pull_request` (paths `tools/**`) to the trigger so bumps are audited before merge, and make `go-vuln-audit` a required check for the automerge path. Separately, add a failure step that opens or updates a tracking issue (`gh issue create` with `issues: write`), so a red scheduled run leaves a record in the issue tracker rather than only an email.

### M6. `persist-credentials: false` is set on exactly one of 30+ checkouts
**Impact: Medium | LoE to fix: Low**

**Evidence**
- `go-vuln-audit.yml:25` — the only occurrence across both trees. Verified with `rg 'persist-credentials' .github/ instance-template/.github/` from the repo root: one hit.
- Every other `actions/checkout` leaves `GITHUB_TOKEN` in `.git/config`. Notable cases: `lint.yml:135,175,377,410,434,485,514` (jobs that then run `checkov`, `kube-linter`, `helm`, `copier`, and third-party Go tooling), `secret-scan.yml:39-43` (full-history checkout), `release-e2e-lane.yml:160,253,291`.

**Why it matters**
Any step after checkout — including third-party linters and `go install`-ed binaries — can read the token from the working tree. `lint.yml`'s jobs install and execute a lot of external tooling (`lint.yml:447` `go install …actionlint@v1.7.7`, checkov, kube-linter). The single existing usage shows the pattern is understood; it just isn't applied where the exposure is largest.

**Recommendation**
Set `persist-credentials: false` by default on every checkout that does not subsequently run a git operation needing auth. `lint.yml`, `secret-scan.yml`, `chart-version-guard.yml`, and `go-tests` are all pure-read and should carry it.

### M7. Fork PRs skip the Kubernetes and Terraform lint gates entirely
**Impact: Medium | LoE to fix: Medium**

**Evidence**
- `lint.yml:123` — `if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository` on the `kubernetes` job.
- `lint.yml:473` — the same condition on the `terraform` job.
- Reason given at `lint.yml:121-122`: fork PRs cannot pull the private `KUBE_IMAGE`.
- `secret-scan.yml:8-11` explicitly notes it uses pure tooling *so that* it runs on fork PRs — the estate knows this distinction matters.

**Why it matters**
`make lint-k8s` is where `wave-health-guard`, `mesh-egress-guard`, `monitoring-label-guard`, `credential-coverage-guard`, `dropped-apiversions-check`, and the ExternalSecret-path checks live (`Makefile:596-600`) — the guards whose original wedges are catalogued in `.claude/skills/e2e-triage/SKILL.md:70-79`. A fork PR touching charts gets none of them. A skipped job also reports as skipped, which several branch-protection configurations treat as satisfying a required check.

**Recommendation**
Split the checks that need no private image out of the container job so they run for forks: `helm-lint-charts`, `helm-dep-lock-check`, `placeholder-guard`, and the pure-Go guards need only helm and the `llz` binary, both obtainable on a public runner. Keep only the genuinely image-dependent steps behind the head-repo condition. If forks are not an accepted contribution path at all, say so in `CONTRIBUTING.md` and the finding closes on documentation instead.

### M8. `tofu fmt -check` runs locally but never in CI, and the local gate is opt-in
**Impact: Medium | LoE to fix: Low**

**Evidence**
- `Makefile:594-595` — "tf-fmt-check is kept OUT of LINT_TF (it uses tofu, absent from the CI TF_IMAGE)".
- `Makefile:689` — the change-aware local path does run it.
- `lint.yml:491` — CI runs `make lint-tf`, which excludes it (`Makefile:601,605`).
- `CONTRIBUTING.md:27` — the hooks require `git config core.hooksPath template-scripts/hooks`, i.e. opt-in per clone.
- `AGENTS.md:158` lists `tofu fmt` among what `make lint` covers.

**Why it matters**
A contributor who never ran the `core.hooksPath` command has no formatting gate at any layer. The doctrine sentence at `AGENTS.md:156` — "`make lint` — the authoritative final gate" — is true locally and not true in CI for this check.

**Recommendation**
Either add `tofu` to the CI image (it is a Terraform image; the absence looks accidental), or run `tf-fmt-check` in a small non-container CI job with `opentofu/setup-opentofu`. The latter is the smaller change and closes it today.

### M9. `keep_cluster` and the warm lane leave billing clusters with no TTL, alarm, or expiry
**Impact: Medium | LoE to fix: Medium**

**Evidence**
- `release-e2e-lane.yml:297-309` — `keep_cluster=true` emits a `::warning::` and instructions; nothing tracks or expires the cluster.
- `release-e2e-warm.yml:119-136` — the warm lane never tears down by design; `:136` emits a `::notice::`.
- `release-e2e-warm.yml:34-38` — "If the e2e state is empty… the apply CREATES a cluster from scratch… and it then stays up billing. There is no cheap 'is it warm?' precondition, so this is documented, not enforced."
- `.claude/skills/e2e-triage/SKILL.md:116-119` — "a kept cluster bills until someone removes it."

**Why it matters**
Three separate paths (keep_cluster, warm lane, warm-lane-on-empty-state) all end in a live cluster whose only cleanup trigger is a human remembering. The warm lane's own header names the un-enforced precondition. These are deliberate trade-offs, but they compound with H2: nothing anywhere notices a cluster that has been up for a week.

**Recommendation**
The sweeper proposed in H2 covers this too if it reports (not deletes) any e2e-labelled cluster older than N hours as a failing check. That converts three documented warnings into one enforced control, and it is the "write the task that makes the warning unnecessary" move.

### M10. Eight-plus jobs omit the `permissions:` block their own directory doctrine requires
**Impact: Medium | LoE to fix: Low**

**Evidence**
- `.github/workflows/AGENTS.md:113` — "Every job must have an explicit `permissions:` block… Never omit the block."
- Missing at job level: `chart-version-guard.yml:43` (`version-bumped`); `go-vuln-audit.yml:19` (`audit`); `release-e2e-lane.yml:247` (`provision`), `:282` (`teardown`), `:335` (`gate`). Per-file counts of job-like keys versus `permissions:` blocks also show shortfalls in `llz-terraform.yml` (20 / 17), `llz-scheduled-checks.yml` (8 / 6), `llz-secret-rotation.yml` (12 / 9), `llz-cluster-health.yml` (3 / 1), `llz-wedge-gameday.yml` (3 / 1), and workflow-level-only blocks in `promote.yml`, `terraform.yml`, `scheduled-checks.yml`, `secret-rotation.yml`, `wedge-gameday.yml`.

**Why it matters**
Each of these currently inherits a workflow-level `contents: read`, so today the effective grant is correct — this is a doctrine and drift finding, not a live exposure. But the doctrine exists because the inheritance is invisible: someone raising a workflow-level permission later silently raises every block-less job with it. `dependabot-automerge.yml:6-8` documents the repo already being bitten by the inverse confusion.

**Recommendation**
Add the explicit blocks. This is statically decidable and belongs in `make lint` as an `llz ci` guard per `docs/e2e-gates.md:165-169` — a guard would also cover the instance tree, which `actionlint` alone does not check for this.

---

## LOW

### L1. `actions/checkout` SHA drift between the two trees
**Impact: Low | LoE to fix: Low** — `.github/workflows/*` pin `3d3c42e5…` (v7.0.1); every `instance-template/.github/workflows/*` pins `9c091bb2…` (v7.0.0). Not a risk, but the delivered surface lags, and `version-pins-check` (M1) is the guard class that would notice — if it ran. Fold action-pin agreement between the trees into that check.

### L2. Missing `timeout-minutes` on long-lived and publishing jobs
**Impact: Low | LoE to fix: Low** — zero `timeout-minutes` in `lint.yml` (567 lines, includes a kind cluster job), `build-images.yml` (multi-arch QEMU builds), `publish-charts.yml`, `secret-scan.yml` (full-history scan), `chart-version-guard.yml`, `dependabot-automerge.yml`. The default is 360 minutes. `cluster-access/action.yml:18-25` documents a 15m34s silent hang that "only showed up by diffing step timestamps" — the same class applies to these jobs, and `build-images.yml`'s `concurrency` group means a hung build blocks the next merge.

### L3. Automerge parses Dependabot PR prose rather than using `dependabot/fetch-metadata`
**Impact: Low | LoE to fix: Low** — `dependabot-automerge.yml:39-48` greps the body for `update-type: version-update:semver-*`. Both failure directions are fail-safe (unrecognised body → skip; any major in a group → skip), so this is a maintenance rather than a security finding. The official `dependabot/fetch-metadata` action exposes `update-type` as a typed output and would remove the dependency on an unversioned prose format.

### L4. No `docker` ecosystem in `.github/dependabot.yml`
**Impact: Low | LoE to fix: Low** — `.github/dependabot.yml` covers `github-actions` and `gomod` only. Base images in `dockerfiles/Dockerfile` — which ship to adopters as `ci-tofu`, `ci-kubernetes`, and `devcontainer` — are never auto-bumped, and `go-vuln-audit` scans only the Go module. `build-images.yml:17-18` weekly-rebuilds them, which picks up floating-tag base updates but not pinned ones. Worth a deliberate decision recorded either way; if base pins are managed by hand, say where.

---

# Done well

These are not filler — several are things most repos of this size get wrong, and a few are better than the industry norm.

1. **100% SHA pinning, verified exhaustively.** Every `uses:` in `.github/workflows/`, `instance-template/.github/workflows/`, `.github/actions/setup-llz/action.yml:35`, and the six instance composite actions is a full 40-hex commit SHA with a `# vN` comment. Not one tag reference. The only non-SHA refs are repo-local `./` paths, which is correct. (Checked with `grep -rn 'uses:' .github/workflows/ instance-template/.github/workflows/ .github/actions/ instance-template/.github/actions/` and inspecting every line.)

2. **Zero script-injection surface.** No `pull_request_target`, no `issue_comment`, no `workflow_run`. All seven consumptions of `github.event.*` go through `env:` — `dependabot-automerge.yml:25`, `chart-version-guard.yml:59`, `publish-charts.yml:103`, `llz-release.yml:127`, `release-e2e-lane.yml:207`, `llz-secret-rotation.yml:143` — and `release-e2e-lane.yml:203-206` states the reasoning: *"a tag name is attacker-influenceable text in the general case, and `${{ }}` inside a shell body is substituted before the shell ever sees it… As an env var it is data."* That comment is the correct model, written down where the next editor will find it.

3. **Container supply chain is close to complete.** `build-images.yml:178-191` builds `linux/amd64,linux/arm64` with `--provenance=true --sbom=true --metadata-file`, then `:199-210` signs **by digest**, not by tag, so the signature binds the exact manifest. And it signs the deprecated `ci-terraform` alias too, with the reasoning at `:193-198`: the population still pinning the old name is "the exact population least likely to notice" an unsigned image. That is a genuinely uncommon degree of care.

4. **`dispatch-and-watch.sh` is hardened by scar, not by theory.** `:68-74` re-attaches because `gh run watch --exit-status` was observed exiting non-zero on a healthy run (a jobs-endpoint 404). `:107-118` re-reads the conclusion because watch was observed returning **0** for an already-failed run. Both directions of a lying exit code, both closed, both explained. `:75-97` bounds the whole wait and cancels rather than pinning the caller.

5. **Concurrency reasoning is explicit and, unusually, sometimes argues for *absence*.** `build-images.yml:36-49` explains why `cancel-in-progress` must stay false (a cancelled build leaves `sha-<>` unpublished for something actively waiting on it). `release-e2e.yml:47-52` plus `max-parallel: 1` at `:60` gives quota-safe single-cluster sequencing. And `bootstrap-openbao.yml:52-56` documents why repeating a group on the top-level caller **deadlocks** — a failure mode most teams discover in production.

6. **Reusable-workflow permission unioning is understood.** `release-e2e.yml:66-75` carries the note that a called workflow can never hold more than the calling job's token, so the caller's block must be the union — *"Left at the workflow default of contents:read, the lane's request is an escalation and GitHub fails the run at startup (startup_failure, before any job logs exist)."* That is a genuinely obscure failure mode, correctly handled and documented.

7. **Fork PRs are correctly denied production secrets.** `llz-terraform.yml:140-141` — *"SECURITY: this job consumes production secrets, so the head-repo check below restricts it to internal PRs"* — and the same guard on `:135, 207, 224`. (H5 is about *internal* PRs, not this control, which is right.)

8. **The instance is genuinely self-contained.** ADR 0003's no-cross-repo-`uses:` rule holds: every instance-side reference is `./`-local, explained at `cluster-access/action.yml:10-16` including why it matters (air-gapped GitHub Enterprise). `terraform.yml:7-11` records the bug this fixed — cross-org `secrets: inherit` arriving silently empty for adopters in a different org.

9. **Environment-scoped secrets on the mutating paths.** Every apply, destroy, bootstrap, breakglass, and rotation job declares `environment: infra-<region>`, and `llz-terraform.yml:53-67` explains why the `workflow_call` secrets are all `required: false` — environment-scoped secrets cannot satisfy a call-time contract, so presence is enforced at runtime by `llz ci require-secret`. That is the right answer to a real GitHub limitation, not a workaround. On the OIDC question: Linode publishes no GitHub OIDC federation, so long-lived PATs are structurally unavoidable here; the mitigations chosen (90-day validity cap at `linode-credentials/action.yml:49-52`, environment scoping, a daily age-based drain, and `llz credentials` writing the new value into each `infra-<deployment>` Environment secret itself) are the correct compensating set.

10. **Guard decision logic lives in unit-tested Go, not workflow bash.** `chart-version-guard.yml:21-23`, `publish-charts.yml:95-98`, and the `untestable-loc-check` budget (`Makefile:670`, `lint.yml:559`) all push judgement out of YAML. `release-e2e-lane.yml:208-213` even justifies a slightly-odd unconditional flag on the grounds that the `if/else` alternative would be "four more lines of untestable workflow bash." A repo that budgets its own untestable lines is doing something most do not.

11. **Secret scanning is correctly scoped.** `secret-scan.yml:1-11` is standalone with **no** `paths:` filter, deliberately — *"a leaked credential can land in any file"* — with `fetch-depth: 0` for full history and a pinned, SHA-verified gitleaks (`Makefile:740-742`, `--redact`). It also uses only public tooling *specifically* so it runs on fork PRs. That reasoning is exactly right and is the model M7 should follow.

12. **The `paths:` filter bug fix at `lint.yml:32-40` is worth citing on its own.** `**.md` rather than `**/*.md`, because GitHub's matcher treats `**` as including `/`, so `**/*.md` requires a separator and silently misses every root-level file — `AGENTS.md` and `README.md`, both of which `docs-guard` reads. The comment names it as "the vacuous pass docs/e2e-gates.md refuses, one level up." That is the doctrine being applied to itself, correctly, by someone who went looking.
