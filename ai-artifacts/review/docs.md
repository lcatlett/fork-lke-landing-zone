# LKE Landing Zone — Documentation Completeness & Accuracy Review (Round 3)

Read-only audit of the full `docs/` tree (~92 files), `README.md`, `CONTRIBUTING.md`,
`SECURITY.md`, `CODE_OF_CONDUCT.md`, all six `AGENTS.md` files, three directory READMEs
(`terraform-modules/`, `kubernetes-charts/`, `tools/`), and the `.claude/skills/` tree that
ships operator-facing guidance. Repo: `fork-lke-landing-zone` @ `main` (`41cd28a`), verified
identical to `gh-upstream/main` — no drift, no stale-branch risk. `docs-guard` baseline: green
(124 files, 862 `llz` invocations / 248 flags, 17 workflow dispatches, 603 links, 144 TOC
entries, zero errors). Every finding below is something that guard structurally cannot see.

This is the third full audit (after PR #406 and PR #411); their residue is encoded in
`.claude/skills/docs/SKILL.md` and was read first. Findings both prior audits already fixed
(duplicate ADR 0002, broken ADR 0011 link, missing ADR/design indexes, the `openbao set --yes`
omission, the `pvc-*` blast-radius framing) were re-verified and are **not** re-reported unless
noted as regressed. One cross-check: `ai-artifacts/review/reliability.md` (a concurrent,
separate engineering-reliability audit in this same working tree) already covers DR from an
infrastructure angle (no state backup, no OpenBao snapshot/restore). The DR findings below are
the **documentation** half of the same gap — what the docs claim, omit, or actively
mis-instruct — and are complementary, not duplicative.

## Executive summary

The delivered adopter path (`quickstart.md` → `adopter-guide.md`) is genuinely complete and
walks a system team from zero to a converged cluster. The damage is concentrated exactly where
`.claude/skills/docs/SKILL.md` predicted it would be: **the delivered 14** (`quickstart.md`,
`runbooks/**`, `playbooks/**`) and **the AI-facing onboarding skill**, which is read by an agent
coaching a human and is unguarded by `docs-guard` in a way that let a fabricated CLI command
survive to this audit. The single most consequential finding is that the documented "start
clean" rebuild path never discloses that the destroy step permanently deletes all
PersistentVolume data — inverting the `reclaimPolicy: Retain` signal an operator would
reasonably trust. Close behind: disaster recovery does not exist as a design at any status
(not even `Proposed`), no SLO documentation exists anywhere, and a finished, tested CIS
compliance-mapping branch has sat unmerged for six weeks while an ADR on `main` already assumes
it landed. A large, well-evidenced seam runs through the LandingZone spec: three spec blocks
were correctly flagged in one place as "inert" after a rendering pipeline was retired, but that
caveat did not propagate — the delivered scaffold, the quickstart's own example command, and an
incident runbook all still teach the dead mechanism as live. Structural root cause behind
several High findings: `docs-guard` validates flags against a *resolved* command but never
proves the command itself exists, so a renamed or invented verb is invisible to both the
numerator and denominator of its own coverage claim.

**Finding counts:** 2 Critical · 20 High · 33 Medium · 8 Low (63 total).

---

# Critical

## C-1. The documented cluster-rebuild path permanently deletes PersistentVolume data, and no delivered doc discloses it

**Impact: Critical** | **LoE to fix: Medium**

**Evidence:**
- `docs/runbooks/first-build-failed.md:104-124` (§6, "Starting clean") — the delivered
  procedure for an unrecoverable instance: `gh workflow run terraform.yml -f action=destroy
  -f module=all …`, then "sweep any leftovers (§5) and re-dispatch the build." The section's
  only stated cost is time and its only explicit warning concerns the Terraform state bucket
  (`:122-124`). Nothing mentions volume data.
- `tools/cmd/llz/ci_teardown.go:372-445` (`llz ci teardown-capture`) snapshots the cluster's
  Block Storage Volumes *before* the destroy, tracked by an authoritative `lke<id>` tag
  stamped at `CreateVolume` time — independent of any label. `docs/workflows/llz-terraform.md:
  553-575` — the destroy's own sweep then deletes every tracked volume after detach, and
  `--require-empty` **fails the destroy** if any survive. This is hardened to fail rather than
  leak, i.e. deletion is the designed outcome, not a side effect.
- `docs/workflows/llz-terraform.md:553-556` states the *opposite* signal in the same file:
  "`reclaimPolicy: Retain`, so CSI **never** deletes its Block Storage Volumes — only an API
  sweep does" — and `block-storage-retain` is the cluster's **sole default** StorageClass
  (`docs/lessons-learned.md:238-241`), so any unqualified `volumeClaimTemplate` lands there.
  `docs/workflows/llz-terraform.md` is not in the `deliver-docs` keep-set — this disclosure
  never ships.
- The asymmetry sharpens it: the same destroy job treats Object Storage buckets as opt-in,
  default-**false**, with an explicit comment — `instance-template/.github/workflows/
  llz-terraform.yml:796-798` — "destroying a cluster is not consent to erase them." Block
  Storage gets no equivalent consent gate; it is swept unconditionally and the destroy is
  hardened to fail rather than leave any behind.

**Why it matters:** this is the one delivered procedure in the whole review scope whose
undocumented side effect is irreversible. An operator with `Retain` on every PVC has no textual
reason to read "a full rebuild" as "and your data is permanently gone." The guardrail a
Kubernetes operator would reasonably lean on (the reclaim policy) points the opposite direction
from what actually happens.

**Recommendation:** add an explicit, loud warning to `first-build-failed.md` §6 stating that
destroy deletes all cluster-owned Block Storage Volumes regardless of `reclaimPolicy`, cross-
reference the mechanism from `docs/workflows/llz-terraform.md`, and consider requiring a
second confirmation token for destroys where cluster-tagged volumes exist and hold data (mirror
the bucket opt-in pattern). Pair with H-2 below (no DR design exists to tell an operator what to
do instead).

## C-2. The AI onboarding skill instructs running a CLI command that does not exist, at the final step of a ~40-minute build

**Impact: Critical** | **LoE to fix: Low**

**Evidence:**
- `.claude/skills/onboard-adopter/SKILL.md:49` — step 8, "Finish + verify": `llz bootstrap dns
  <env> --yes` (needs `LINODE_DNS_TOKEN`).
- Verified against the built binary: no `bootstrap` verb exists anywhere in the cobra tree
  (`llz --help` lists no such command; `llz bootstrap --help` falls through to root help).
- Both canonical docs say this step **does not exist by design**: `docs/quickstart.md:153-154`
  — "DNS-01 needs no step — the `llz-letsencrypt-*` ClusterIssuers sync via Argo CD once
  `LINODE_DNS_TOKEN` is set"; `docs/quickstart.md:915` — "no dedicated command";
  `docs/adopter-guide.md:404-405` — "DNS — no dedicated step." (Separately, per H-5 below, the
  named `llz-letsencrypt-*` issuers do not exist in this repo either — a second, deeper defect
  behind the same line.)

**Why it matters:** this is the file an AI agent loads when a human asks it to help onboard to
the landing zone. The agent will confidently instruct a human to run a command that exits
non-zero at the very last step of the build, at the exact moment an operator is least equipped
to tell a doc bug from a broken cluster — and the invented step implies DNS is a manual gate,
so the operator will wait for something that already happened.

**Recommendation:** one-line fix — replace the invented command with the "no dedicated step"
fact both canonical docs already carry. Land together with H-1 (the docs-guard gap that let
this ship undetected), since fixing only the text leaves the class able to recur.

---

# High

## H-1. `docs-guard` silently skips unknown/fabricated `llz` verbs — the structural reason C-2 survived two prior audits

**Impact: High** | **LoE to fix: Medium**

**Evidence:** `tools/cmd/llz/ci_docs_guard.go:195-244`. The guard walks `.claude/` (13 files,
confirmed scanned) and validates **flags against a resolved command** — but never asserts the
command path itself resolves. Probed the envelope with synthetic fixtures:

| Fixture | Guard result |
|---|---|
| `llz status lab --bogusflag` | finding, exit 1 (correct) |
| `llz env add lab --bogusflag` | finding, exit 1 (correct) |
| `llz bootstrap dns lab --yes` (unknown top-level verb) | `checked 0 llz invocation(s)` — no finding |
| `llz env bogus lab` (unknown subcommand of a real verb) | `checked 1 invocation(s), 0 findings` |

An unknown verb is dropped **before** the check runs, so it is invisible to both the numerator
and denominator of the guard's own "862 invocations checked" summary — that count reads as
coverage it does not have. This is the docs SKILL's own Rule 2 "silent success" class, turned
on the guard itself: a procedure that runs clean and proves nothing.

**Why it matters:** every `llz` command in every doc — delivered or not — is currently
unguarded against "this verb was renamed or never existed." C-2 is not a one-off typo; it is
the predictable output of this gap, and nothing currently prevents a second one.

**Recommendation:** the fix is a Go change (per docs SKILL Rule 6, this logic belongs in
`tools/cmd/llz`, not a shell/python script — `python-scripts` is ratcheted to 0), and per
`AGENTS.md:39-52` it needs a gate that fails when the behavior regresses: an oracle-style
fixture asserting an unknown verb produces a *finding*, not a skip. Expect fixing this to turn
the tree red across more of the 124 files — that is the point, not a regression.

## H-2. No disaster-recovery design exists at any status, and no delivered runbook explains what actually happens on rebuild

**Impact: High** | **LoE to fix: High**

**Evidence:**
- `docs/designs/README.md` indexes 20 designs across Shipped/Partial/Proposed/Superseded. None
  concerns backup, restore, or recovery — the status vocabulary the repo built specifically for
  "how much of this is real yet?" has never been applied to DR because DR was never written up.
- `velero` appears twice repo-wide (`grep -rn velero docs/ tools/ platform-apl/`), both naming
  it an apl-core **optional app that is off by default**
  (`docs/adr/0006-managed-default-apps.md:9`, `docs/adr/0005-managed-app-platform.md:215`).
  Nothing enables it, configures a backup target, or documents restoring from one. Zero
  references anywhere in `instance-template/`, `kubernetes-charts/`, or `terraform-modules/`.
  Sharper still: `llz import` itself *generates* an adopter migration doc with a "Data —
  migrate (Velero / dump-restore)" heading (`tools/cmd/llz/import_init.go:328`) while the
  platform ships no Velero — the generated doc implies a capability that does not exist.
- Grep for `RTO`, `RPO` across the whole repo: zero hits.
- What DR-adjacent capability genuinely exists — offline OpenBao seal-key escrow, a cluster
  re-create path framed as an incident wedge rather than DR, encrypted Terraform state as a
  rebuild substrate — is real and coherent (per-region independence, no cross-region
  replication, rebuild-from-Terraform-plus-OpenBao-escrow is a defensible posture) but is
  **nowhere stated as a model**, so nobody can review, test, or time it.
- `docs/runbooks/README.md`'s symptom index has no "the cluster is gone" row; the closest is
  `import-apl-site.md`, which is adopting a *pre-existing* site, not recovering this one, and
  explicitly excludes PV/Object Storage/database data migration (`:112-119`).

**Why it matters:** paired with C-1, an LKE landing zone marketed for enterprise adoption has
no documented cluster-rebuild procedure, no backup product deployed, and no RTO/RPO commitment
— a mature enterprise readiness review will not clear this regardless of how strong the
credential and observability engineering is elsewhere (and it is strong — see Done Well).

**Recommendation:** the cheapest honest first move is a `docs/designs/` entry at status
`Proposed` that writes down the rebuild-based model already implicit in what exists today —
per-region independence, encrypted TF state as substrate, OpenBao escrow as the seed — and
names what remains unresolved (PVC data, registry contents, RTO/RPO). That converts an
invisible absence into a tracked row, which is what the status vocabulary exists to do.

## H-3. The `llz render` → apl-core `values.yaml` seam: one doc states the mechanism correctly, five other surfaces — three of them delivered — still teach the retired one

**Impact: High** | **LoE to fix: Medium**

**Evidence:**
- `docs/landing-zone-spec.md:369-383` is the one place that correctly states the current
  reality: `llz render` does **not** write apl-core's `values.yaml` (LLZ runs only on managed
  App Platform, ADR 0005); `template-scripts/ci/scaffold-render-check.sh` *fails the build* if
  a render ever emits one. Three spec blocks (`spec.dns.*` incl. `acmeEmail`,
  `spec.defaults.platform.*`, `spec.alerting.*`) are flagged "validated but never rendered."
  Verified against `tools/internal/clusterspec/values.go:3-10`, which records the render
  pipeline's retirement outright, and confirmed `RenderValues` has no definition anywhere in
  the tree — every remaining hit is a stale comment. "Never rendered" is correct and if
  anything understated; "validated" over-claims for two of the three blocks (`spec.dns` and
  `spec.defaults.platform` have no validator function at all — only `spec.alerting` does; see
  M-30).
- **This does not propagate.** Four other sites treat the fields as live:
  - `docs/quickstart.md:497` — `llz spec set dns.acmeEmail=ops@example.com` presented as a
    normal, working example command in the copy/paste TL;DR path. `env_set.go:121,126-127`
    advertises the same dead example in the CLI's own `--help`.
  - `docs/runbooks/reconciler-alerts.md:41` — tells an operator responding to a live
    `LLZCertificatesNotReady` alert that deferred issuers are "expected NotReady until
    `spec.dns.acmeEmail` is set," implying setting it resolves the alert. It does not — and the
    `llz-letsencrypt-*` ClusterIssuers this line names **do not exist anywhere in the repo**
    (no `dns/` directory under `platform-apl/manifest/`, no `acme:` block anywhere in the tree
    — confirmed by direct grep, not inference).
  - **Delivered scaffold**: `instance-template/apl-values/README.md:8-9,63,77-78,89-90` and
    `instance-template/apl-values/values.yaml:9-15,28,30,360-362,408-420` — both ship into
    every adopter instance and both describe `llz render` writing `acmeEmail` into
    `platform-apl/manifest/dns/letsencrypt-clusterissuer.yaml` (a file that does not exist) and
    resolving `spec.alerting`/`spec.defaults.platform` into the base `values.yaml`. This
    directly contradicts `landing-zone-spec.md` and is the delivered artifact an adopter's
    mental model is built from.
  - Upstream source of the confusion: `tools/internal/clusterspec/types.go:57-66,109-136`
    carries doc-comments asserting the same retired mechanism as current — the next engineer
    who reads the Go types, not the Markdown, will re-derive the wrong model.
- `llz spec set dns.acmeEmail=…` and seeding `secret/alerts/webhooks` per a `spec.alerting.
  receivers` gate (`docs/secrets.md:295`) are both live instances of the Rule 2 "silent
  success" class: the command exits 0 and changes nothing that reaches a cluster.

**Why it matters:** an adopter reading the delivered README/`values.yaml` will conclude `llz
render` produces a file that CI actively fails the build for producing, and an operator
following `reconciler-alerts.md` during a live cert incident will act on a remedy that cannot
work and search for a Kubernetes resource that was never created.

**Recommendation:** fix the delivered scaffold (`apl-values/README.md`, `values.yaml`) first —
they are the highest-traffic surface — then the Go doc-comments in `types.go` (or the next
author re-derives the wrong model from source), then `quickstart.md:497` and
`reconciler-alerts.md:41`. `docs-guard` cannot catch this class (it validates flags/commands/
links, not attribution or resource existence); the enumerable piece (`spec.components`, see
M-9) is the tractable place to add a guard.

## H-4. `.claude/skills/onboard-adopter/SKILL.md` drops `TF_STATE_ENCRYPTION_PASSPHRASE` from the post-bootstrap escrow list

**Impact: High** | **LoE to fix: Low**

**Evidence:** `.claude/skills/onboard-adopter/SKILL.md:44-48` — step 7, "the **two** things the
bootstrap cannot do": seal key + recovery keys 4&5 + root token, and deleting
`OPENBAO_ROOT_TOKEN`. The canonical set is **three** escrow items plus the deletion
(`docs/quickstart.md:134-142`, `:634-638`): the same two, plus
`TF_STATE_ENCRYPTION_PASSPHRASE` — "saved offline — printed once by `llz tokens`; lose it and
every Terraform state file is unreadable" (`docs/quickstart.md:913`, referencing ADR 0007). The
string `TF_STATE_ENCRYPTION_PASSPHRASE` does not appear anywhere in the skill.

**Why it matters:** the only cached copy of this value is `.llz/secrets.env` on one laptop —
`docs/quickstart.md:748-751` calls that a *cache*, explicitly not escrow. An adopter coached by
this skill who loses that laptop has permanently unreadable Terraform state for the instance.
The skill's confident "the two things" framing reads as a complete enumeration, so a careful
reader has no reason to keep looking.

**Recommendation:** add the third item verbatim from `quickstart.md:913`. One-line fix.

## H-5. Delivered runbooks name Kubernetes resources and remedies that do not exist

**Impact: High** | **LoE to fix: Medium**

**Evidence:**
- `docs/quickstart.md:153-154,915` and `docs/runbooks/reconciler-alerts.md:41` all name
  `llz-letsencrypt-*` ClusterIssuers as syncing via Argo CD once `LINODE_DNS_TOKEN` is set.
  Repo-wide grep for `llz-letsencrypt` in any YAML returns only comments (`instance-template/
  apl-values/values.yaml:76,88`) — no manifest defines them, no `dns/` directory exists under
  `platform-apl/manifest/`, no `acme:` block exists anywhere in `platform-apl/` or
  `kubernetes-charts/`. This is consistent with the architecture, not an accident:
  `components.go:199-206` and `validate.go:574-577` confirm apl-core owns the DNS-01/wildcard
  cert on the managed platform and LLZ's own issuers were retired — the narration did not
  follow the retirement.
- `docs/quickstart.md:915` additionally claims `llz ci bootstrap-cluster` "renders" the DNS
  token into apl-core's DNS values. Verified against `ci_bootstrap_cluster.go:131-133,156-158`:
  this command explicitly does **not** install or configure apl-core; its only interaction with
  the token is a warning when unset (`dns_token_notice.go:48,54,58`). It warns; it does not
  render.

**Why it matters:** `quickstart.md` is in the `deliver-docs` keep-set and this is a checklist
item at the end of a ~40-minute build. It names a mechanism and a resource that both do not
exist, which makes the symptom unresolvable: an operator whose certs are `NotReady` will
`kubectl get clusterissuer` looking for a name that was never created. The same defect reaches
an incident runbook (`reconciler-alerts.md:41`).

**Recommendation:** name the actual ACME mechanism (apl-core-owned on managed platform) or
explicitly state that this repo creates no ClusterIssuer of its own and defer entirely to
platform documentation. Coordinate with H-3 — this is the same retired-mechanism class.

## H-6. `docs/runbooks/orphan-volume-cleanup.md` teaches the wrong volume-label prefix — the exact bug the previous audit fixed, recurring in a new form

**Impact: High** | **LoE to fix: Low**

**Evidence:** `docs/runbooks/orphan-volume-cleanup.md:33-38,62` states the volume-labeler
renames volumes to `<env>-<namespace>-<pvc>` and that the sweep's safe-filter matches
`pvc-*` or `<env>-*`. The real prefix is `<REGION_SHORT>-` — the **first three characters** of
the deployment name — per `tools/internal/linode/reap.go:118-140`, whose own comment records
that `<env>-` is precisely the string identified as a prior bug: "on `primary` the relabeler
writes `pri-harbor-…` and the reaper looked for `primary-…` — so the sweep stayed blind on
every deployment whose name is longer than three characters, i.e. all the real ones." The
runbook's own worked dry-run example (`:96-104`) uses `lab-…` — a **three-character** deployment
name, the one length at which the error is invisible — reproducing the exact blind spot the
source comment warns against. `docs/runbooks/volume-labels.md:29-42` states the correct format
and is verified against `ci_relabel_volumes.go:201-213`; the two delivered runbooks explicitly
cross-reference each other (`orphan-volume-cleanup.md:128`) and disagree.

**Why it matters:** this is the runbook whose entire purpose is preventing a clean-looking
sweep that deletes the wrong thing. An operator on a real deployment (`primary`, `staging`,
etc.) told to look for `<env>-*` will see `pri-*`/`sta-*` and conclude the invariant doesn't
apply, at exactly the moment they are deciding what to delete. The CLI itself is correct
(`--env primary` derives `pri` internally); the defect is purely documentary and lives in the
operator's head at the worst possible moment. Read together with C-1, this is the pair of
findings that most directly risks operator-triggered data loss.

**Recommendation:** correct the prefix to `<REGION_SHORT>-` (first 3 chars) in both the prose
and the worked example, and change the worked example off a 3-character deployment name so it
stops reproducing the blind spot it is meant to prevent.

## H-7. `reconciler-alerts.md` — the designated landing page for every in-cluster alert — forwards two of its most severe alerts to content that isn't there

**Impact: High** | **LoE to fix: Low**

**Evidence:** `docs/runbooks/reconciler-alerts.md:1-5` states every alert below carries a
`runbook_url` pointing at this file. Two of its own responses then point elsewhere and miss:
- `:51-54` — OpenBao's alerts (`OpenBaoSealed`, `OpenBaoNoActiveLeader`,
  `OpenBaoRaftQuorumDegraded`) are "in `bootstrap-openbao.md`." Grepped: they are not. The only
  file naming them is `docs/alerting.md:100-105`, which is **not** in the delivered set.
- `:37` — `LLZClusterNotConverged`'s response says "see `bootstrap-openbao.md` for the wedge
  classes (sync-wave, ESO timing)." Grepped: `bootstrap-openbao.md` contains no occurrence of
  `sync-wave`, `ESO timing`, or `wedge class`.

**Why it matters:** degraded Raft quorum on a 3-pod OpenBao is the precursor to losing the
secret store — every `ExternalSecret` in the cluster stalls behind it
(`reconciler-alerts.md:40` says this explicitly for a related alert). Both dead pointers fire
during a live incident, on the credential store and the convergence gate respectively.

**Recommendation:** either move the OpenBao alert-response table into the delivered set, or
write the response directly into `reconciler-alerts.md` instead of pointing elsewhere.

## H-8. Real converge-wedge classes are documented only in files that never ship

**Impact: High** | **LoE to fix: Medium**

**Evidence:** every concrete converge-stall mechanism found in the repo lives outside the
delivered set: `cert-manager-default-deny` starving the aggregated DNS-01 APIService (pod stays
`1/1 Running`, converge polls forever), `harbor-default-deny` blocking the CNPG operator, an NP
sync-wave race on cold bootstrap, a Kyverno image-signature admission race, an `argocd-redis`
auth split, and a destroy deadlock on ~60 Argo finalizers — all documented only in
`docs/lessons-learned.md:143-184` (explicitly scoped to "agents and contributors," not
operators) and `.claude/skills/netpol-change/SKILL.md` (a Claude-only skill, structurally
undeliverable). Zero of these appear in `docs/runbooks/`.

**Why it matters:** `apl-branch-recreate-wedge.md` proves the repo knows how to write this kind
of runbook excellently (symptom, signal set, proven remediation, timings) — the pattern exists,
it just was not applied to the other wedge classes, all of which present as a healthy-looking
pod with a hang somewhere unrelated.

**Recommendation:** a delivered `docs/runbooks/converge-wedges.md` or `networkpolicy-wedges.md`
sourced from `lessons-learned.md` + the `netpol-change` skill, indexed by symptom (see H-11).

## H-9. TLS/certificate-expiry has near-zero delivered coverage, and the token behind it is a documented but disconnected chain

**Impact: High** | **LoE to fix: Medium**

**Evidence:** the entire delivered coverage for cert expiry is one table row
(`reconciler-alerts.md:41`). Three separately-delivered facts compose into an outage nobody has
connected: `LINODE_DNS_TOKEN` is the DNS-01 solver's credential and challenges fail without it
(`bootstrap-openbao.md:88`); it is a Linode PAT under the ≤90-day expiry policy
(`linode-credential-rotation.md:33`); and it is "not yet wired into `secret-rotation.yml` and
remains manual" (`linode-credential-rotation.md:157-162`). A silently-expired token breaks
DNS-01 renewal weeks before `LLZCertificatesNotReady` fires, and the runbook that alert points
to never names the token as the likely cause. The `credential-single-pane` check does measure
this token's expiry — nothing connects that signal to the certificate alert.

**Recommendation:** add a cross-reference from `reconciler-alerts.md`'s cert row to the
`LINODE_DNS_TOKEN` manual-rotation fact, and consider wiring the token into automated rotation
(engineering fix, out of docs scope but worth flagging — it is the root cause).

## H-10. Two credential-rotation failure modes have no recovery procedure for a half-succeeded rotation

**Impact: High** | **LoE to fix: Medium**

**Evidence:**
- **TF-state key** — `linode-credential-rotation.md:225-248` gives a clean create-then-revoke
  and then only says what *not* to do ("do not hand-swap this credential"). Per
  `llz-secret-rotation.yml:360-363`, the rotated pair is written into *every* deployment's
  environment with no transaction. A partial write leaves some `infra-<env>` environments on
  the new key and some on the old (old key still live), and nothing tells the operator how to
  detect which environments got it or how to converge them.
- **`lke-admin` rotation** — the documented sequence (`lke-admin-rotation.md:27-46,78-83`) is
  delete-kubeconfig → `tofu apply -refresh-only` → health gate, and the refresh-only step is
  named as "the single propagation point — nothing else pushes the rotated kubeconfig"
  (`llz-secret-rotation.yml:193-198`). If delete succeeds and refresh-only fails (a trap the
  runbook itself flags at `:85-90`), Terraform state holds a **dead** kubeconfig and every CI
  consumer is broken until someone re-runs it. The runbook's "Verification" section
  (`:108-113`) covers only the happy path. The actual recovery primitive exists —
  `llz ci fetch-kubeconfig` reads from the Linode API directly (`operator-onboarding.md:
  103-105`) — but nothing in `lke-admin-rotation.md` points at it.

**Recommendation:** add a "partial rotation" section to both runbooks. For `lke-admin`, link
`fetch-kubeconfig` as the recovery path from within the runbook itself.

## H-11. NetworkPolicy/Cilium wedge class has zero delivered coverage, and the one playbook that walks an operator into a first workload never mentions it

**Impact: High** | **LoE to fix: Medium**

**Evidence:** grepped `docs/runbooks/` + `docs/playbooks/` for `cilium|networkpolicy|netpol`:
two incidental hits, neither a procedure. Every real trap (post-DNAT policy evaluation meaning
apiserver egress needs `6443` not `443`; aggregated APIService webhooks needing their own
`:443` allow; NP sync-wave races; ESO default-deny blocking its own admission webhook; Istio
sidecar egress; STRICT mTLS as an invisible second policy layer no NetworkPolicy edit can fix)
lives only in `lessons-learned.md:143-184` and `.claude/skills/netpol-change/SKILL.md` — both
non-delivered. `docs/playbooks/first-workload.md:11-14` explicitly advertises "the default-deny
network baseline" as a platform feature, then walks four steps (namespace, Harbor pull,
ExternalSecret, HTTPRoute) with zero mention of NetworkPolicy, Cilium, mesh, or STRICT mTLS —
and its "When it doesn't work" table (`:177-186`) has no connectivity-failure row at all.

**Why it matters:** `netpol-change/SKILL.md:8-9` states the shared signature precisely: "A
NetworkPolicy that matches nothing looks exactly like a healthy cluster." This is the highest
symptom-to-cause-distance failure class on the stack, aimed at the one playbook meant for an
operator doing something genuinely new.

**Recommendation:** same shape as H-8 — a delivered runbook sourced from the existing
non-delivered material, indexed by symptom ("pod is Running and healthy but X cannot reach
it," "converge stalls on an APIService").

## H-12. A finished, tested CIS control-mapping and evidence-pack branch has sat unmerged for ~6 weeks while an ADR on `main` already assumes it exists

**Impact: High** | **LoE to fix: Low** (the work is done; this is a merge decision)

**Evidence:** branch `feat/cis-evidence-pack` exists on both remotes; commit `f216552`
(2026-06-28, +1,447 lines across 15 files) adds `docs/infosec/control-mapping.md`, Trivy
`ClusterComplianceReport` CRs, `llz ci cis-evidence` + full test coverage, and 100 lines of
scheduled-checks wiring. Verified **not merged**: `git merge-base --is-ancestor f216552` fails
against `main`/`gh-fork-lke/main`/`gh-upstream/main`; `grep -rn "cis-evidence\|internal/
evidence" tools/` on main returns zero hits. The branch's control-mapping doc carries the
**LKE-Enterprise shared-responsibility carve-out** (the managed control plane is not
user-auditable, so those CIS sections are out of scope by design) — the single most-asked
question an enterprise auditor puts to a managed-Kubernetes landing zone, and on `main` there
is currently no answer to it anywhere. `docs/adr/0013-llz-as-apl-cli.md:95,212` already plans
around this work ("Re-home CIS evidence… under `apl doctor`/compliance") — an ADR on `main`
assumes a capability `main` does not have.

**Recommendation:** this is the highest-leverage, lowest-effort item in the whole audit —
review and merge the branch.

## H-13. No compliance-framework mapping of any kind exists on `main`

**Impact: High** | **LoE to fix: High** (absent the branch in H-12; Low if H-12 merges)

**Evidence:** grep across all `*.md`/`*.go`/`*.yaml`/`*.yml` for `CIS` (word-boundary), `SOC 2`/
`SOC-2`, `ISO 27001`, `FedRAMP`, `PCI-DSS`, `NIST`: zero substantive hits on `main` (every `CIS`
match was a substring of "deterministic"/"decision," except three planning references in ADR
0013 pointing at the unmerged branch). The only compliance-shaped artifact is the generic SOG
requirement table in `docs/infosec/linode-account-request-checklist.md:63-81`, which maps to no
published framework.

**Recommendation:** subsumed by H-12 if that branch lands; otherwise this is a from-scratch
gap.

## H-14. No SLO documentation exists anywhere — the repo has alert-rule documentation only

**Impact: High** | **LoE to fix: High**

**Evidence:** grepped `docs/` case-insensitively for `SLO`, `error budget`, `availability
target`, `SLI`, `uptime target`, `99.9`: zero hits for every term. All 20+ `SLA` hits are
credential-rotation SLAs (90-day PAT, 120-day bucket key) — not one is a service-availability
SLA. The closest artifact, `docs/alerting.md:139-141`'s "one availability + one error-rate + one
resource-saturation alert per service" bar, is an alert-*coverage* target, explicitly not an
error budget. Confirmed structurally: all 43 alerts across the platform's four
`PrometheusRule` files are threshold/absence alerts (`up == 0`, `> N`, `absent()`) — there are
no multi-window multi-burn-rate rules, no `slo:` recording rules, no error-budget series
anywhere. The default Alertmanager receiver is `[none]` until an instance opts in
(`docs/alerting.md:22-26`) — a platform whose default notification path is a null route has, by
construction, no error budget it could be measuring yet.

**Why it matters:** "what availability does this landing zone commit to?" and "does this alert
mean we breached something we promised?" have no answer in this repo — invisible in an
alert-by-alert review precisely because the alerting itself is genuinely good (100%
`runbook_url` coverage, confirmed below in Done Well).

**Recommendation:** out of scope to design here; flagging as the gap between a
well-instrumented platform and one an enterprise can contract against.

## H-15. Security posture is real but scattered across 15+ files with no security-facing index; `docs/infosec/` is a procurement checklist, not an architecture doc

**Impact: High** | **LoE to fix: Medium**

**Evidence:** `docs/infosec/` contains exactly one file — a checklist for *getting a Linode
account approved*, not a statement of the landing zone's own security posture. Mapped every
standard review domain against what actually exists: credential lifecycle is genuinely
enterprise-grade (`docs/secrets.md`, two ADRs, four rotation runbooks); encryption at rest is
real and gated but has no single at-rest statement; encryption in transit is known-incomplete
and the closure work is reserved as ADR 0011, **not written**; access control has no
consolidated "who can do what" doc and no RBAC/Pod Security Standards doc on `main`; the
kube-apiserver audit story is entirely absent (the "managed, not user-auditable" answer exists
only on the H-12 branch); incident response is nothing security-shaped at all — no severity
model, no escalation contract; data residency has one incidental hit despite the platform being
explicitly multi-region.

**Why it matters:** none of this is a hidden defect — the underlying security engineering is
strong. The defect is navigability: a reviewer handed this repo has one directory named
`infosec/` containing a procurement checklist and must reconstruct the actual posture from 3
ADRs, 4 designs, 5 runbooks, and a 767-line secrets guide.

**Recommendation:** this is exactly what `control-mapping.md` (H-12) does — a second, stronger
argument for landing that branch. Absent that, a `docs/infosec/README.md` index over the
existing material would close most of the gap cheaply.

## H-16. `terraform-modules/RELEASING.md`, the `release` skill, and `.github/workflows/AGENTS.md` describe a `firewall-controller` build that does not exist in this repo

**Impact: High** | **LoE to fix: Low**

**Evidence:**
- `terraform-modules/RELEASING.md:19,83` states the umbrella release tag versions "the
  `firewall-controller` image (tagged `:vX.Y.Z` by `firewall-controller.yml`)" and lists
  `firewall-controller.yml` in the release-promotion table alongside `llz-release.yml`.
  `.claude/skills/release/SKILL.md:20-22,61-63` repeats both claims for the benefit of an
  agent cutting a release.
- No `firewall-controller.yml` exists in `.github/workflows/` (confirmed by directory listing
  and `git log --all -- .github/workflows/firewall-controller.yml`, which returns nothing
  across any branch on either remote) — this file has never existed in this repo's history.
- `.github/workflows/AGENTS.md:50,67` separately claims the `ci-tofu` CI image "bundles the
  `firewall-cidrs` Go binary" and that `ci-tofu` "ships `gh` and the prebuilt Go CLIs (`llz`,
  `firewall-cidrs`) on PATH." Confirmed false: `dockerfiles/` and `build-images.yml` build only
  the `llz` binary; `firewall-cidrs` appears nowhere in either.
- The actual state, correctly documented in three *other* places (`README.md:160-163`,
  `AGENTS.md:64`, `CONTRIBUTING.md:42`, `docs/consume-lke-landing-zone-internal.md:1-7`): both
  `firewall-cidrs` and `firewall-controller` moved to the private `lke-landing-zone-internal`
  repo as an Akamai-internal feature. This public repo builds only the pieces that are safe to
  ship (`terraform-modules/llz-cluster/firewall.tf`, the `cidrFirewall` spec component,
  `llz ci bootstrap-cloud-firewall`).

**Why it matters:** an agent following `.claude/skills/release/SKILL.md` to cut a release would
check for a `firewall-controller` image build that will never appear in this repo — the
migration to the internal repo was never propagated to these three places.

**Recommendation:** update `RELEASING.md` and the release skill to drop the
`firewall-controller.yml`/image claims (or explicitly note they apply only in the internal
repo); correct `.github/workflows/AGENTS.md`'s `ci-tofu` bundle description to `llz` only.

## H-17. `template-scripts/AGENTS.md` instructs a `git config core.hooksPath` command that points at a directory that does not exist, silently disabling every git hook

**Impact: High** | **LoE to fix: Low**

**Evidence:** `template-scripts/AGENTS.md:44` — "Enable with `git config core.hooksPath
scripts/hooks`." No `scripts/` directory exists at the repo root (confirmed by `ls`) — the
directory was renamed to `template-scripts/` at some point, and this nested `AGENTS.md` (which
lives *inside* `template-scripts/` and still titles itself "# scripts/" throughout) was never
updated. Three other files give the correct path: `CONTRIBUTING.md:27`, root `AGENTS.md:151`,
and `docs/agents.md:31` all say `git config core.hooksPath template-scripts/hooks`, matching
the real directory (`template-scripts/hooks/pre-commit`, `pre-push` both exist on disk).
`git config core.hooksPath` silently accepts a nonexistent path — no error, hooks simply never
fire — which is precisely the "silent success" class `docs/playbooks/operator-onboarding.md:
62-65` separately warns about for the instance-repo equivalent of this exact command.

**Why it matters:** a contributor who reads this file specifically (it is the nested AGENTS.md
that governs `template-scripts/`, so someone working there is likely to open it) and follows
its literal instruction disables the secret-file pre-commit guard and the pre-push lint gate
with zero indication anything is wrong.

**Recommendation:** fix the path; while there, rename the file's internal references from
"scripts/" to "template-scripts/" throughout — the whole file is a stale-rename artifact, not
just this one line.

## H-18. ADR 0010's status line contradicts both the ADR index and the ADR's own body

**Impact: High** | **LoE to fix: Low**

**Evidence:** `docs/adr/0010-in-cluster-mtls.md:3` — "Status: **Proposed** — implemented but NOT
validated on a live cluster." `docs/adr/README.md:24` lists it as **Accepted**. The file's own
body refutes its status line: `:380-382` strikes through prerequisite 1 as "FALSE — checked on
a live cluster (e2e, 2026-07-29)"; `:426-428` confirms prerequisite 2 "CONFIRMED on the live
e2e cluster." A later, Accepted ADR (`0012-credential-observability-gaps.md:5-9`) builds on
0010 as settled, stating it "was correct about the mechanism." The mechanism is live and
statically gated (`llz ci mtls-wiring-guard` exists in the cobra tree, verified).

**Why it matters:** this is the fail-closed mTLS posture. A reader triaging a handshake failure
who reads "Proposed / not validated" may conclude the posture is not actually live and disable
it — exactly the "stale status text conceals current state" class the docs SKILL's Rule 9
warns about, applied to an ADR status field rather than prose.

**Recommendation:** reconcile to `Accepted`, using the repo's own existing idiom for partial
completion (ADR 0007-terraform-state-encryption's "accepted, phase 1 shipped. Phase 2 is a
follow-up.") — name the genuinely-open prerequisites (3, 4, 5) as the residue.

## H-19. Two design docs' safety/status claims are false against the current tree, in ways that understate risk or shipped work

**Impact: High** | **LoE to fix: Low**

**Evidence:**
- `docs/designs/blast-radius-decomposition.md:3-7` (and `docs/designs/README.md:48`) — status
  `Partial`, naming "decomposing the four toggleable kustomize Components into independently
  health-gated Argo Applications" as the remaining work. All four prescribed Applications exist
  in `tools/internal/clusterspec/components.go` at exactly the design's prescribed sync-waves
  (verified line-for-line), the renderer implements both required artifacts, and the design's
  own follow-on requirement (`wave-dependency-guard` gaining cross-Application awareness) is
  also shipped (`ci_wave_dependency_guard.go:291-292`). This is the *inverse* of the safe
  error: a reader planning work will either re-implement a shipped decomposition or wrongly
  conclude the blast-radius containment claim is unproven.
- `docs/designs/obj-sse-c-gateway.md:3-4,186`, `docs/designs/README.md:53`, and
  `platform-apl/components/objProxy/obj-proxy/kustomization.yaml`'s own comment (four places
  total) state "the DNS rewrite that activates it is deliberately outside the kustomization" as
  a safety mechanism (manual step 5). The sibling file the kustomization actually executes
  (`platform-apl/components/objProxy/obj-proxy/kustomization.yaml:21-26`) shows the opposite,
  deliberately: the rewrite **is** in the kustomization now, ordered by sync-wave (10 vs. 5)
  with a pod-roll backstop (`llz ci harbor-trust-obj-proxy-ca`). The design predates a real
  redesign of the safety mechanism, and the stale text understates what enabling the component
  does — this component sits on the write path of every image pull and every log write. Track D
  flagged this may be a code-comment bug rather than only a docs bug (the kustomization file's
  own header comment contradicts the file's own executed content) — worth the component owner's
  attention alongside the docs fix.

**Why it matters:** `docs/designs/README.md`'s own rule states "a design whose status is stale
is worse than one with no status, because the reader trusts it" — both of these are exactly
that failure, in the two directions (understating shipped work; misstating a safety mechanism)
that cost the most.

**Recommendation:** `blast-radius-decomposition` → `Shipped` (note the two extra carved
components beyond original scope). `obj-sse-c-gateway` → correct all four mentions to describe
the actual wave-ordering + pod-roll-backstop mechanism; the caution is still valid, the
mechanism it describes changed.

## H-20. The `.untestable-budget.yaml` ratchet's entire rationale lives only as a YAML comment — the highest-value undocumented decision found

**Impact: High** | **LoE to fix: Low**

**Evidence:** Track D's sub-task 3 (decisions made in code with no ADR) surfaced nine
candidates; most are recorded-but-mis-filed (see M-31/M-32) rather than truly undocumented, but
`.untestable-budget.yaml` — the repo-wide ratchet that caps non-Go logic per category and drove
`python-scripts` to 0 (cited throughout this report, e.g. H-1's recommendation) — has no ADR and
no design doc. Its entire rationale is a comment in the YAML file itself. This is a load-bearing,
cross-cutting policy (it shapes where every future fix in this report must be implemented) with
a single point of failure for institutional memory: whoever edits or removes that comment
removes the only record of why the ratchet exists.

**Why it matters:** this report alone cites the ratchet's consequence three times (H-1, and the
docs SKILL's own Rule 6 origin story about the 74-line Python script that failed at 74/60). A
decision this frequently load-bearing belongs in a location a reader would think to check before
editing it away.

**Recommendation:** a short ADR capturing the ratchet's rationale and the incident that produced
it (the 74-line Python script), linked from the YAML file's own header comment.

---

# Medium

## M-1. Contributor entry point is split across two files, each pointing at the other as canonical

**Impact: Medium** | **LoE to fix: Low**

`CONTRIBUTING.md` (88 lines) and `AGENTS.md` (180 lines) overlap on prerequisites, build, lint,
hooks, and commit style, with `AGENTS.md` strictly larger on every shared topic.
`CONTRIBUTING.md:79` calls `AGENTS.md` "the canonical instruction file"; `AGENTS.md:1` calls
itself canonical "for AI agents **and contributors**" — both point outward, neither is the
destination. GitHub auto-surfaces `CONTRIBUTING.md` in the PR-compose UI; `AGENTS.md` it does
not. Content that exists **only** in `AGENTS.md` and that a human routed through
`CONTRIBUTING.md` will never see: the "name the gate" PR requirement (the single rule the repo
cares most about, backed by two named production regressions), the umbrella-tag/SemVer
publishing contract, and the "no org-identity hardcoding"/"scars as defaults" conventions.
`CONTRIBUTING.md` mentions `AGENTS.md` only under a heading called "AI assistant instructions,"
which actively signals to a human it isn't for them. **Recommendation:** fold the load-bearing
contributor content that only lives in `AGENTS.md` into `CONTRIBUTING.md` directly, or retitle
the heading so a human doesn't self-select out.

## M-2. `docs/e2e-gates.md` and `docs/lessons-learned.md` are unreachable from the root README

**Impact: Medium** | **LoE to fix: Low**

Both linked only from `AGENTS.md`. Neither appears in `README.md`'s Documentation or
Operations tables. `e2e-gates.md` is the mandatory pre-read before adding any behavior
(`AGENTS.md:47`); `lessons-learned.md` is the mandatory skim before non-trivial work
(`AGENTS.md:18-21`). A contributor arriving via the README front door cannot find either.
**Recommendation:** add both to `README.md`'s "Operations & architecture" table.

## M-3. `docs/extending-llz.md` — the adopter's `owned`-file escape hatch — is reachable only from one buried bullet

**Impact: Medium** | **LoE to fix: Low**

Full inbound-link set: `docs/adopter-guide.md:286`, one bullet inside a subsection about git
hooks. Not in `README.md`'s Documentation table, not in `quickstart.md`'s "See also." An
adopter looking for "how do I add my own `llz` commands without `llz upgrade` clobbering them"
— precisely the question asked *after* a first upgrade conflict — will not find it by browsing.
**Recommendation:** add a row to `README.md`'s Documentation table.

## M-4. README and quickstart disagree on build duration by 2x

**Impact: Medium** | **LoE to fix: Low**

`README.md:260` — "~20 minutes." `docs/quickstart.md:129,615` — "~40 minutes" (twice). Every
other "20 minutes" reference is a sub-stage figure consistent with a 40-minute total
(`quickstart.md:108,709,873`, `adopter-guide.md:241`) — `README.md` is the outlier, and it's
the doc a first-time reader hits first. This is the number an adopter plans their afternoon
around, off by 2x in the direction that makes a healthy build look hung.
**Recommendation:** fix `README.md:260` to ~40 minutes.

## M-5. The onboarding skill's verify step teaches the exact failure the quickstart explicitly warns about

**Impact: Medium** | **LoE to fix: Low**

`.claude/skills/onboard-adopter/SKILL.md:56` shows `llz status <env>` with no `--wait`, at the
exact moment `docs/quickstart.md:155-160` warns a bare `llz status` polls once and a red ✗
seconds after a build is the *normal* first answer, not a failure. Three sources disagree on
timeout too: the skill (no flag), `quickstart.md:160,676` (`--wait`, default 300s), `README.md:
268` (`--wait --timeout 900`, "convergence is slower"). If the README's own comment is right,
the delivered doc carries the weaker advice. **Recommendation:** add `--wait --timeout 900` to
the skill's verify step; reconcile the three timeout values.

## M-6. The onboarding skill never says to export `LINODE_TOKEN` before `llz env add`

**Impact: Medium** | **LoE to fix: Low**

The skill explains *why* values need account-checking (step 4) but never says to export the
token that makes the check happen — the string doesn't appear anywhere in the skill.
`docs/quickstart.md:105-111,906` treats this as a first-class, checklist-level step: without
it, every account-shape check silently skips and a typo'd region is first caught by
`terraform apply`, 20 minutes in. The runtime skip is a loud stderr warning
(`account_check_skip.go:70-75`), not silent — which caps this at Medium rather than a
Rule-2 silent-success class — but an agent following the skill has no reason to surface it.
**Recommendation:** add the export step.

## M-7. Delivered playbooks require tools that `llz doctor` — called "authoritative" in five places — never checks

**Impact: Medium (High for `crane`, which has no documented install path at all)** | **LoE to fix: Medium**

`wizard.go:420` — the actual toolchain `llz doctor` probes — lists `git, copier, gh, kubectl,
helm, bao, jq, linode-cli`. No `yq`, `crane`, `trivy`, `syft`. Yet delivered docs require them:
`first-workload.md:162` and `argocd-ops.md:70` need `yq` (documented only via the
non-delivered `docs/devcontainer.md:37`); `operator-onboarding.md:82,85` need `crane`, `trivy`,
`syft` with **no install path documented anywhere in the repo**. Five docs
(`quickstart.md:203,226`, `adopter-guide.md:48`, `CONTRIBUTING.md:13`,
`operator-onboarding.md:58`) call `llz doctor` "the authoritative, always-current list."
**Recommendation:** add the four tools to `llz doctor`'s check list, or scope the
"authoritative" claim to exclude playbook-specific tooling.

## M-8. The zero-to-converged walkthrough is written twice at full length and has already drifted

**Impact: Medium** | **LoE to fix: Medium**

`README.md:223-273` and `docs/quickstart.md:74-161` are near-identical ~40-80 line copy/paste
paths (same prereq block, same `read -rs` idiom, same command sequence). Observed drift today:
build duration (M-4), `--wait`/`--timeout` presence (M-5's sibling), post-build escrow framing.
This is the docs SKILL's Rule 11 duplication class, already producing observable drift rather
than remaining a latent risk. **Recommendation:** make one canonical (quickstart.md is the
natural owner); trim README's copy to a teaser that names the next step.

## M-9. `landing-zone-spec.md`'s component set is missing `objProxy`, repeated across four locations

**Impact: Medium** | **LoE to fix: Low**

`docs/landing-zone-spec.md:288-303` (repeated at `:144-145,194-195,289`) enumerates the
default-off components; the registry (`components.go:371-374`) has 18 entries, the doc lists
17 — `objProxy` (`DefaultDisabled: true`, depends on `externalSecrets`) is absent from all four
enumerations. It is documented only in `docs/designs/obj-sse-c-gateway.md`, a dated design
record, not the spec reference an operator authoring `spec.components` would consult. Mitigated
by the doc explicitly deferring to `llz components` as authoritative — but not for a reader who
trusts the prose. `docs-guard` cannot catch this (it validates flags/commands, not component
enumerations). **Recommendation:** add `objProxy` to all four locations; consider a guard
comparing `llz components`' live output against the doc's enumerated list.

## M-10. ADR 0013 is `Proposed` while its own appendix marks five items "Align (done)," and the code agrees

**Impact: Medium** | **LoE to fix: Low**

`docs/adr/0013-llz-as-apl-cli.md:3` and the index both say `Proposed`. The ADR's own Appendix B
marks five dispositions "Align (done)" (`users`→`apl user`, `components`→`apl app`,
`render`→`apl values render`, `validate`→`apl values validate`, `openbao`→`apl openbao`) — all
five verified present in the built binary — and Phase 0's package boundary
(`internal/apl/`+`internal/provider/`) is not just present but test-enforced. Severity capped
at Medium because the direction is conservative (understating adoption is the safer error).
**Recommendation:** `Accepted — Phase 0 landed (the apl subtree + the package boundary);
Phases 1–6 outstanding`, changing file and index together.

## M-11. ADR status-line format is not uniform, blocking any future mechanical status↔index gate

**Impact: Medium (structural)** | **LoE to fix: Medium**

Two mutually exclusive shapes coexist: a `- Status:` metadata block (0002/0003/0004/0013) vs. a
bare `Status: **value**` paragraph (0005/0006/0007-app/0007-tf/0008/0009/0010/0012). A
grep pass anchored on `^(\*\*)?Status` misses the first shape entirely — a false negative that
would let H-18-class drift recur silently. Several ADRs in the second shape also carry no
`Date`/`Deciders` line at all, with the index as the only place that data exists.
**Recommendation:** normalize to the `- Status:`/`- Date:`/`- Deciders:` block, backfilling
missing fields, then add an index↔file status assertion to `docs-guard` (reads only the status
field, so it does not violate the Rule 4 body exemption).

## M-12. `e2e-instrumentation` design is `Shipped` under a command name that does not exist

**Impact: Medium** | **LoE to fix: Low**

`docs/designs/e2e-instrumentation.md:3` and `docs/designs/README.md:37` both say "phase timing
landed as `llz ci phase-timing`." That command does not exist (`llz: unknown command
"phase-timing"`); the capability shipped under `llz ci phase-mark` and `llz ci phase-report`,
named correctly in the design's own body. `docs-guard` exempts `docs/designs/` bodies from the
command check, but the **status line** is where an exempt directory leaks a live claim — same
structural gap as H-19. **Recommendation:** correct both mentions to the real verb names.

## M-13. Two design index rows understate their own file's shipped work

**Impact: Medium** | **LoE to fix: Low**

`forge-abstraction` — `docs/designs/README.md:51` says "the GHE/GitLab flavours are unbuilt";
the file's own status and the tree (`tools/internal/forge/gitlab_capabilities.go`,
`github_app.go`, each with `_test.go` **and** `_mutation_test.go`) both say Phases 2, 3, and 6
(the GitLab capabilities) landed. `shared-managed-postgres` — index and file both say "the
OpenBao seed command and CI jobs are outstanding"; all three verbs
(`seed-db-admin`/`rotate-db-admin`/`db-declared`) exist and the file's own later sections
describe them as built — the status line was written before those sections and never revised.
**Recommendation:** update both status lines and index rows; these are the only two cases where
index and file agree with each other and both disagree with the tree.

## M-14. Four design status lines carry unresolvable author-seat branch references

**Impact: Medium** | **LoE to fix: Low**

`apl-core-values-branch-isolation.md:3` ("Shipped — implemented on the branch that carries this
file"), `forge-abstraction.md:3`, `apl-core-v6-migration.md:3`, `apl-overlay-obj-native.md:3`
all name a feature branch in the status line; none resolves in the repo today
(`git rev-parse --verify` fails for all four). Same class the docs SKILL's Rule 9 already fixed
in `alerting.md` and Rule 10 fixed in Go comments — missed here because prior audits treated
`docs/designs/` as archival and fully exempt; the exemption covers bodies, not status lines.
**Recommendation:** rewrite as "landed on `main`" (three of four already are) per Rule 9's
standing-fact-plus-version pattern.

## M-15. `apl-core-v6-migration`'s lab-validation caveat is stale against a baseline that has moved two releases past it

**Impact: Low-Medium** | **LoE to fix: Low** (needs an owner who knows current lab state)

`apl-core-v6-migration.md:3-5` is `Partial`, pinned to `v6.0.0`, 11 of 13 lab-validation items
unchecked; the actual baseline is `v6.1.0` (`aplversion.go:30`), with `apl-core-v61-upgrade`
sitting on top of it and carrying the same "validate in lab" caveat. Cannot be settled from
source — flagging that the status field is carrying two different meanings at once ("the code
move is incomplete" vs. "the code moved but was never lab-verified"), which is the same root
cause as M-14's `apl-core-values-branch-isolation` note. **Recommendation:** whoever owns
current lab state should split these meanings; per Rule 8 these read as "records of a move that
already happened" and likely belong at `Shipped` or `Superseded`.

## M-16. A design document titled `# ADR:` is not in the ADR index

**Impact: Medium** | **LoE to fix: Low**

`docs/designs/apl-core-values-branch-isolation.md:1` — "# ADR: apl-core writes to a per-env
branch…" — is structurally an ADR (`## Decision`, `## Consequences`, `## Alternatives
considered`, a `**Date:**` field no other design carries) but lives in `docs/designs/`,
unnumbered, absent from `docs/adr/README.md`. The decision it records is cross-cutting: cited
by ADR 0013, two other designs, and enforced in code by a validator
(`clusterspec/validate.go:391`). A reader searching the ADR index for the `apl-<env>` branch
decision finds nothing — it can only ever be cited by file path, never by number, breaking the
citation discipline the repo otherwise maintains carefully (see the two-`0007` note in the ADR
README). Track D additionally found this design is marked `Shipped` while its own
"Lab-validation checklist (gate before promoting past lab)" has 4 of 5 boxes unchecked,
including "release-e2e green end-to-end" — under the README's "prefer the weaker claim" rule
this reads as `Partial`, not `Shipped`, though the code genuinely is on `main`.
**Recommendation:** promote to `docs/adr/0014-…` (next free number *from the table*) with a
pointer left behind, or demote the H1 to a design title — the promotion is correct given who
cites it. Separately, reconcile the `Shipped`/`Partial` tension against the unchecked
lab-validation boxes.

## M-17. `docs/alerting.md` claims to be the complete alert inventory; it names 25 of 43 alerts, including the highest-severity credential alert

**Impact: Medium** | **LoE to fix: Low**

`docs/alerting.md:3` — "this page catalogues **every** item… it is the inventory." Checked
every alert name (43 total across four `PrometheusRule` files) against the file: 18 never
appear, including `LLZCredentialRootTokenParked` (ADR 0012, 2026-07-30) — arguably the single
highest-severity alert shipped (it detects a live, unexpiring, full-admin OpenBao root token
left by a half-run break-glass) — documented only in `docs/secrets.md` and ADR 0012, not the
file that advertises itself as the name-searchable inventory. Nuance: `alerting.md` does cover
much of this ground organized by item/mechanism rather than alert name, so coverage is not as
thin as 25-of-43 suggests — but an operator paged by name greps for the name.
**Recommendation:** either narrow line 3's claim or add the 18 missing rows; prioritize the four
credential alerts from ADR 0012.

## M-18. `docs/alerting.md`'s own heading is wrong about OTel scrape-gating

**Impact: Low-Medium** | **LoE to fix: Low**

`docs/alerting.md:143` — "Open gap — Harbor and OTel are not scrape-gated." `otel-collector-
monitoring` **is** in `defaultScrapeMonitors` (`ci_assert_scrape.go:50`); the paragraph body
discusses only Harbor and never returns to OTel, so the heading is the sole carrier of a false
claim (OTel's genuinely-provisional piece is its *alert-arming*, a different failure class,
correctly described at `:133`). An operator acting on the heading would find the entry already
present and either distrust the accurate Harbor half two lines below or add a duplicate.
**Recommendation:** drop "and OTel" from the heading.

## M-19. `docs/infosec/` has no README index — the only doc subdirectory without one

**Impact: Low now, Medium the moment H-12 lands** | **LoE to fix: Low**

`docs/adr/`, `docs/designs/`, `docs/runbooks/`, `docs/playbooks/`, `docs/workflows/` all carry
an index; `docs/infosec/` does not. Defensible at one file; becomes two the moment H-12's
`control-mapping.md` lands, and the docs SKILL's Rule 8 is explicit that a growing directory
gets its own index. **Recommendation:** add one now, ahead of H-12.

## M-20. No symptom-indexed runbook entry for "the monthly rotation run went red"

**Impact: Medium** | **LoE to fix: Low**

Rotation runs monthly across every deployment, Environment-approval-gated. `docs/runbooks/
README.md`'s "By symptom" table has no rotation-failure row — rotation appears only under
"Scheduled / on-demand rotation," which reads as a happy-path index, not a failure entry point.
**Recommendation:** add a symptom row pointing at the relevant rotation runbook's failure
section.

## M-21. `llz-scheduled-checks.md` contradicts `linode-credential-rotation.md` on how OBJ-key rotation actually works

**Impact: Medium** | **LoE to fix: Low**

`docs/workflows/llz-scheduled-checks.md:135-140` — "the object-storage Terraform module
force-rotates the key declaratively (`time_rotating`), but the OpenBao reseed hop is manual."
`docs/runbooks/linode-credential-rotation.md:35,180-186` states the opposite, twice: rotation
is "now automated in-cluster by the llz-reconciler… the TF-managed keys and their
`time_rotating` clock were removed… No operator action in steady state." Corroborated by
`llz-terraform.md:661-665`. The runbook is correct; the workflow doc carries a stale
present-tense claim (not a "where did X go?" historical note, which Rule 9 would exempt — this
asserts the old mechanism is still how it works today). Capped at Medium since
`docs/workflows/` never ships. **Recommendation:** update `llz-scheduled-checks.md:135-140` to
match the runbook.

## M-22. `docs/workflows/llz-secret-rotation.md` names a workflow file that does not exist

**Impact: Medium** | **LoE to fix: Low**

`:58` — "the previous PAT drains daily via `linode-pat-revoke.yml`." No such file exists among
the 16 files in `instance-template/.github/workflows/`. `linode-pat-revoke` is a `scope` value
and a concurrency-group name inside `secret-rotation.yml`/`llz-secret-rotation.yml`, not a
standalone workflow — and the same document describes it correctly, as a job, 150 lines later
(`:317-336`). **Recommendation:** fix line 58 to say "job," matching `:317-336` and
`linode-credential-rotation.md:104-106`.

## M-23. A broken markdown blockquote fractures the highest-stakes manual checklist in the repo

**Impact: Medium** | **LoE to fix: Low**

`docs/runbooks/bootstrap-openbao.md:164-183` — "After first-time bootstrap — required operator
actions." The blockquote opens with "⚠️ These steps are not automated. Do not skip them," then
an unclosed bold marker at `:168` causes lines `:170-180` — the explanation that the seal key
is never printed, and the `kubectl … jsonpath` command that is the **only** remaining way to
retrieve it — to render outside the blockquote, visually detaching the recovery command from
its warning. The content is correct and is the difference between a recoverable and an
unrecoverable cluster; worth fixing purely on rendering grounds. **Recommendation:** close the
bold marker; re-render and visually confirm the blockquote holds the full checklist.

## M-24. OpenBao namespace/data rebuild is a documented mechanism, not an ordered procedure

**Impact: Medium** | **LoE to fix: Low**

`bootstrap-openbao.md:55,139` establish that `OPENBAO_SEAL_KEY` is restored on a namespace
rebuild and that losing it loses the data; `docs/workflows/llz-bootstrap-openbao.md:361-373`
carries the actual mechanism (idempotent seed, restore-from-`infra-<region>`) — but that file
never ships. Neither delivers an ordered "the `llz-openbao` namespace/PVCs are gone — do this"
procedure. **Recommendation:** extract an ordered procedure from the workflow doc into the
delivered runbook.

## M-25. Two `ClusterIssuer` stuck-pod remedies never ship anywhere

**Impact: Medium** | **LoE to fix: Low**

`docs/workflows/llz-bootstrap-openbao.md:341-343,356-359` gives precise, correct remedies for
OpenBao/otel pods stuck `ContainerCreating` (ensure the `openbao-ca`/`otel-bootstrap-ca`
`ClusterIssuer` is Ready, not hand-seeding a cert) — neither appears in any delivered file;
`bootstrap-openbao.md:138` mentions the two CAs only in a parenthetical with no failure mode
attached. **Recommendation:** move both remedies into `bootstrap-openbao.md` directly.

## M-26. `import-apl-site.md` excludes data migration by design, with no compensating recovery runbook for a lost cluster

**Impact: Medium** | **LoE to fix: Medium**

`docs/runbooks/import-apl-site.md:112-119` is admirably explicit that the import flow covers
configuration/inventory only and does not migrate PV/Object Storage/database data (points at
issue #114). `llz import plan` generates `rclone`/`pg_dump` recipes only for the *adoption*
path, never the *recovery* path — an operator who has lost a cluster has no equivalent.
**Recommendation:** tracked by H-2 (DR design); note here as the concrete missing counterpart.

## M-27. What actually survives a rebuild is documented only outside the delivered set

**Impact: Medium** | **LoE to fix: Low**

The durability story is real and reasonably designed: buckets survive destroy
(`llz-terraform.md:678-682`, destroy is module-scoped), databases/object storage persist
between e2e runs, in-cluster ArgoCD/apl-core objects are disposable, and the recreate landmine
(`apl-<env>` branch must be deleted first or bootstrap wedges on undecryptable SealedSecrets,
`apl-branch-recreate-wedge.md:66-75`) is delivered and correct — but `first-build-failed.md` §6,
the file that actually instructs a destroy-and-rebuild, does not link to it; its "See also"
lists quickstart, bootstrap-openbao, and apl-values-propagation instead.
**Recommendation:** add `apl-branch-recreate-wedge.md` to §6's "See also," and consider
promoting the "what survives" facts into the delivered set given C-1.

## M-28. `OPENBAO_SECRETS_WRITE_TOKEN` permissions and the zero-to-converged walkthrough are each written at full length in 2+ places

**Impact: Low-Medium** | **LoE to fix: Medium**

Beyond M-8 (already Medium on its own): `docs/runbooks/bootstrap-openbao.md:62-85`'s
`OPENBAO_SECRETS_WRITE_TOKEN` permissions box is restated at near-full length in
`docs/quickstart.md:758-773`, which then closes with "Canonical reference: [bootstrap-openbao
runbook]…" and restates it anyway — the exact pattern the docs SKILL's Rule 11 already
identified once in the workflow docs, recurring here. Currently consistent (latent-drift
finding, not a live contradiction), but both files are in the delivered set, so any future edit
to one and not the other ships to every adopter. **Recommendation:** trim the quickstart copy to
the canonical-reference link only.

## M-29. `openbao set` `--yes` and `pvc-*` fixes verified genuinely fixed — folded here to close the loop, not a new defect

**Impact: N/A — confirmation only**

Re-verified per the docs SKILL's instruction not to re-report already-fixed findings, but
recording explicitly since both were named candidates in this task: all five real `openbao set`
invocations in delivered docs carry `--yes` (`openbao-team-login.md:137-139`, `grafana-
access.md:82`, `harbor-accounts.md:138-143,172`, `openbao-accounts.md:32`); both directory
indexes carry the trap as a standing warning. The `pvc-*` blast-radius framing is fixed in
shape (both runbooks correctly teach the relabeled form, `orphan-volume-cleanup.md` correctly
states the prefix is no longer a scoping boundary) — but see H-6: fixed in shape, wrong in
detail, which is a new defect of the same root cause, not a regression of the old one.

## M-30. `docs/landing-zone-spec.md`'s "validated" language over-claims for two of the three inert spec blocks

**Impact: Medium** | **LoE to fix: Low**

Continuation of H-3's evidence, isolated as its own fix: `spec.alerting` genuinely is validated
(`validate.go:56,198`); `spec.dns` and `spec.defaults.platform` have no validator function at
all (`func validateDNS`/`func validatePlatform` do not exist) — both are merely accepted by the
strict-decode step. An operator reading "validated" for all three reasonably infers a typo'd
`acmeEmail` is checked; it is not. **Recommendation:** narrow the language to "accepted by the
schema but neither validated nor rendered" for `spec.dns.*`/`spec.defaults.platform.*`; keep
"validated but never rendered" for `spec.alerting.*` only.

## M-31. `imagePullSecret` per-team-namespace distribution — an ADR 0007 open gap — has adjacent machinery that could be mistaken for closure

**Impact: Medium** | **LoE to fix: Low** (docs-only; the design decision itself is a separate,
larger question)

**Evidence:** `docs/adr/0007-app-delivery-boundary.md:90-118` leaves open that nothing
distributes an `imagePullSecret` to a workload namespace, naming an ESO template per team
namespace as "the likely answer." Adjacent-but-different machinery exists —
`kubernetes-charts/llz-cert-automation/` renders a `dockerconfigjson` from `secret/harbor/robot`
via an ESO v2 `configTemplate`, and `ci_bootstrap_cluster_manifests.go:91-112` builds a
`ghcrPullSecretManifest` — but neither is the per-team-namespace distribution the ADR
describes. Verified: the gap is genuinely still open, so the ADR text is accurate as written.
Recorded here (not as a defect, as a "do not re-open this question" note) since it was one of
Track D's undocumented-decision candidates and is easy to mistake for resolved on a shallow
read of the adjacent code.

## M-32. Two more of Track D's nine undocumented-decision candidates are recorded but mis-filed, not absent

**Impact: Medium** | **LoE to fix: Low**

Track D's sub-task 3 swept for architectural decisions embedded in code/config with no ADR
backing (nine candidates, U1-U9). Beyond H-20 (genuinely undocumented) and M-31 (documented and
accurate), two more decisions exist in writing but in a location a reader would not think to
check: the repo's "one umbrella tag, not per-module tags" release-versioning decision is
explained in prose in `terraform-modules/RELEASING.md`'s own callout box rather than as a
numbered ADR (a *documented* decision, just not in ADR form — low-severity on its own, folded
here since the release skill/RELEASING.md are already flagged under H-16 for a related, more
severe defect), and the remaining candidates were judged conventions better suited to
`AGENTS.md` (org-identity/prefix rules, scars-as-defaults) than to a point-in-time ADR — noted
so a future audit does not re-raise them as gaps. **Recommendation:** no action required beyond
H-20; this entry exists to close out the sub-task's scope for the record.

## M-33. `secrets-before-apps.md` design was unreadable during this audit — permission-blocked, not verified

**Impact: N/A — coverage gap, not a finding**

Track D's design-status sweep could not read `docs/designs/secrets-before-apps.md`: both `Read`
and `Bash` were blocked on this path by a permission rule (the filename matches a secrets-guard
pattern). Its status line was captured only via an earlier directory-wide grep ("Partial. Phase
1 (bounded steady-state propagation) landed") and is consistent with the index row, but the
body was never opened and no claim inside it was checked — unlike every other design in this
report. **Recommendation:** the permission rule is almost certainly matching on the filename
substring rather than content; either carve out an exception for `docs/**` or reassign this one
file's review to a session with a different permission profile.

---

# Low

## L-1. `core.hooksPath` guidance reads as contradictory between the template repo and instance repo without naming each other's context

**Impact: Low** | **LoE to fix: Low**

`CONTRIBUTING.md:24-28`/`AGENTS.md:150-151` (template repo): enable hooks via
`core.hooksPath template-scripts/hooks`. `docs/playbooks/operator-onboarding.md:62-65`
(instance repo): "Do not set `core.hooksPath`." Both are correct for their own repo (checked-in
hooks dir vs. `llz hooks`'s `.git/hooks/pre-commit` install) but neither names the other's
context, so a contributor who also operates an instance reads a flat contradiction.
**Recommendation:** one clause in each pointing at why the other repo differs.

## L-2. Delivered cross-doc links resolve to upstream `main`, not the adopter's pinned version — a documented, deliberate tradeoff

**Impact: Low (informational)** | **LoE to fix: N/A**

`ci_deliver_docs.go:327-329` pins the link-rewrite target to `main` deliberately (a pinned link
would go stale in the other direction and trip the upgrade-churn guard). Consequence: an
operator following a cross-link out of a delivered runbook mid-incident needs internet access
and may land on a newer LLZ's docs than the one they're running — in tension with
`docs/runbooks/README.md:7-8`'s own framing that runbooks are "read by operators who do not
have this repo checked out, often mid-incident." Recorded as a tradeoff, not a bug.

## L-3. Post-bootstrap escrow content is duplicated within `quickstart.md` itself

**Impact: Low** | **LoE to fix: Low**

`docs/quickstart.md:134-142` (copy/paste path) and `:630-649` (§4 explained section) carry the
same escrow content, cross-linked to each other — a deliberate structural choice. Flagged only
because it is the fourth/fifth copy once H-4's skill copy and the bootstrap runbook are
counted, and H-4 proves at least one copy already lost an item.

## L-4. `docs/architecture/` and `docs/infosec/` have no directory index

**Impact: Low** | **LoE to fix: Low**

Both are small (2 and 1 files) and every file is linked directly from the root README's
Operations table, so nothing is lost today. Flagged because the reasoning that justifies an
index ("the list next to the files it describes") starts to apply once either grows —
`docs/architecture/` is the natural home for more. Do not add preemptively.

## L-5. `argocd-redis` auth-split has no day-2 pointer outside a converge run

**Impact: Low** | **LoE to fix: Low**

The WRONGPASS/NOAUTH split is reactively self-healed inside `llz ci converge`
(`llz-bootstrap-openbao.md:682-690`, which itself notes the permanent fix still belongs in
apl-core) — outside a converge run, an operator hitting it on day 2 has no pointer. Only
material gap found in an otherwise excellent `argocd-ops.md` (see Done Well).

## L-6. `docs/infosec/` checklist and `docs/secrets.md` are linked one-way, with unguarded duplicate rotation cadences

**Impact: Low** | **LoE to fix: Low**

Checklist → secrets.md: three links. secrets.md → infosec: zero. No duplication of content
(policy vs. mechanism, structurally disjoint), but the same cadences (90-day PAT, 120-day
bucket key) are stated in both with no guard binding them — currently consistent, a latent
desync risk. The checklist's 180-day CSPM figure has no in-repo counterpart to check against at
all. **Recommendation:** low priority; note in secrets.md's cross-references section that the
checklist restates the same cadences.

## L-7. The infosec checklist is the oldest-dated enterprise doc in the repo

**Impact: Low** | **LoE to fix: Low**

`docs/infosec/linode-account-request-checklist.md:4` — "Last Updated: 2026-05-05," predating
ADR 0009, 0010, and 0012 (every credential-observability and mTLS decision since). Nothing in
it is currently wrong (spot-checked the LDE claim and rotation cadences — both hold), but it is
the doc most likely to drift next and the one an enterprise reviewer opens first.

## L-8. `docs-guard`'s summary line reads as broader coverage than it has

**Impact: Low (documentation-of-the-tool, not a docs-content defect)** | **LoE to fix: Low**

Not a new defect beyond H-1 — recorded separately because it affects how this whole report
should be read going forward: the guard's own `docs-guard: N Markdown file(s) OK` output line
does not state what it does *not* check (command-path existence, resource/attribution claims,
component enumerations, design-status accuracy). A one-line addition to the guard's own output
or `docs/e2e-gates.md` scoping what "OK" means would prevent future audits from over-trusting a
green run, exactly as H-1 shows happened here across two prior audits.

---

# What's done well

- **`docs/playbooks/argocd-ops.md`** is the strongest single file in the delivered set. It
  correctly distinguishes `OutOfSync` causes including the client-side-diff trap, documents the
  Lua health-check deadlock (a failure whose defining property is that merging the fix does
  nothing) with a convergent patch, and gives the right negative instruction ("do not 'fix'
  this with `ignoreDifferences` on an image path"). Only gap found: L-5.
- **`docs/runbooks/apl-branch-recreate-wedge.md`** is the reference shape every other wedge
  runbook should follow: symptom, a four-signal recognition set where each signal is
  individually ambiguous, a proven remediation with observed timings, prevention, and an honest
  open-follow-up note.
- **Alert-to-runbook linkage is genuinely strong at the mechanical level**: all 43 platform
  alerts across four `PrometheusRule` files carry a `runbook_url` annotation — 100% coverage,
  better than most production platforms. The gap (H-7, M-17) is in the response content two of
  those links land on, and the inventory doc's completeness claim — not the linking discipline
  itself.
- **Credential-lifecycle documentation is enterprise-grade**: `docs/secrets.md` (767 lines),
  two ADRs that name their own residue honestly (0009's "What this does not cover," 0012's
  "What this does NOT do"), four rotation runbooks, and the best partial-failure coverage in the
  repo for the cases it does handle (H-10's Recommendation notwithstanding).
- **`kubernetes-charts/README.md`** documents its own known-stale references
  (`components.go`'s dangling `ArgoApps` paths) inline, with why they're unreachable rather than
  broken — the right instinct, applied consistently.
- **The runbooks/playbooks split-index structure (Rule 8) is fully intact and correctly
  applied**: all 12 runbooks and 7 playbooks are indexed by symptom/task respectively, zero true
  orphans in `docs/` under the task's own definition, and the root README correctly does not
  enumerate directory contents.
- **`docs/adr/README.md` and `docs/designs/README.md`'s numbering/status discipline holds up
  well under adversarial checking**: the two `0007`s are deliberately documented, "next free
  number" guidance is internally consistent, every ADR supersession found is explicitly
  labeled in both directions (0005↔0006 cross-reference each other's correction), and 11 of 11
  `Partial` designs correctly name their phases per the README's own rule — the defects found
  (H-18, H-19, M-9 through M-16) are individually real but sit inside a structure that is
  fundamentally sound and was clearly built to prevent exactly this class of drift.
- **`docs/lessons-learned.md` and `docs/e2e-gates.md` are both excellent** — the problem
  identified in this audit (M-2, H-8, H-11) is reachability and delivery-set membership, never
  quality.
- **The docs-guard mechanical baseline is real and valuable**: 124 files, 862 `llz` invocations,
  248 flags, 17 workflow dispatches, 603 links, 144 TOC entries, all green, and it correctly
  catches malformed flags on resolved commands. Its one structural blind spot (H-1) does not
  diminish what it does catch.
