# LKE Landing Zone — Enterprise Reliability / Scalability / DR Review

**Scope:** convergence machinery, failure modes & SPOFs, disaster recovery, scalability, upgrade-path reliability, state management.
**Method:** read-only. Every claim below cites `file:line` from the tree at commit `41cd28a`. Documented, accepted tradeoffs are labeled as such and are not counted as defects.

---

## Executive summary

The convergence contract (`docs/architecture/convergence-contract.md`) is a genuinely strong piece of reliability engineering, and the `internal/health` classifier implements it with unusual discipline — fail-closed on vacuity, `probeUnknown` distinguished from "absent", per-scar predicates (`IsGitAuthError`, `IsAnnotationLimitError`) carrying the incident that produced them. This is well above the bar for a landing-zone template.

Three findings undercut it. First, `llz ci converge`'s hard-fail re-check branch is the one path through the loop that never evaluates the deadline, so a component flapping between hard-fail and in-progress polls **without bound** until GitHub's job axe lands — the exact "no verdict, no diagnostics" outcome the repo wrote a scar comment about elsewhere. Second, `kubectlReachable()` probes an endpoint Kubernetes serves anonymously, so a **revoked or expired kubeconfig credential is indistinguishable from "still converging"** and burns the entire budget before reporting the wrong cause — the same failure class the repo already fixed once for Argo→git but never applied to llz→apiserver. Third, the convergence contract's own section 1 describes `waitAplPipeline` as the load-bearing bootstrap readiness gate; that verb has **zero callers** anywhere in the repo, and the gate that replaced it is materially weaker on the two axes its predecessor's comments explicitly justified.

On DR the posture is honest but thin at enterprise grade. There is **no backup of OpenBao's data at all** — and the natural offline path (VolumeSnapshot the raft PVC) is blocked by a two-layer admission deny built for an unrelated tagging reason and protected by its own green CI gate, so the gap costs more to close than "nobody built it yet" suggests. The cross-region HA pair is two independent OpenBao clusters reconciled by operator-side dual-write with **no automated reconciliation and no drift check**, so the standby's promotion-readiness is unknown at all times. No RTO or RPO is stated anywhere in the repository. State locking is absent-by-platform, and two of the five CI concurrency groups meant to substitute for it are pointed at the wrong key — including one job that declares exclusivity it does not have while re-encrypting every state file.

**Counts: 5 Critical · 15 High · 19 Medium · 7 Low.** The highest-leverage first action is not any single fix but a decision record: this repo has a well-exercised convention for writing down a gap it chooses to live with (ADR 0009's "honest residue", ADR 0012's what-this-does-not-do list) and has never applied it to disaster recovery. A DR ADR stating RTO/RPO per data class would settle roughly a third of the findings below by turning them from omissions into decisions.

**Method note:** areas 1-2 were read directly; areas 3-6 were reviewed by four parallel subagents whose full findings, with additional detail and citations, are in `ai-artifacts/review/parts/{dr,state,scale,upgrade}.md`. Two of those reviews were blocked by a local permission rule from opening `docs/secrets.md` directly and worked around it with targeted extraction; both disclosed the limitation rather than papering over it, and claims sourced that way are quoted verbatim.

---

## Findings index, in severity order

Findings are grouped by domain below (convergence, state, upgrade/drift, DR) because the evidence for each cluster together. This index is the strict severity ordering.

| # | Finding | LoE |
|---|---|---|
| **C-1** | `llz ci converge` can poll without bound — the hard-fail re-check branch never checks the deadline | Low |
| **C-2** | A revoked/expired kubeconfig credential reads as "still converging" and burns the full budget | Low-Med |
| **C-3** | A PR plan job runs `tofu import` — a state write — in a concurrency group disjoint from the apply group | Low |
| **C-4** | `rotate-state-passphrase` declares exclusivity it does not have while re-encrypting every state file | Low |
| **C-5** | No backup of OpenBao's data at all, and the offline path is foreclosed by an admission gate | Medium |
| **H-1** | The convergence contract documents a bootstrap readiness gate that no longer runs | Low |
| **H-2** | The replacement bootstrap gate is weaker on the two axes its predecessor justified | Medium |
| **H-3** | `converge` prints its long-pole diagnostic on success and omits it on every failure | Low |
| **H-4** | OpenBao's static seal key is an unreplicated, unrotatable single point of total data loss | Medium |
| **H-5** | The cross-region HA pair has an unbounded, unmeasured RPO | Medium |
| **H-6** | The `vpc` root's documented state key differs from CI's, and the test pins the wrong one | Low |
| **H-7** | No state bucket versioning, backup, or recovery runbook — while two docs say to restore a snapshot | Low-Med |
| **H-8** | Merging an `llz upgrade` commit is a production platform deploy, ungated | Low |
| **H-9** | Nothing compares repo desired-state to the live cluster; no scheduled Argo-Synced check | Low |
| **H-10** | No destructive-plan gate, and the documented approval control is never verified to exist | Medium |
| **H-11** | The upgrade path is never exercised end-to-end; release-e2e is greenfield-only | Medium |
| **H-12** | The Linode API client has no rate-limit handling; one caller discards the status code | Low |
| **H-13** | The state-passphrase rotation job inits against a directory that does not exist | Low |
| **H-14** | Nothing verifies the seal-key DR copy still exists — checked once per instance lifetime | Low |
| **H-15** | Three required-anti-affinity OpenBao replicas on a three-node pool: zero headroom | Low |
| **M-1** | No stated RTO or RPO anywhere | Low |
| **M-2** | `instance-custom` tolerance decided by name prefix, and fails open | Medium |
| **M-3** | The phase1 hard-fail downgrade has one veto and cannot report why it gave up | Medium |
| **M-5** | State encryption is phase 1 — `enforced` not set *(documented tradeoff)* | Low |
| **M-6** | The encryption key *name* is a mutable repo variable; losing it is as fatal as the passphrase | Low |
| **M-7** | The apl-core migration runbook contradicts shipped behaviour | Low |
| **M-8** | No rollback path for an upgrade, in code or docs | Low |
| **M-9** | Identical cron literals in every instance — no jitter or stagger | Low |
| **M-10** | The account-quota preflight ships disarmed *(quota itself documented)* | Low |
| **M-11** | Two rotation paths select their deployment by hand-typed string and array index | Low |
| **M-12** | Platform sizing is fixed; the reconciler is a singleton on the day-2 signal path | Medium |
| **M-13** | Shared VPC — the lever that relieves the hardest ceiling — has no live proof | Medium |
| **M-14** | Three daily jobs per env race the same LKE-E ACL *(documented, unlanded)* | Low |
| **M-15** | A runbook documents a restore path that has no implementation | Low |
| **M-16** | Managed Postgres has no declared backup posture and nothing asserts one | Low |
| **M-17** | OpenBao audit log on `emptyDir`; the predictive alert its comment prescribes does not exist | Low |
| **M-18** | The break-glass path is well built and exercised by no live gate | Medium |
| **M-19** | The cluster-rebuild path is documented for infra and silent on what does not survive | Low |
| **M-20** | OpenBao's `updateStrategyType: OnDelete` is inherited, correct, and undocumented | Low |
| **L-1** | Cross-instance state collision prevented by convention, not construction | Low |
| **L-2** | No `prevent_destroy` anywhere; no bucket versioning | Low |
| **L-3** | Two stale scar comments misdirect operators to unprotected hand-apply paths | Low |
| **L-4** | Autoscaler wired and exposed but off by default, and the default is not stated | Low |
| **L-5** | One node pool per cluster, with hardcoded labels | Medium |
| **L-6** | No documented tested limit for nodes, environments, clusters, or apps | Low |
| **L-7** | `llz upgrade`: not transactional, near-vacuous `--dry-run`, `git add -A`, silent gate skips | Low |

*(The number M-4 is unused: an early concurrency/locking finding was superseded during review by the more precise C-3, C-4 and H-6. Remaining IDs were left unrenumbered so they remain stable across drafts.)*

---

# Findings

## C-1. `llz ci converge` can poll without bound — the hard-fail re-check branch never checks the deadline

**Impact: Critical** | **LoE to fix: Low**

**Evidence**
- `tools/cmd/llz/ci_health.go:210` — the loop is `for attempt := 1; ; attempt++` with no loop-level deadline test.
- `tools/cmd/llz/ci_health.go:250-256` — `case health.ConvergePoll:` evaluates `time.Now().After(deadline)`.
- `tools/cmd/llz/ci_health.go:278-289` — `case health.ConvergeUnreachable:` evaluates it.
- `tools/cmd/llz/ci_health.go:257-277` — `case health.ConvergeRetryHard:` **does not**. When the re-check at `:268` returns exit 2 or 3, neither inner `case` matches, control falls out of the inner switch at the `:277` comment (`// recovered to in-progress (or the apiserver blipped) — keep polling`) and re-enters the outer loop with the deadline never consulted.

**Why it matters**
A cluster whose main poll reports exit 1 and whose 60s-later re-check reports exit 2 never reaches a budget check. This is not a contrived state: a pod in `CrashLoopBackOff` classifies `CatFail`, and the same pod mid-restart classifies `CatPending` — `tools/internal/health/argo.go:155-157` returns `CatPending` for `Progressing` precisely so a rollout is polled rather than failed. A flapping component therefore alternates 1↔2 indefinitely. Each lap costs `--retry-delay` (60s default, `ci_health.go:109`) plus two full health scans, which the repo measures at 35-58s each (`ci_health.go:262-267`) — roughly 2.5 minutes per lap, forever.

The blast radius is the whole convergence gate. `instance-template/.github/workflows/llz-bootstrap-openbao.yml:728` runs `llz ci converge --budget 1200` inside a job whose `timeout-minutes` is 90 on the bootstrap path (`llz-bootstrap-openbao.yml:168`). A 20-minute budget silently becomes a 90-minute job kill with **no verdict and no diagnostics** — precisely the outcome `tools/cmd/llz/ci_wait_apl_pipeline.go:73-88` was rewritten to prevent ("no stage could ever report its own timeout on a genuinely slow run; GitHub's job axe always landed first, with no verdict and no diagnostics"). It also breaks the contract doc's own promise at `docs/architecture/convergence-contract.md:116` ("After `$BUDGET` seconds of total elapsed time with no exit `0`, gives up with exit `1`"). For the in-cluster sibling driven by an Argo `WorkflowTemplate` (`tools/cmd/llz/ci_health_incluster.go`), there may be no job axe at all.

The gap is invisible to the test suite: `tools/cmd/llz/ci_health_test.go:708-720` scripts exactly `[1, 2, 2]`, so the third scan lands in the `ConvergePoll` branch and does hit the deadline. Nothing drives the alternating `[1, 2, 1, 2, …]` sequence.

**Recommendation**
Hoist the deadline test to the top of the loop body, immediately after `attempt` increments, so every path is bounded regardless of which branch it took — the budget then means what the doc says it means. Keep the per-branch messages for their specific diagnostics. Add a regression test that scripts `[1, 2, 1, 2]` with a budget that expires, asserting the run terminates with the budget error rather than exhausting the script.

---

## C-2. A revoked or expired kubeconfig credential reads as "still converging" and burns the full budget

**Impact: Critical** | **LoE to fix: Low-Medium**

**Evidence**
- `tools/cmd/llz/ci_health.go:499-502` — `kubectlReachable()` is `execOutput("kubectl", "version", "--request-timeout=10s")`, returning `err == nil`.
- `tools/cmd/llz/ci_health.go:345-351` — a false result is the *only* producer of exit 3 at the top of the scan.
- `tools/cmd/llz/kubectl_probe.go:18-22` — the probe doctrine explicitly names the case: "An unreachable API server, **an expired token**, a throttled request or a 10s timeout all read as ABSENT."
- `tools/cmd/llz/kubectl_probe.go:55-58` — `probeUnknown` covers "unreachable, **unauthorized**, timed out, throttled".
- `tools/cmd/llz/ci_health.go:545-552` — after retries, an unanswerable list records `health.CatPending` ("treating as inconclusive rather than 'none found'") → exit 2.

**Why it matters**
The repo-sourced half is unassailable and carries the finding on its own: **no path in the health tree turns a refused credential into a terminal verdict.** `kubectl_probe.go:18-22` names "an expired token" as a `probeUnknown`; `probeUnknown` survives its retries and becomes `CatPending` via `ci_health.go:545-552`; `CatPending` is exit 2; exit 2 means poll. So `converge` runs to budget exhaustion and reports **"budget of 1200s exhausted with the cluster still in-progress"** — for a cluster that is entirely healthy and a credential that is dead. The operator is pointed at the cluster; the fault is in the kubeconfig.

Whether the top-level gate catches it *earlier* depends on one fact to confirm against LKE-Enterprise: stock Kubernetes serves `/version` to unauthenticated callers, because the default `system:public-info-viewer` ClusterRole is bound to both `system:authenticated` and `system:unauthenticated`. If LKE-E keeps that default, an expired or revoked token is treated as anonymous, `/version` returns 200, and `kubectlReachable()` reports the apiserver reachable — so the misdiagnosis is total. If LKE-E has tightened it, `kubectlReachable()` correctly returns exit 3, and `converge` then retries the unreachable branch against the budget without ever naming the credential (`ci_health.go:284-289`). **Both branches end in budget exhaustion with the wrong cause reported**; they differ only in which message the operator gets. The recommendation below fixes both.

This is not hypothetical for this platform. The per-cluster `lke-admin` token embedded in the LKE kubeconfig is on a rotation schedule with its own runbook (`docs/runbooks/lke-admin-rotation.md`), and `docs/alerting.md:222` tracks it going overdue. A rotation that lands mid-flight, or a runbook step executed out of order, produces exactly this signature.

The repo has already paid for this lesson in its sibling form and fixed it there: `tools/internal/health/argo.go:174-195` documents gsap-apl run 29709276389 burning "its entire 1200s convergence budget" on a git credential the remote had already refused, and `IsGitAuthError` + `Report.GitAuthFailure` (`tools/internal/health/contract.go:83-93`) exist solely to make that terminal and veto the phase1 downgrade. The identical reasoning — "the remote answered, the answer was no, and it will keep being no until an operator fixes the credential" — applies verbatim to llz's own apiserver credential and has not been applied.

**Recommendation**
Make `kubectlReachable()` probe an **authenticated** endpoint (`kubectl auth can-i list namespaces`, or any namespaced GET) and classify its failure by kind: connection-refused / TLS / timeout → exit 3 as today; **401/403 → terminal**, exit 1, with a message naming the kubeconfig and the `lke-admin` rotation runbook. Add a `CredentialRefused` flag to `health.Report` that vetoes the phase1 downgrade exactly as `GitAuthFailure` does at `ci_health.go:432`. Extend `classifyKubectlErr` in `kubectl_probe.go` to return a distinct `probeUnauthorized` verdict so the distinction is available to every check, not only the top-level gate.

---

## H-1. The convergence contract documents a bootstrap readiness gate that no longer runs

**Impact: High** | **LoE to fix: Low** (docs) / **Medium** (if the gate should be restored)

**Evidence**
- `docs/architecture/convergence-contract.md:90-98` — section "1. The bootstrap command (`llz ci bootstrap-cluster`)" states `waitAplPipeline` is the "one loud readiness gate", and that "The command applies the bootstrap Argo Application only **after** that gate returns".
- `docs/architecture/convergence-contract.md:137` — points readers to `ci_bootstrap_cluster.go` for "the `waitAplPipeline` gate raced against the two Kyverno policies".
- `tools/cmd/llz/ci_bootstrap_cluster.go:1-22` — the command's own header says the opposite: on a managed cluster "Linode installs+manages apl-core… **LLZ never self-installs apl-core** (see ADR 0005)".
- `tools/cmd/llz/ci_bootstrap_cluster.go:258-400` — the actual flow. It calls `waitManagedArgoReady` at `:320`. It never calls `waitAplPipeline`.
- `tools/cmd/llz/ci.go:372` — `ciWaitAplPipelineCmd()` is still registered as a verb.
- Verified: `wait-apl-pipeline` / `waitAplPipeline` has **zero** callers in `Makefile`, `template-scripts/`, `.github/`, `instance-template/`, or `terraform-modules/`.
- `docs/workflows/llz-bootstrap-openbao.md:176-177` repeats the stale claim: "the in-cluster bootstrap phase (apl-core install + the `wait-apl-pipeline` gate) runs at the head of this job too: 70m."
- `docs/adr/0002-thin-terraform-native-bootstrap.md:50` also describes the gate as live.

**Why it matters**
This document is load-bearing by the repo's own construction — `AGENTS.md` and the contract doc itself position it as the shared model every contributor and agent reasons from, and `docs/architecture/convergence-contract.md:126` instructs contributors to reuse `waitAplPipeline` rather than write new polling. Anyone following that instruction today would wire a gate into a bootstrap that does not call it. The stale `70m` budget arithmetic in `llz-bootstrap-openbao.md:176-190` is likewise reasoning about ~55 minutes of stage ceilings that are never consumed, which distorts every subsequent timeout decision made from that page.

**Recommendation**
Rewrite section 1 of the convergence contract to describe `waitManagedArgoReady` (`ci_bootstrap_cluster.go:402-428`) as the actual gate, and state plainly that ADR 0005's managed-apl-core model moved apl-core install ownership to Linode. Correct `docs/workflows/llz-bootstrap-openbao.md:176-190` and `docs/adr/0002:50`. Then decide `wait-apl-pipeline`'s fate explicitly: either retire the verb, or restore the coverage it uniquely provided (see H-2) — a registered command with no caller and three docs describing it as load-bearing is the worst of both. A `make lint` guard asserting every registered `llz ci` verb has at least one caller (or an explicit allowlist entry) would make this class statically decidable.

---

## H-2. The replacement bootstrap gate is materially weaker than the one it replaced, on the two axes its predecessor's comments justified

**Impact: High** | **LoE to fix: Medium**

**Evidence**
- `tools/cmd/llz/ci_bootstrap_cluster.go:406-428` — `waitManagedArgoReady` gates on exactly two facts: the `applications.argoproj.io` CRD exists (`:410`), and `deploy/argocd-server` has non-zero `availableReplicas` (`:411-413`).
- `tools/cmd/llz/ci_wait_apl_pipeline.go:89-98` — the retired gate's six stages: Argo CD CRD Established; **argocd-application-controller** StatefulSet `readyReplicas=1`; Kyverno CRD; **Kyverno admission-controller Available**; cert-manager CRD; **cert-manager webhook Available**.
- `tools/cmd/llz/ci_wait_apl_pipeline.go:14-19` — why the controller, not the CRD: "The CRD is Established ~60-90s before the controller actually serves, so **gating on the CRD alone is too weak**."
- `tools/cmd/llz/ci_wait_apl_pipeline.go:23-25` — why the cert-manager webhook: "Argo's first sync applies Certificate CRs (openbao-tls, harbor-tls, …); before the validating webhook is up they **503 with 'failed calling webhook'**."

**Why it matters**
Two distinct regressions in coverage:

1. **`argocd-server` is not the reconciler.** `argocd-server` is the API/UI deployment; `argocd-application-controller` is the StatefulSet that actually reconciles Applications. The new gate proves the apiserver will *accept* the Application object, not that anything will *act* on it. The retired gate deliberately waited on the controller, and its comment states the weaker property is insufficient.
2. **Kyverno and cert-manager webhook readiness are no longer gated at all.** The bootstrap now server-side-applies the bridge Applications (`ci_bootstrap_cluster.go:389-397`) with no assurance that the cert-manager validating webhook is serving — the exact precondition whose absence produces the documented `failed calling webhook` 503 on the first Certificate sync.

This is mitigated rather than fatal: Argo's `retry: backoff` re-syncs, and `llz ci converge` polls, so the steady state is usually reached. But the mitigation converts a fast, precisely-attributed gate failure into slow, diffuse convergence churn charged against the converge budget — and given C-1, churn is the input to the unbounded-loop path.

**Recommendation**
Extend `waitManagedArgoReady` to wait on `statefulset/argocd-application-controller` `readyReplicas>=1` rather than (or in addition to) `argocd-server`, reusing the jsonpath `--for` technique already proven at `ci_wait_apl_pipeline.go:92`. Add existence-then-Available waits for the cert-manager webhook before the bridge apply — on a managed cluster these are Linode-installed, so the wait is a readiness observation, not an install dependency, and cannot deadlock the way the deliberately-omitted ESO stage would (`ci_wait_apl_pipeline.go:34-39`). Budget them from the measured managed-cluster timings, not from the retired gate's ceilings.

---

## H-3. `converge` prints its long-pole diagnostic on success and omits it on every failure

**Impact: High** | **LoE to fix: Low**

**Evidence**
- `tools/cmd/llz/ci_health.go:166-181` — `reportConvergeLongPole` emits both a `::notice::` and a `GITHUB_STEP_SUMMARY` section naming what was still not-OK.
- Called at `ci_health.go:248` (converged) and `ci_health.go:274` (re-check converged). **Both are success paths.**
- The three failure returns print a bare one-line message and nothing else: `:253-254` (budget exhausted, in-progress), `:271-272` (hard-failed twice), `:285-286` (budget exhausted, unreachable).
- `prevNonOK` is in scope and populated at `:251`; `res.nonOK` is populated on every scan (`ci_health.go:441-450`).
- `docs/architecture/convergence-contract.md:116` promises the opposite: "gives up with exit `1` and **dumps a final diagnostic**."

**Why it matters**
The diagnostic is emitted exactly when it is least needed and withheld exactly when it is most needed. An operator whose bootstrap fails at 20 minutes gets `budget of 1200s exhausted with the cluster still in-progress` and no indication of *which* components were pending — despite the process having computed that list on every one of ~20 polls. Recovering it means re-running `llz ci health` by hand against a cluster whose state has since moved, and in CI the runner and its cluster access are already gone. This materially lengthens every convergence incident, and it is the reason C-1's silent job-kill is so costly.

**Recommendation**
Call `reportConvergeLongPole` (or a failure-mode variant that prints `res.nonOK` from the *final* scan rather than the previous one) on all three failure returns before returning the error. Roughly three lines. Include the `Failed` bucket separately from `Pending` in the failure variant — on a hard-fail abort those are the actionable items.

---

## H-4. OpenBao's static seal key is an unreplicated, unrotatable single point of total data loss

**Impact: High** | **LoE to fix: Medium** | *(the KMS decision itself is a documented, accepted tradeoff)*

**Evidence**
- `docs/secrets.md:125` — the 32-byte static auto-unseal key is seeded as the `openbao-unseal-key` Secret and persisted as `OPENBAO_SEAL_KEY` in the `infra-<deployment>` GitHub environment "for disaster recovery and **must be copied offline — losing it loses the data**".
- `docs/runbooks/bootstrap-openbao.md:55` — same warning, same emphasis.
- `docs/secrets.md:734` — the rationale is documented and sound: no managed KMS exists on Linode, and a human quorum on every pod restart is untenable.
- `tools/internal/health/infra.go:88-89` — a sealed pod classifies `CatFail` with the message "pods auto-unseal from the static seal key at boot; check the openbao-unseal-key Secret and Raft storage".
- `docs/secrets.md:338` — the recovery keys are `OPENBAO_RECOVERY_KEY_1/2/3`, explicitly "**Static by design**", and per `docs/secrets.md:125` they "**cannot** decrypt the root key" — they authorize `generate-root`/`rekey` only.

**Why it matters**
The choice of a static seal key over KMS is a documented, well-reasoned tradeoff and is labeled as such. What is *not* adequately handled at enterprise grade is everything downstream of it:

- The key's only in-band copy is a GitHub Actions **environment secret**. Loss of the GitHub org, an environment misconfiguration, or a repo deletion is therefore correlated with loss of all OpenBao data — the DR artifact and the system that needs it share a failure domain. The "copy it offline" instruction is prose addressed to a human, with no gate confirming it happened.
- The recovery quorum does **not** substitute: it cannot decrypt the root key (`docs/secrets.md:125`). So there is no quorum path back from a lost seal key — the 3-of-5 escrow protects root-token regeneration only.
- **The 3-of-5 recovery quorum is defeated by co-location.** `bao operator init` runs with `-recovery-shares=5 -recovery-threshold=3`, and `docs/secrets.md:125` then stores **recovery keys 1-3 as `OPENBAO_RECOVERY_KEY_1/2/3` in the same `infra-<deployment>` environment**. A threshold of 3 implies split custody across three parties; storing exactly the threshold number of shares in one store means anyone with access to that environment holds a **complete quorum**, and losing that one environment loses the quorum outright. The cryptographic control is present and the operational property it exists to provide is not.
- The key is static "by design" with no rotation story, so its exposure window is the lifetime of the deployment.

**Recommendation**
Do not revisit the KMS decision — it is correctly reasoned. Instead close the custody gap: (a) add a bootstrap-time gate that requires an explicit operator acknowledgement (a checked input, recorded in the run summary) that `OPENBAO_SEAL_KEY` has been escrowed outside GitHub, and fail bootstrap without it — per the repo's own "write the task that makes the warning unnecessary" doctrine; (b) add a scheduled check that asserts the key is present and readable in at least one non-GitHub escrow, alerting like the other credential-health gauges in `docs/alerting.md`; (c) document the seal-key-loss scenario explicitly in `docs/runbooks/bootstrap-openbao.md` as *unrecoverable*, with the resulting rebuild-from-scratch procedure, so the RTO is a known number rather than a discovery made during an incident.

---

## C-5. There is no backup of OpenBao's data at all — and the obvious offline path is foreclosed by an admission gate built for an unrelated reason

**Impact: Critical** | **LoE to fix: Medium**

**Evidence — the absence, grepped not assumed.** A search for `raft.snapshot|operator raft snapshot|snapshot save|snapshot restore|bao operator raft` across `docs/ tools/ kubernetes-charts/ instance-template/ terraform-modules/ platform-apl/ Makefile` returns exactly two hits, neither a snapshot mechanism: `tools/cmd/llz/manifests/kyverno-pvc-deny-untaggable-clone.yaml:8` (a policy that *denies* snapshot-sourced PVCs) and `kubernetes-charts/llz-openbao-platform/values.yaml:387` (prose about `retry_join`). A broader `grep -rni 'snapshot'` over the same tree finds only Terraform *state* snapshots, apl-values git branch snapshots, Argo health snapshots, an upgrade file-mode snapshot, and a Linode Volume-**ID** capture before teardown (`docs/workflows/llz-terraform.md:531-533`) — which, read in context (`:553-575`), records identifiers so the post-destroy sweep can **delete** them. It is the opposite of a backup.

There is also no destination for a snapshot: `terraform-modules/llz-object-storage/main.tf:51-77` provisions exactly four buckets (harbor-registry and three Loki), and the one bucket that ever existed for backup purposes, `gitea_backup`, was **removed** (`main.tf:79-87`). And no gate: `tools/cmd/llz/ci.go:43-386` registers 27 `assert-*` verbs; none concerns snapshots, backup, or restore.

**Evidence — the offline path is actively blocked.** The platform rejects clone/snapshot-sourced PVCs at two layers: a Kyverno `ClusterPolicy` with `validationFailureAction: Enforce` (`tools/cmd/llz/manifests/kyverno-pvc-deny-untaggable-clone.yaml:39-84`) and its in-apiserver twin, a `ValidatingAdmissionPolicy` with `failurePolicy: Fail` applied by `llz ci bootstrap-cluster` so it enforces before Kyverno exists (`manifests/vap-pvc-deny-untaggable-clone.yaml:32, 40-46`; applied at `ci_bootstrap_cluster.go:294-295`). It is *gated* — `llz ci assert-admission-enforcement` proves the deny is live (`ci_assert_admission_enforcement.go:65, 127`). The rationale has nothing to do with DR: the Linode CSI clone path cannot carry the `lke<id>` ownership tag, so a clone "dangles as an orphan after teardown" (`kyverno-pvc-deny-untaggable-clone.yaml:4-12`). Its load-bearing premise — "The platform uses NO clone/snapshot PVCs … so denying them has zero functional impact today" (`:19-22`) — is true **only because no DR mechanism was ever built**.

**Why it matters**
OpenBao holds the platform's entire credential set — `secret/linode/api-token`, `secret/harbor/robot`, `secret/loki/object-store`, `secret/harbor/registry-s3`, `secret/cert-automation/github-token`, `secret/infra/github-dispatch-token`, and every `spec.teams` subtree (`docs/runbooks/bootstrap-openbao.md:146-161`, `docs/landing-zone-spec.md:344-367`). The only thing standing between that and total loss is three `ReadWriteOnce` Linode Block Storage Volumes in one region, protected by `reclaimPolicy: Retain` (`tools/cmd/llz/manifests/block-storage-class.yaml:135`). A regional Block Storage incident, a `terraform destroy` against the wrong `--field region`, or the destroy-time Volume sweep firing against the wrong cluster (`docs/workflows/llz-terraform.md:553-575`) destroys every credential the platform holds, with no recovery other than re-bootstrapping and re-minting each one by hand.

The second half is what makes this Critical rather than merely High: the next engineer who reaches for the natural fix — VolumeSnapshot the data PVC, restore into a rebuilt cluster — hits a hard deny at two layers, one of which cannot be disabled without an apiserver-level change, plus a green CI gate asserting the deny still works. That is the difference between "nobody built it" and "a gate blocks it."

**Recommendation**
Do **not** relax the denies. A Volume-level snapshot of a live raft store is crash-consistent at best, whereas `bao operator raft snapshot save` is application-consistent by construction — so the blocked path is also the inferior one. Instead: (1) add a CronJob in `llz-openbao` (or a lane of `llz-reconciler`, which already holds an OpenBao Kubernetes-auth token, `values.yaml:135-141`) running `bao operator raft snapshot save` into a new snapshots bucket added to `terraform-modules/llz-object-storage/main.tf`. The snapshot is **seal-encrypted**, readable only with the static seal key, so object storage does not widen the blast radius — but it does make the seal key strictly more load-bearing, so ship it together with H-4's escrow gate. (2) Name the gate: `llz ci assert-raft-snapshot` — list the bucket, assert an object newer than N hours, assert it parses as a raft snapshot archive — wired into the e2e assert battery alongside `assert-obj-roundtrip`. Per `docs/e2e-gates.md:21-39`, a CronJob that exists and a snapshot that lands are different facts. (3) Add a sentence to the clone-deny policy header naming OpenBao raft DR as the use case that is explicitly *not* a reason to relax it, pointing at the snapshot path — otherwise the next person spends a day rediscovering the coupling from an admission error.

---

## H-5. The cross-region HA "pair" has an unbounded, unmeasured RPO — the standby's promotion-readiness is unknown at all times

**Impact: High** | **LoE to fix: Medium**

**Evidence**
- The HA pair is `spec.cluster.ha.{role,group}`, validated to exactly one `active` and one `standby` per group (`docs/landing-zone-spec.md:168-190`). But the two clusters run **completely independent** OpenBao instances: "the secondary region runs its own independent OpenBao cluster. Cross-region consistency is achieved by operator-side dual-write … not by OpenBao replication. OpenBao OSS has no Performance Replication equivalent" (`kubernetes-charts/llz-openbao-platform/values.yaml:212-217`). The choice is deliberate and recorded (`docs/secrets.md:103-109` rejects a stretched raft cluster) — **that decision is sound and is not what this finding contests.**
- `llz openbao set` writes both, compares SHA-256 of the post-write payload, and rolls back the primary if the secondary fails (`docs/secrets.md:485-488`). That is a **per-write, in-band** check.
- **Anything that writes to only one cluster silently diverges**, and that set is not small: the in-cluster `harbor-robot-provisioner`, `linodeCredRotator` / `llz ci rotate-linode-creds`, `broad-pat-rotator`, and ESO PushSecrets all write into their own cluster's OpenBao (`docs/runbooks/bootstrap-openbao.md:147, 153-154`; `values.yaml:134-160`). The standby's Harbor credentials are replicated only at **bootstrap time**, by a one-shot workflow step (`docs/runbooks/bootstrap-openbao.md:147, 235-239`).
- `docs/secrets.md:601` instructs the operator, after a primary outage, to "run a drift check and, if needed, re-apply the last-written values." **There is no drift-check command.** Searching `docs/` and `tools/` for `drift check|drift-check|openbao diff|bao-diff|compare.*regions` finds only that prose, the unrelated template/chart-pin drift checks (`tools/cmd/llz/drift.go`, `ci_chart_pin_guard.go:153`), and one **manual, single-key** two-line `diff <(llz openbao get active …) <(llz openbao get standby …)` recipe buried in a migration runbook (`docs/apl-core-migration-runbook.md:188-193`).
- `docs/secrets.md:603-605` states outright: "there is no automated reconciliation."

**Why it matters** — the effective cross-region RPO is "whenever an operator last ran `llz openbao set` for that particular path" — unbounded, unmeasured, and per-path. The standby exists specifically to be promoted during a regional loss, and its readiness for that role is unknown at all times. A standby promoted after a primary loss is missing every credential written in-cluster since its bootstrap: rotated object-storage keys, rotated Harbor robots, rotated Linode PATs. From the cluster, that looks exactly like a working cluster with an authentication problem.

**Recommendation** — add `llz ci assert-openbao-parity`: enumerate the KV v2 paths in `credPaths` (the list `assert-rotation-health` already derives from, `tools/cmd/llz/ci.go:297`), read each from both clusters via the existing `llz openbao get active|standby` primitive, and compare **metadata version and `rotated_at` stamp** — never the values, which the gate must not handle. Fail on any path present in one cluster and not the other, or whose stamps differ beyond a configured window, reporting per path in the style `docs/e2e-gates.md:244-247` requires. Run it on the **scheduled-checks** lane, not only e2e: the divergence accrues on live instances, not on the harness. This is the single change that converts an unmeasured RPO into a measured one.

---

# Medium

## M-1. No stated RTO or RPO anywhere in the documentation set

**Impact: Medium** | **LoE to fix: Low**

**Evidence** — a case-insensitive grep for `RTO` / `RPO` / `disaster recovery` across `docs/` returns only the OpenBao seal-key escrow lines (`docs/secrets.md:125`, `docs/runbooks/bootstrap-openbao.md:55`, `docs/secrets.md:743`). No recovery-time or recovery-point objective is stated for the cluster, OpenBao, Terraform state, or object storage.

**Why it matters** — RTO/RPO is the contract an enterprise adopter's own continuity programme consumes; without it, every adopter re-derives it, and the derivation depends on facts (H-5's absent snapshots, the rebuild path's real duration) that are not written down either. It also means no design decision in the repo can be evaluated against a recovery target, so the targets are implicitly whatever the implementation happens to deliver.

**Recommendation** — add an RTO/RPO table to `docs/landing-zone-spec.md` covering cluster, OpenBao, Terraform state, and object storage, with each figure traced to the mechanism that delivers it. Where the honest answer today is "unbounded — no restore artifact exists" (OpenBao contents, per H-5), state that; a documented gap is actionable and an undocumented one is not.

## M-2. `instance-custom` escape-hatch tolerance is decided by name prefix, and fails open

**Impact: Medium** | **LoE to fix: Medium**

**Evidence**
- `tools/internal/health/argo.go:56-58` — `IsInstanceCustomApp` is `strings.HasPrefix(name, "instance-custom-")`.
- `tools/internal/health/argo.go:68-74` — `ClassifyArgoApp` remaps any matching app from `CatFail`/`CatPending` to `CatInstance`.
- `tools/internal/health/contract.go:41-56, 121-130` — `CatInstance` is excluded from `Verdict()` exactly like `Deferred`/`Drift`.
- `docs/architecture/convergence-contract.md:84` justifies the tolerance on **provenance**: "these Applications are instance-**owned** (`.template-manifest` `owned`, not `managed`)".

**Why it matters** — the doc's justification is provenance; the code's test is a string. Any Application named `instance-custom-*` is silently removed from the convergence verdict, whether or not it came from the instance-custom ApplicationSet. A hand-created app, or a platform app renamed into that space, becomes invisible to the gate that decides whether the platform is converged. This is a fail-open in the one place `AGENTS.md:48-52` says must fail closed ("a gate that passes having examined nothing looks exactly like the outage it exists to catch"). Impact is bounded — the ApplicationSet is the normal producer and `llz ci assert-instance-custom` hard-verifies the mechanism separately — but the invariant is asserted by convention rather than enforced.

**Recommendation** — key the remap on provenance: match on the `ownerReferences` the ApplicationSet stamps, or on a label the generator sets. `ParseArgoApp` (`argo.go:267-313`) already decodes `metadata`; adding one field and one predicate is small, and it makes the code's test match the doc's justification.

## M-3. The phase1 hard-fail downgrade has exactly one veto, and no way to report why it gave up

**Impact: Medium** | **LoE to fix: Medium** | *(the downgrade itself is a documented, accepted tradeoff)*

**Evidence**
- `tools/internal/health/converge.go:42-47` — `PhaseAwareExitCode` converts **every** exit 1 to exit 2 while `phase1` holds.
- `tools/cmd/llz/ci_health.go:432` — `demotePhase1 := phase1 && !r.GitAuthFailure`; `GitAuthFailure` is the only veto.
- `tools/internal/health/converge.go:28-41` documents the rationale (apl-core installs CRDs, webhook Services and endpoints across later helmfile phases) and the accepted cost ("A cluster genuinely stuck in phase1 still fails — it exhausts the budget").

**Why it matters** — the tradeoff is sound and labeled, but its cost concentrates on exactly the failures an operator most needs named. During phase1, a bad image tag, a Kyverno policy rejecting a platform workload, or an exhausted Linode quota are all unreportable until budget exhaustion — at which point the message is "budget exhausted with the cluster still in-progress" (and, per H-3, carries no component list). The contract doc at `docs/architecture/convergence-contract.md:28` already names several states as unambiguously terminal — ImagePullBackOff on a nonexistent image, a Job past `backoffLimit` — and none of them veto the downgrade the way a git-auth refusal does.

**Recommendation** — generalize the veto. Replace the single `GitAuthFailure` boolean with a `TerminalFailures []string` set on `health.Report`, populated by the predicates that already identify unrecoverable states, and veto the phase1 downgrade whenever it is non-empty. Fixing H-3 in the same change makes the budget-exhaustion message actionable even for the classes that remain downgraded.

---

# State management (Terraform / OpenTofu)

There is **no state locking of any kind** in this system: `use_lockfile` appears nowhere in the repo, and Linode Object Storage offers no DynamoDB equivalent. This is known and documented (`docs/workflows/llz-terraform.md:261-264` — "Linode OBJ has no state locking, so a single concurrency group … serializes all VPC applies to avoid concurrent same-state corruption"). The *tradeoff* is accepted; the finding is that the mitigation it names does not cover the write paths that exist.

State lives in Linode OBJ via the S3 backend, bucket from `TF_STATE_BUCKET` (default `<repo-name>-tfstate`, `tools/cmd/llz/tokens.go:160`), key layout `<module>/<deployment>/terraform.tfstate` (`instance-template/.github/actions/terraform-init/action.yml:128-129`).

## C-3. A PR plan job runs `tofu import` — a state **write** — in a concurrency group disjoint from the apply group

**Impact: Critical** | **LoE to fix: Low**

**Evidence**
- `instance-template/.github/workflows/llz-terraform.yml:89-91` — `group: terraform-infra-${{ inputs.region || 'pr' }}`. On a `pull_request` event `inputs.region` is empty (`:33-37`), so every PR run lands in `terraform-infra-pr`, while an apply for deployment `primary` lands in `terraform-infra-primary`. **Different groups run concurrently.**
- `instance-template/.github/workflows/llz-terraform.yml:180-187` — the PR job is not read-only: it runs `llz ci tf-import --region "$REGION"`, which shells out to `tofu import` (`tools/cmd/llz/ci.go:729, 874`). `terraform import` persists a new state serial.
- The PR job inits against the *live* per-deployment key: it calls the shared composite with no `state-key` override (`llz-terraform.yml:162-174`), so the composite default `cluster/<region>/terraform.tfstate` applies (`terraform-init/action.yml:128-129`) — the same object `apply-cluster` writes.

**Why it matters** — with no lock, two writers each GET the state object, mutate in memory, and PUT; the later PUT wins wholesale. A PR opened while a release apply is in flight silently discards what the apply recorded, including resources that now exist in Linode but no longer exist in state. The next apply proposes to *create* them and fails on duplicate-label/duplicate-VPC errors, landing the operator in the wedge class this repo maintains a dedicated triage skill for. Blast radius is a 20-30 minute cluster apply.

**Recommendation** — (1) **Enable S3-native locking.** OpenTofu 1.10+ supports `use_lockfile = true` on the S3 backend using conditional-write semantics, no DynamoDB; CI already runs OpenTofu 1.12.5 (`docs/adr/0008-opentofu-migration.md:62`). Add it to the static partial in all four `tools/internal/tfroots/roots/*/backend.tf`. This depends on Linode OBJ implementing `If-None-Match` conditional PUT — **probe it and record the measured result the way ADR 0007 probed SSE** (`docs/adr/0007-terraform-state-encryption.md:23-42`), which is already this repo's house standard. (2) Meanwhile, either drop `tf-import` from `plan-cluster-pr` or give that job `concurrency: group: terraform-infra-${{ matrix.region }}` so it contends with the apply it can corrupt — a two-line change matching the pattern `apply-vpc` already uses (`llz-terraform.yml:271-273`).

## C-4. `rotate-state-passphrase` declares itself exclusive against every Terraform job but uses a group no Terraform job shares

**Impact: Critical** | **LoE to fix: Low**

**Evidence**
- `instance-template/.github/workflows/llz-secret-rotation.yml:459-460` — the comment: "EXCLUSIVE against every other Terraform job — a concurrent apply would write state with one key while this rewrites it with another."
- `llz-secret-rotation.yml:479-481` — its actual group is `terraform-state-${{ matrix.region }}`. The apply group is `terraform-infra-<region>` (`llz-terraform.yml:90`). **`terraform-state-primary` ≠ `terraform-infra-primary`** — the stated exclusivity does not exist.
- The correct pattern is present two jobs away in the same file: `rotate-lke-admin` (`:173-175`) and `rotate-db-admin` (`:619-620`) both use `terraform-infra-${{ matrix.region }}`.
- The job rewrites every root's state (`llz-secret-rotation.yml:514` → `llz ci rotate-state-passphrase --apply`).
- A second incorrect belief sits at `:467-468`: "two deployments re-keying at once would contend on the shared VPC root's **state lock**." There is no state lock; `max-parallel: 1` at `:469` is doing that work.

**Why it matters** — a re-key rewrites every state object with a new encryption key. An apply racing it writes the *old* key's ciphertext over the new. While `TF_STATE_ENCRYPTION_PASSPHRASE_OLD` is still set the rollover appears to converge; once the operator deletes the old secret — which the runbook gates on this job exiting 0 (`llz-secret-rotation.yml:449-453`) — the clobbered root is **permanently unreadable**. This is the exact failure ADR 0007 identifies as unrecoverable (`docs/adr/0007-terraform-state-encryption.md:123-127`).

**Recommendation** — change `llz-secret-rotation.yml:480` to `group: terraform-infra-${{ matrix.region }}`. Correct the "state lock" claim at `:467-468`; an incorrect mental model of what serializes these jobs is what produced the bug. Then add a static `llz ci` guard asserting that every job running `tofu apply`/`import`/`rotate-state-passphrase` declares a `terraform-infra-*` group — this repo's `add-ci-guard` pattern is designed for exactly this, and it is what stops the class recurring.

## H-6. The `vpc` root's documented state key does not match the key CI uses — and the unit test pins the wrong value

**Impact: High** | **LoE to fix: Low**

**Evidence** — three sources, three keys:
- `tools/internal/tfroots/roots/vpc/backend.tf:10` — `key = "vpc/terraform.tfstate"`
- `tools/internal/tfroots/roots/vpc/main.tf:7` — "applied per-network (state key `vpc/<name>`)"
- `instance-template/.github/workflows/llz-terraform.yml:396` — `state-key: vpc/${{ steps.net.outputs.net }}/terraform.tfstate` (this is what actually runs)
- `tools/internal/tfroots/backend_key_test.go:28` asserts the **`backend.tf` comment**, i.e. the wrong one — while its own header (`:13-21`) states the hazard precisely: "`terraform init` against another root's state key loads that root's state, and every resource in it is absent from this configuration — so the next plan proposes DESTROYING them… this only bites a by-hand apply — which is what the runbooks describe."

**Why it matters** — an operator following `backend.tf:10` by hand inits `vpc/terraform.tfstate`, an object no CI run ever writes. That state is empty, so `tofu apply` creates a second VPC (or 409s on the label), and a later `tofu destroy` against that key does nothing while appearing to succeed. The test that exists specifically to catch cross-root key drift is currently *enforcing* it.

**Recommendation** — fix `roots/vpc/backend.tf:10`, update `backend_key_test.go:24-28`, and extend the test to assert against the **workflow's** `state-key:` expression rather than a hand-maintained second copy. That converts a comment-linter into a genuine coupling test, which is what the repo's own gate doctrine asks for.

## H-7. No state bucket versioning, no backup, no recovery runbook — while two documents tell the reader to restore a snapshot nothing produces

**Impact: High** | **LoE to fix: Low-Medium**

**Evidence**
- The state bucket is created by a bare API call — `tools/cmd/llz/tokens.go:164` → `CreateObjectStorageBucket`, body `{"cluster", "label"}` only (`tools/internal/linode/rotate.go:228-231`). No versioning, object lock, or lifecycle policy is configured anywhere; a repo-wide grep for `versioning` returns only Renovate config.
- The state bucket is deliberately not Terraform-managed (chicken-and-egg); the `object-storage` root manages only the Loki/Harbor buckets.
- `docs/adr/0008-opentofu-migration.md:68-70` — "Rolling back means **restoring a pre-migration state snapshot from the bucket**"; `:134` — "the pre-migration state snapshot is worth keeping." **Nothing keeps it.**
- `instance-template/terraform-iac-bootstrap/AGENTS.md:113` — "State corruption is not recoverable without a backup." Correct, and there is no backup.
- None of the 12 files in `docs/runbooks/` covers state loss, state corruption, or `terraform import` recovery.

**Why it matters** — the repo has correctly identified state loss as unrecoverable, written it down twice, and then built a provisioning path that leaves the operator no way to recover. Combined with C-3 and C-4 — two live paths that can clobber state — the probability side of this risk is not zero.

**Recommendation** — (1) enable bucket versioning at creation time in `llz tokens` (one `PUT ?versioning` after `tokens.go:164`), with a lifecycle rule expiring noncurrent versions; probe and record the result per the ADR 0007 precedent. (2) Add `docs/runbooks/state-loss.md` covering prior-version restore, the per-root `tofu import` re-adoption path, and the interaction with encryption (a restored object needs the passphrase *and* the key-provider name it was written under — see M-6). (3) Reconcile ADR 0008's two "snapshot" sentences with reality; as written they hand a false assurance to a future reader.

## M-5. State encryption is phase 1 — `enforced` is not set, so an unencrypted state write is accepted *(documented, accepted tradeoff)*

**Impact: Medium** | **LoE to fix: Low** (schedule, not redesign)

**Evidence** — `tools/internal/tfroots/roots/*/encryption.tf:54-66` uses `fallback { method = method.unencrypted.migrate }` (byte-identical across all four roots). Rationale and the two-phase plan are documented at `encryption.tf:30-44` and `docs/adr/0007-terraform-state-encryption.md:90-104`; status line `ADR 0007:3` — "accepted, phase 1 shipped. Phase 2 (enforcement) is a follow-up." `docs/adr/0012:140` independently flags that the fallback "is also what makes an unencrypted state file accepted."

**Assessment** — correctly documented and correctly reasoned. OpenTofu genuinely rejects `enforced` together with an `unencrypted` fallback (ADR 0007:117), so phase 1 is not optional. The posture-in-code / key-in-env split (ADR 0007:71-88) is good design.

**Recommendation** — phase 2 has no owner or trigger recorded. Add an `llz ci` verb that *reports* the phase-2 precondition per deployment (every root's state shows `encrypted_data`), so the flip becomes a measurement rather than a judgement call.

## M-6. The state encryption key **name** is a mutable repo variable with no coupling to the state it decrypts, and losing it is as fatal as losing the passphrase

**Impact: Medium** | **LoE to fix: Low**

**Evidence**
- The rotation mechanism itself is well built: `instance-template/.github/actions/tf-encryption-env/action.yml:131-166` emits both key providers during rollover plus an *encrypted* fallback (`:140-154`), so rotation never relaxes the posture; injection guards on both passphrases and on the key names as HCL identifiers (`:87-92`, `:99-108`, `:113-118`).
- The load-bearing constraint (`tf-encryption-env/action.yml:33-44`): "OpenTofu stores pbkdf2's salt at `meta["key_provider.pbkdf2.<name>"]`, so state can only be decrypted by presenting its passphrase **under the name it was WRITTEN with**… Defaults to `llz`… changing this default would strand all of them."
- That name lives in `vars.TF_STATE_ENCRYPTION_KEY_NAME` (`llz-terraform.yml:171`, `llz-secret-rotation.yml:501`) — a GitHub **repo variable**, mutable by any repo admin, with no guard tying it to what the state was written under.
- `docs/secrets.md:275`, `docs/quickstart.md:737,913` escrow the *passphrase*. A grep for `TF_STATE_ENCRYPTION_KEY_NAME` across `docs/` returns nothing.

**Why it matters** — after one rotation the decryption secret is a *tuple*, and the escrow checklist captures half of it. An operator who escrows the new passphrase but loses the repo variable holds a passphrase that cannot decrypt anything — indistinguishable from having lost it.

**Recommendation** — add the key name to the escrow checklist and secrets table, stated as "escrow the pair"; have `llz ci rotate-state-passphrase` print the `(key-name, written-at)` pair on success; and add a preflight that reads the state object's `meta` keys and fails with a named diagnosis when `key_provider.pbkdf2.<KEY_NAME>` is absent from state carrying `encrypted_data` — the same move the action already makes for a missing passphrase.

## L-1. Cross-instance state collision is prevented by convention, not construction

**Impact: Low** | **LoE to fix: Low**

**Evidence** — `tools/cmd/llz/tokens.go:160` defaults the bucket to `repoSlug(instanceRepo)+"-tfstate"`, and `repoSlug` (`tokens.go:547-552`) returns the **repo name only — the owner is discarded**. So `acme/landing-zone` and `globex/landing-zone` both default to `landing-zone-tfstate`. Reachable only when two instances share one Linode account (accounts are separate bucket namespaces), and the key layout below the bucket carries no instance discriminator, so `primary` collides with `primary` exactly. `CreateObjectStorageBucket` returns 2xx for an already-owned bucket (`tools/internal/linode/rotate.go:225-227`), so the collision is **silent at provision time** and first manifests as a destroy-everything plan.

**Recommendation** — include the owner in the default slug, or have `llz tokens` fail when the target bucket already exists *and* contains `*/terraform.tfstate` objects under a different instance's key set. A one-line HEAD probe catches it at the only moment it is cheap.

## L-2. No `prevent_destroy` anywhere — defensible for the cluster, worth adding for data buckets

**Impact: Low** | **LoE to fix: Low**

**Evidence** — `grep -rn 'prevent_destroy'` repo-wide returns no matches; the only `lifecycle` blocks are `ignore_changes` (`terraform-modules/llz-cluster/main.tf:92-94`, `firewall.tf:101-103`).

**Assessment** — defensible. Destruction is gated procedurally and thoroughly: `llz ci assert-destroy-confirm` runs at the head of all six destroy jobs (`llz-terraform.yml:618, 693, 749, 1003, 1053, 1283, 1339`) with a `destroy:<deployment>:<module>` token, and `infra-*` GitHub Environments carry required reviewers. A `prevent_destroy` on the cluster would also break the e2e teardown lane, a first-class supported workflow.

**Recommendation** — do **not** add it to `cluster`. **Do** add it to the four data buckets in `terraform-modules/llz-object-storage/main.tf:51-77`, which today carry three attributes each and no `lifecycle` block at all. Two reasons beyond the general one: `llz-terraform.yml:48-52` already states that "destroying a cluster is not consent to erase them" and defaults `drain_data_buckets: false`, so `prevent_destroy` makes a stated policy structural rather than a flippable default; and `docs/landing-zone-spec.md:252-258` documents that because the module declares no `create_before_destroy`, Terraform plans **destroy-then-create on all four buckets** when `objLabelPrefix` changes — deleting an empty bucket "and their data with them". That trap is documented but ungated. The correct form is a module variable (`protect_buckets`, default `true`) that the `destroy-object-storage` job sets false, matching the shape of the existing `assert-destroy-confirm` guard (`tools/cmd/llz/ci.go:80`).

**Related, and higher value than `prevent_destroy` itself:** enable **bucket versioning** on the four data buckets and on the Terraform state bucket at creation (`tools/cmd/llz/tokens.go:164`). A repo-wide grep for `versioning` returns no configuration anywhere. Loki chunks are the platform's log retention and the destination of OpenBao's audit stream (`values.yaml:769`); the Harbor bucket is every platform image; and the destroy path runs an `s3 rm --recursive`-equivalent drain (`llz-object-storage/main.tf:17-20, 109-111`). Without versioning, an errant drain is unrecoverable. Gate it with `llz ci assert-obj-versioning`, alongside the existing `assert-obj-encryption` (`tools/cmd/llz/ci.go:165`), which already proves a per-bucket property live — same lane, same shape. Cross-region replication is **not** recommended: Linode OBJ buckets are pinned to a single endpoint (`llz-object-storage/main.tf:30-45`), so replication would be an application-level copy job, and versioning gets most of the value for a fraction of the complexity.

## L-3. Two stale scar comments misdirect operators toward unprotected hand-apply paths

**Impact: Low** | **LoE to fix: Low**

**Evidence** — `tools/internal/tfroots/roots/databases/backend.tf:19-21` claims "the workflow jobs for this root do not exist yet, so applying it by hand … is currently the only way to run it." They exist: `apply-databases` (`llz-terraform.yml:1197`), `plan-destroy-databases` (`:1261`), `destroy-databases` (`:1316`). `backend_key_test.go:19-21` repeats the stale claim. Separately, `instance-template/terraform-iac-bootstrap/AGENTS.md:73` documents `force_path_style = true` while the roots use `use_path_style = true` (`roots/*/backend.tf:31`) — the removed pre-TF-1.6 spelling, presented directly above a copy-pasteable `terraform init` block (`:97-100`); and `:29-36` says "three generated roots" where there are four.

**Why it matters** — the `databases` comment is the sole justification given for a hand-apply path, which is the one path with *zero* concurrency protection. It tells a future operator that hand-applying is normal.

**Recommendation** — delete the "jobs do not exist yet" sentence from both locations and point at `apply-databases`; keep the destroy-hazard paragraph, which is still true. Correct the three `AGENTS.md` drifts.

> **Cross-cutting root cause for C-3, C-4 and H-6:** serialization is expressed in GitHub Actions `concurrency:` groups, which are strings with no relationship to the state object being written. Nothing makes "this job writes `cluster/primary/terraform.tfstate`" imply "this job holds `terraform-infra-primary`" — they are matched by hand. Five `group:` declarations govern Terraform state writes (`llz-terraform.yml:90`, `:272`; `llz-secret-rotation.yml:174`, `:480`, `:622`) and **two of the five are wrong**. `use_lockfile` fixes this at the layer that owns the resource; the group corrections are the mitigation while that is measured; a static guard on the job↔group↔state-key correspondence is what stops it recurring.

---

# Upgrade path and drift detection

`llz upgrade` (`tools/cmd/llz/commands.go:747-886`) is a five-stage **local** command: snapshot `owned` files, run `copier update --vcs-ref <ref>`, restore `owned`, overwrite `managed` from a clean second render, apply `.template-removals`, then four gates and a re-render. Ownership is declared in `instance-template/.template-manifest`; the class→action table is `ci_template_manifest.go:63-70`. Nothing in the command touches a cloud API — which is exactly what makes H-8 surprising.

## H-8. Merging an `llz upgrade` commit is a production platform deploy — no plan, no approval — while the Terraform half of the same upgrade is gated three ways

**Impact: High** | **LoE to fix: Low** (documentation + an optional diff flag)

**Evidence**
- The pin the upgrade moves is consumed as a **kustomize remote base**: `clusterspec.RemoteBase` builds `github.com/akamai-consulting/lke-landing-zone//platform-apl/<path>?ref=<pin>` (`tools/internal/clusterspec/kustomize.go:36, 51-52, 101`).
- `renderAfterUpgrade` (`commands.go:856, 899-910`) rewrites every committed `apl-values/<env>` kustomization on every upgrade — deliberately, because skipping it "left a live instance running three releases behind on what ArgoCD DEPLOYS" (`commands.go:843-852`).
- Argo syncs those **automatically with prune and selfHeal**: `platformBootstrapApplicationManifest` sets `syncPolicy.automated{prune:true, selfHeal:true}` (`ci_bootstrap_cluster_manifests.go:215-219`); the carved Apps do the same (`kustomize.go:454-459`), as does the `instance-custom` ApplicationSet (`kustomize.go:342-347`); `llz-secret-store` is pinned directly to the template ref (`ci_bootstrap_cluster_manifests.go:242`), which is why `ci_bootstrap_cluster.go:386` notes its `targetRevision` "changes every upgrade".
- Meanwhile the Terraform half of the same upgrade is dispatch-only (`llz-terraform.yml:265-271, 421-427`), runs under `environment: infra-${{ inputs.region }}` (`:277, :431`), and posts a plan on PRs (`:142-201`).

**Why it matters** — `llz upgrade --commit` plus a push to the default branch deploys a new platform-apl manifest tree to production. The only gate is whatever branch protection the adopter happened to configure. An operator reading `docs/workflows/llz-terraform.md:246-250` and `docs/playbooks/operator-onboarding.md:245` reasonably concludes that environment approval gates infra changes — half of it does; the Argo half deploys on merge. The repo names this tradeoff for the `kubernetes-custom/` escape hatch (`kustomize.go:225-228`, "a commit to the default branch deploys itself; the gate is that branch's review") but has not recorded it for the far larger case of the upgrade moving the platform manifests.

**Recommendation** — (a) state it in one sentence in `docs/adopter-guide.md` §4: merging an `llz upgrade` commit deploys the new platform manifests, and Argo syncs them automatically; (b) add `llz upgrade --diff-apl-values` (or fold it into the dry-run fix in L-7) so the Argo-visible delta is reviewable *before* commit — the only point at which a human can see it.

## H-9. Nothing compares the repo's desired state to the live cluster — there is no scheduled "all Argo Applications Synced" check

**Impact: High** | **LoE to fix: Low** (the assertion primitives already exist)

**Evidence**
- `llz render --check` is **repo-vs-repo**: it computes render targets from the spec and diffs against on-disk files (`render.go:179-203`), filtering out the gitignored TF build artifacts (`:191-198`). It answers only "do the committed `apl-values/<env>` artifacts match what the current spec and pin would render?"
- `llz drift` is **pin-vs-branch-head**: it compares `.copier-answers.yml`'s SHA against the template repo's branch head via `git ls-remote` (`drift.go:18, 38`). Nothing about the cluster.
- The Argo-sync assertion primitives exist but run only on the provisioning path: `ci_assert_argo_app.go:75, 129, 158`, `ci_assert_instance_custom.go:70-105`, `ci_diagnose_argocd.go:144`.
- The scheduled matrix (`llz-scheduled-checks.yml:74, 157, 222, 343, 443`) covers OpenBao seal, cert readiness, VAP binding, Keycloak client, secret age, lke-admin rotation SLA, credential single-pane, PrometheusRule health, and monthly template drift. **None asserts that every Argo Application is Synced.**

**Why it matters** — given H-8, everything deploys automatically with `selfHeal`, so a wedged Application is the failure mode that matters most, and it is precisely the one nothing observes. `ci_bootstrap_cluster_manifests.go:196-199` records a real instance where `platform-bootstrap` sat permanently OutOfSync with selfHeal re-applying in a loop (`autoHealAttemptsCount 7 on lke638381`) — found by hand, not by a check. This is the single highest-value addition in this review: it is the only check that would observe a bad upgrade *after* it deployed.

**Recommendation** — add an `assert-argo-synced` job to the weekly matrix in `llz-scheduled-checks.yml`, built from the existing primitives in `ci_assert_argo_app.go`, and route its result through the same Prometheus alert path the credential lane already uses (`llz-scheduled-checks.yml:322-324`, `llz ci alert-eval --strict`) so it produces an alert rather than a log line.

## H-10. Nothing inspects a Terraform plan for destroys or replacements, and the documented approval control is never verified to exist

**Impact: High** | **LoE to fix: Medium**

**Evidence**
- `llz ci tf-plan` (`ci_tfplan.go:58-99`) tees plan output to a file, retries once on a control-plane API flake (`:72-77`), and appends the last 80 lines to the step summary (`:85, 91-99`). **It parses nothing** — no destroy count, no "forces replacement" detection, no threshold.
- Plan and apply are consecutive steps in one job with nothing between them: `llz-terraform.yml:505-509` (plan → `tfplan.bin`) then `:511-519` (`llz ci tf-apply --plan tfplan.bin`). `ci.go:492-517` confirms `tf-apply` has no destructive-change guard either.
- The asserted control is GitHub Environment required reviewers on `infra-<region>` (`docs/workflows/llz-terraform.md:246-250`). **But `llz` only configures the deployment-branch policy, never reviewers:** `lockInfraEnvBranchPolicy` (`branchpolicy.go:36-128`) restricts the environment to `main` and deliberately sends `reviewers` only if already present (`:91-93`) to avoid a 422 on repos without a paid plan (`:79-84, 143-147`). On a private repo without GitHub Pro/Team/Enterprise the whole protection call fails and is downgraded to a warning (`errEnvProtectionUnsupported`, `:31, 106, 120`). Nothing anywhere asserts a reviewer exists.
- The destructive shapes are documented and real: `docs/lessons-learned.md:188-191` ("LKE pool inline-drift **destroys the pool**… without `ignore_changes = [pool]` a refresh plans to null the pool and Linode interprets that as 'delete the pool'"); `docs/adr/0005-managed-app-platform.md:56` (`apl_enabled` is `ForceNew` — "existing clusters must be recreated").

**Why it matters** — for a module-ref bump, which is exactly the case where a plan can be destructive, review depends entirely on a human reading an 80-line plan tail in a step summary, under an approval gate that may silently not be configured. The repo's own doctrine is that an unobserved control is indistinguishable from an absent one.

**Recommendation** — (1) have `tf-plan` run `tofu show -json tfplan.bin` (the plan file is already saved, so this is parsing, not re-planning), count `delete` and `create-then-delete` actions, and fail above zero for the `cluster` root unless an explicit `allow_destroy=true` dispatch input is set. (2) Add an `llz doctor` check that `infra-<env>` has at least one required reviewer, warning when the plan tier does not support it — the same "observe the control" standard `assert-rotation-health` and `assert-adopter-pin` already set.

## H-11. The upgrade path is never exercised end-to-end — the gate tests copier, and release-e2e is greenfield-only

**Impact: High** | **LoE to fix: Medium**

**Evidence**
- `llz ci upgrade-test` (`ci_upgrade_test_gate.go:253-425`) scaffolds at the previous release tag and calls `copierUpdateArgv(to)` **directly** (`:353`), asserting non-interactive update, answers preserved, pin advanced, clean merge (`:364, 381, 392, 399`). It never invokes `llz upgrade`.
- `runUpgrade` is reachable from exactly one place — `main.go:303`. So the manifest snapshot/restore, the managed-overwrite pass, `.template-removals`, the answer-regression gate, the conflict gate, `renderAfterUpgrade` and `commitUpgrade` are unit-tested in isolation and **never run together against a real copier run**. The ordering constraints between them are load-bearing and argued at length in comments (`commands.go:792-796, 854-855`) — and ordering is exactly what per-function unit tests cannot check.
- `release-e2e-lane.yml` jobs are `llz-functional`, `instantiate`, `provision`, `teardown`, `gate` (`:149, 227, 247, 282, 335`); the lane instantiates a fresh scaffold and applies from zero (`:16-35`). There is no "instantiate at N-1, upgrade to N" step.
- The repo already identified this class one layer down: `ci_upgrade_test_gate.go:1-11` says "the SCAFFOLD path was gated and the UPGRADE path, which every adopter takes on day 2 … was exercised by nobody" — and closed it *for copier*. The cluster-level equivalent is still open.

**Why it matters** — the failure modes that matter most on an upgrade (a module ref bump forcing resource replacement, a platform-apl manifest that no longer applies over the previous release's objects, an Argo App that goes OutOfSync-and-selfHeals in a loop) are invisible to a greenfield run by construction. Combined with H-8, an upgrade both deploys automatically and is untested against a live prior-version cluster.

**Recommendation** — (1) extend `llz ci upgrade-test` to invoke the `llz upgrade` binary in the built instance rather than `copier update` — it already has a real scaffold at a real prior tag in a real git repo, which is all `runUpgrade` needs; a small diff converts four copier-layer assertions into a genuine day-2 gate. (2) Add a dispatch-only `upgrade` e2e lane that provisions at the previous release tag, runs `llz upgrade` to the candidate, pushes, waits for converge, and tears down. Given the one-live-cluster quota (`release-e2e.yml:57-60`), quarterly is enough — it is the only thing that can see this class at all.

## H-12. The Linode API client has no rate-limit handling, and one caller discards the status code entirely

**Impact: High** | **LoE to fix: Low**

**Evidence**
- `tools/internal/linode/linode.go:34` — a bare `http.Client` with a per-request timeout. No `Transport` wrapper, no limiter.
- `:46-66` — `do()` is the single funnel for every verb ("Every verb below … funnels through here"). It builds the request, sets `Authorization`, calls `c.http.Do`, and returns. **No status inspection, no `429` branch, no `Retry-After`, no backoff.**
- `:78-95` — `ListRaw` returns `(nil, resp.StatusCode, nil)` on any non-2xx; its doc comment states "the returned error is non-nil only for transport or parse failures".
- A grep for `rate.?limit|429|backoff|TooManyRequests|retryable` across `tools/internal/linode/**` returns **zero matches**.
- `ListRaw`'s only production caller discards the status: `tools/cmd/llz/import_linode.go:124` — `if buckets, _, err := client.ListRaw(...); err == nil {`. A rate-limited response therefore yields `buckets == nil, err == nil` — **indistinguishable from "this account has no buckets"**.

**Why it matters** — this is the fail-open shape `docs/lessons-learned.md:121-133` names as the reason gates exist ("reported 'none matched the filter' — which reads exactly like 'nothing to do'"). It is reached by the traffic pattern the rest of the system generates: coincident cron literals across instances (M-9), unbounded per-env matrix fan-out (M-14), and a 4-attempt ACL retry loop. The first symptom of rate limiting will not be an error; it will be a scheduled check that reports clean. This scoping is precise — Terraform-path Linode calls go through the provider, which has its own retry behaviour and is outside this repo.

**Recommendation** — in `do()`, add a bounded retry on `429` and `5xx` honouring `Retry-After` with jittered exponential backoff; `runner_acl.go:79-89` is the in-repo precedent for the jitter shape. Make `ListRaw` fail closed by returning a typed error for `429` so a caller cannot spell it `_`, then fix `import_linode.go:124`. The gate is a unit test against an `httptest` server — the `base` field at `linode.go:24-27` exists as exactly that seam.

## H-13. The state-passphrase rotation job inits against a directory that does not exist in the repo

**Impact: High** | **LoE to fix: Low**

**Evidence**
- `instance-template/.github/workflows/llz-secret-rotation.yml:496` — the "Terraform init (rotation window — both keys)" step passes `module: aws-init-only` to the shared composite.
- `instance-template/.github/actions/terraform-init/action.yml:123` — the composite's init step runs with `working-directory: terraform-iac-bootstrap/${{ inputs.module }}`, i.e. `terraform-iac-bootstrap/aws-init-only`.
- **That directory exists nowhere.** A repo-wide grep for `aws-init-only` returns exactly one hit — the line above — and `find . -type d -name 'aws-init-only'` returns nothing.
- It is also outside the composite's own documented enum: `terraform-init/action.yml:8` declares `module` as "Module directory under terraform-iac-bootstrap/ **(cluster, object-storage, databases, vpc)**". Those four are what `tools/internal/tfroots/roots/` defines.
- **The obvious alternative explanation is eliminated.** The TF roots and tfvars are gitignored generated artifacts (`render.go:6-11`), and this job does a bare `actions/checkout` — so one might expect *every* `module:` value to fail here. It does not: the composite's **first** step regenerates them (`terraform-init/action.yml:80-89`, `llz render "$REGION" --tfvars-only`), described at `:74-76` as "the universal guarantee — every TF job inits through this action, so none can read a stale or missing `<env>.tfvars`". So the four real roots resolve, and `aws-init-only` — which no render produces — does not.
- GitHub Actions fails a step whose `working-directory` does not exist, so the job aborts **before** its next step (`:513`, `llz ci rotate-state-passphrase … --apply`) — the step that performs the actual re-key.

**Why it matters** — the encryption-passphrase rotation that ADR 0007 designs and documents appears never to have completed a run. The rotation machinery itself is genuinely well built (see M-6), which makes the failure easy to miss: the design, the HCL emission, the both-keys window and the verify-with-new-key-alone pass are all correct, and the job dies at a step that does none of that work. It also compounds C-4: a rotation that aborts early never reaches the racy re-key, so C-4's corruption window has likely not been exercised — but fixing C-4 without fixing this would open it.

**Scope of the claim:** the static facts above are verified. The runtime consequence (step fails → job aborts) follows from documented GitHub Actions behaviour but was not observed — confirm against the most recent monthly `secret-rotation` run, whose cron is `0 4 1 * *` (`instance-template/.github/workflows/secret-rotation.yml:23`). If that job has been green, the composite is tolerating the missing directory somehow and the finding reduces to the static one: a `module:` value outside the declared enum, unguarded.

**Recommendation** — check whether this job has ever succeeded (its cron is `0 4 1 * *`, `instance-template/.github/workflows/secret-rotation.yml:23`). The init step looks vestigial: `llz ci rotate-state-passphrase --roots-dir terraform` builds its own encryption config in one unit-tested place, per the comment at `llz-secret-rotation.yml:505-509`, so the composite init may simply be removable. If the step is needed for AWS credential/env wiring, point it at a root that exists. Then add the gate: an `llz lint` check asserting that every `module:` value passed to `terraform-init` names a directory `llz render` actually produces — a static, cheap coupling test of exactly the kind `docs/e2e-gates.md` prescribes, and one that would have caught this at PR time.

## H-14. Nothing verifies that the seal-key DR copy still exists — it is checked exactly once in the life of an instance

**Impact: High** | **LoE to fix: Low** (the probe code already exists)

**Evidence**
- The custody chain is three copies: the live `llz-openbao/openbao-unseal-key` Secret in etcd (`values.yaml:711-714`; `ci_bao_seed_seal_key.go:33, 208-218`), the `infra-<region>` GitHub Environment secret `OPENBAO_SEAL_KEY` (`ci_bao_seed_seal_key.go:198`; `docs/runbooks/bootstrap-openbao.md:55`), and an offline copy that is **manual operator action, unenforced** (`docs/runbooks/bootstrap-openbao.md:166-180`).
- The seed step restores from `OPENBAO_SEAL_KEY` only if the env var is set at that moment (`ci_bao_seed_seal_key.go:158-169`) — it never *verifies* on a normal run that the environment secret still holds the key the live Secret holds.
- The idempotent path (`ci_bao_seed_seal_key.go:134-137`) short-circuits **before** `resolveSealKey` is called, so on every run after the first, `infra-<region>::OPENBAO_SEAL_KEY` is not read, not compared, not validated. Deleting or overwriting it produces **no signal anywhere** until a namespace rebuild needs it.
- The runbook's own recovery instruction for the persistently-sealed case is to "check … that it matches `OPENBAO_SEAL_KEY` on the `infra-<env>` environment" (`:201`) — a comparison the tooling could make continuously and does not.
- The alert set (`platform-apl/components/observability/prometheus-rules/openbao-alerts.yaml:32, 42, 57`) covers `OpenBaoSealed`, `NoActiveLeader`, `RaftQuorumDegraded` — all fire *after* the key is already a problem. None is predictive.

**Scope note** — ADR 0012 (`docs/adr/0012-credential-observability-gaps.md:175-180`) explicitly accepts that the seal key is **never rotated**, and ADR 0012 also brought `OPENBAO_SEAL_KEY` onto the credential single pane. That accepted tradeoff is H-4 and is labeled there. What no record accepts, and no mechanism checks, is whether the **DR copy is present** — a distinct gap.

**Why it matters** — this is the highest-consequence single value in the platform. Its DR copy lives in a store that is write-only from the UI, mutable by anyone with Environment admin, and validated exactly once. The failure is perfectly silent until the moment it is fatal — the shape `docs/lessons-learned.md:130-133` names as the reason gates exist. The same argument applies verbatim to `TF_STATE_ENCRYPTION_PASSPHRASE`, which `tools/cmd/llz/state_passphrase.go:278-279` calls "the **same blast radius as OPENBAO_SEAL_KEY**" — and which compounds it, because every recovery action starts by fetching a kubeconfig from Terraform state (`docs/runbooks/bootstrap-openbao.md:137`), so losing that passphrase locks the operator out of the recovery path itself. (The mitigation exists and is correctly framed: `llz ci fetch-kubeconfig` is the Linode-API path that "needs no `terraform init`, no S3 backend and no git auth" — `docs/runbooks/bootstrap-openbao.md:267-270`.)

**Recommendation** — add `llz ci assert-escrow-present` covering **both** keys. The probe is already written: `state_passphrase.go:63-72, 91-124` implements a definite-answer-only presence check across repo and every `infra-*` environment scope, treating an indefinite answer as unknown rather than absent. Reuse it. Comparing values is impossible (GitHub has no read-back), so assert presence, and additionally assert the live seal Secret is exactly 32 bytes and decodes — the two failure modes `resolveSealKey` guards at seed time (`ci_bao_seed_seal_key.go:161-166`) and nothing re-checks afterwards. Wire it into the **scheduled-checks** lane so it runs against live instances, not only e2e.

## H-15. Three required-anti-affinity OpenBao replicas on a three-node pool leave zero scheduling headroom

**Impact: High** | **LoE to fix: Low**

**Evidence**
- OpenBao runs 3 replicas (`kubernetes-charts/llz-openbao-platform/values.yaml:271`) with `podAntiAffinity` / **`requiredDuringSchedulingIgnoredDuringExecution`** on `kubernetes.io/hostname`, inherited from the vendored subchart (`charts/openbao-0.13.0.tgz` → `openbao/values.yaml:623-631`); the wrapper overrides no `affinity:` key.
- The PodDisruptionBudget is enabled with computed `maxUnavailable: 1` for 3 replicas (subchart `openbao/values.yaml:977-982`; template `server-disruptionbudget.yaml:9, 23`; helper `_helpers.tpl:132-140`).
- **But the minimal spec example and the staging example both provision `nodePool.count: 3`** (`docs/landing-zone-spec.md:219`, `:158`). The `spec.defaults` example uses `count: 5` (`:102`), which is fine — the minimal and staging examples are not.

**Why it matters** — because the anti-affinity is `required`, not `preferred`, losing one worker node leaves the evicted OpenBao pod **permanently `Pending`**: there is no fourth node it can land on. Raft still has quorum at 2/3, so the platform keeps serving — but it is now one more node loss from total unavailability with no self-healing path. `OpenBaoRaftQuorumDegraded` (`openbao-alerts.yaml:57-58`, `count(vault_core_unsealed) < 3`) fires, but the remedy — add a node — is documented nowhere, and an operator reads that alert as a seal problem.

**Recommendation** — this is statically knowable from the spec, so make it a `llz render --check` validation error when a deployment enables `components.openbao` with `cluster.nodePool.count < 4`, worded to name the required-anti-affinity reason. That is the repo's house form, the same class as the existing HA-pair and CIDR-overlap validators (`docs/landing-zone-spec.md:408-412`). Alternatively raise the minimal and staging examples to `count: 4` — but the static guard is what prevents the next adopter from re-deriving it.

## M-7. The apl-core migration runbook contradicts shipped behaviour on the one lever operators reach for

**Impact: Medium** | **LoE to fix: Low**

**Evidence**
- `EffectiveAplChartVersion` (`clusterspec/aplversion.go:43-48`) has **no non-test consumer**. The pin is read only by `assertAplVersion` (`ci_assert_apl_version.go:73-97`) and by `llz ci validate-apl-values --chart-version` for a `helm template` schema check (`ci_apl_schema.go:81, 117-121, 169`) — both are *checks*.
- Nothing helm-installs apl-core: `ci_bootstrap_cluster.go:794-796` — "A managed cluster has no customer-`helm`-upgradeable `apl` release". ADR 0005's status line is explicit: "**managed is the ONLY mode**… LLZ no longer self-installs apl-core" (`docs/adr/0005-managed-app-platform.md:3-5`).
- `docs/adopter-guide.md` states the shipped behaviour correctly: `aplChartVersion` "optional | **Omit it.** … bootstrap does not consume this field, so a pin deploys nothing."
- **But** `docs/apl-core-migration-runbook.md:32-35` instructs the operator to update `spec.cluster.bootstrap.aplChartVersion` as a deploy lever; `:82-88` says `llz ci bootstrap-cluster` "**Helm-installs apl-core**"; and `:68-78` tells the operator to edit `<env>.tfvars` under `instance-template/terraform-iac-bootstrap/cluster` and run `terraform apply` there — a template path that does not exist in a rendered instance, where the TF roots and tfvars are gitignored generated artifacts (`render.go:6-11`).

**Why it matters** — this is the document an operator opens to perform an apl-core upgrade. Following it produces a spec edit that deploys nothing, a shell command against a path that does not exist, and a mental model in which LLZ controls the apl-core version. The v6 design doc already applied the correct remedy to itself (`docs/designs/apl-core-v6-migration.md:257-265` — "**This step is historical.** … following this step would have edited a file that feeds nothing"); the runbook did not get that treatment. Note that `docs/runbooks/README.md:3-5` states the standard this violates: "Every file here is a **live procedure**, not a record — if one describes something the platform no longer does, that is a bug in the runbook."

**Recommendation** — apply the v6 design's own remedy: mark the superseded phases historical and replace them with the managed reality (Linode owns the version and the timing; `aplChartVersion` moves only the assert floor and the schema check; the operator's levers are `llz ci prepare-apl-upgrade`, the lab checklist in `docs/designs/apl-core-v61-upgrade.md:115-127`, and the `assert-apl-version` floor). No gate would have caught this — pair the fix with the docs-guard extension suggested in H-1.

## M-8. There is no rollback path for an upgrade, in code or in documentation

**Impact: Medium** | **LoE to fix: Low** (documentation) | *(roll-forward is the repo's stated model)*

**Evidence** — a grep for `rollback|downgrade|revert` across non-test `tools/cmd/llz/*.go` returns only unrelated hits. There is no `llz downgrade`, no `--to`, no saved pre-upgrade state. `docs/adopter-guide.md:222-252` documents the upgrade in detail and names no rollback. `terraform-modules/RELEASING.md:39` states tags are immutable and "to release a change, cut a new one" — i.e. the intended recovery is roll-*forward*. The one documented rollback is at the cluster layer and is destructive: `docs/apl-core-migration-runbook.md:201-210` — "the rollback path is **recreate the LKE-E cluster**".

**Why it matters** — `llz upgrade --ref <older-tag>` is not a supported downgrade: copier's 3-way merge against an older template is untested (H-11), and `.template-removals` is only ever additive, so files deleted on the way up are not restored on the way down. An operator who upgrades into a broken release has `git revert` of the upgrade commit as their only lever — and that does not undo the GitHub repo *variables* the upgrade told them to re-pin (`reportCIImageSkew`, `commands.go:973-1007`).

**Recommendation** — state the supported recovery explicitly in `docs/adopter-guide.md` §4: `git revert` of the upgrade commit plus re-pinning `TF_IMAGE`/`KUBE_IMAGE`, and that downgrading the template ref is not supported. Add a gate only if downgrade is meant to work.

## M-9. Cron literals are identical in every scaffolded instance — no jitter, no stagger, no templating

**Impact: Medium** | **LoE to fix: Low**

**Evidence** — five bare literals with no copier token: `instance-template/.github/workflows/scheduled-checks.yml:18` (`0 2 * * 0`), `:21` (`0 6 * * *`), `:23` (`0 7 1 * *`); `secret-rotation.yml:23` (`0 4 1 * *`), `:25` (`30 3 * * *`). `copier.yml` asks **no** scheduling question — its entire question set is `upstream_org`, `instance_repo`, `openbao_team`, `llz_version`, `llz_image_ref` (`copier.yml:156-257`). The stubs are `merge`-classified so an operator *may* tune them (`instance-template/.template-manifest:114-127`), but `merge` is an affordance, not a default.

**Why it matters** — 50 instances scaffolded from this template all fire at exactly 02:00 / 03:30 / 04:00 / 06:00 / 07:00 UTC. Where those instances share a Linode account — the common case for one org running many system teams — the 06:00 spike is `instances × 3 jobs × N deployments` simultaneous control-plane-ACL read-modify-write cycles against one account's API, into the client described in H-12.

**Recommendation** — render a deterministic per-instance offset at scaffold time. The mechanism exists: the stubs are `merge`-classified so a rendered value survives, and copier can derive a stable offset from an existing answer (a hash of `instance_repo` → minute 0-59, hour 6 or 7 for the daily jobs). Ship it as hidden copier questions (`when: false`, as `llz_image_ref` already does at `copier.yml:248-251`) so adopters are not prompted. Spreading five crons across a 60-minute window converts a coordinated spike into flat load with zero behavioural change.

## M-10. The account-quota preflight — the guard for the documented 30-minute silent hang — ships disarmed

**Impact: Medium** | **LoE to fix: Low** | *(the quota ceiling itself is a documented, accepted constraint)*

**Evidence** — the failure is fully written up at `docs/lessons-learned.md:192-218`: a fresh LKE-E apply stuck on `Still creating...` to the job timeout, because "at the account's VPC quota the create hangs with **no API error**". The guard exists — `tools/cmd/llz/ci_preflight.go:70-71` defines `--vpc-limit`/`--vcpu-limit` and `:173-177, :199-203` fail the apply with an explicit `::error::` naming the hang. **But both default to `0`, which the flag help defines as report-only**, and the workflow wires them from repo variables defaulting to `'0'` (`llz-terraform.yml:484-485`). The runbook tells the operator to set them *after* the first failure (`docs/runbooks/first-build-failed.md:77`).

**Why it matters** — this is the hard ceiling on cluster count, and it is per-*account*, so two teams' instances on one Linode account share it and neither preflight knows about the other. A fresh instance ships with the guard off, so the Nth cluster-create hangs for 30 minutes and fails with no diagnostic — the documented worst experience in the repo.

**Recommendation** — an onboarding fix, not a code fix: add `PREFLIGHT_VPC_LIMIT`/`PREFLIGHT_VCPU_LIMIT` to the variable set `llz tokens` and `llz doctor` already assert on, as a warning when unset, with a message naming the 30-minute hang. The pattern to copy is `docs/adr/0009-unmeasurable-credential-coverage.md:104-107`: an *absent* value is reported explicitly rather than skipped, precisely because "a never-written credential is indistinguishable from a healthy one."

## M-11. Two rotation paths select their deployment by hand-typed string and by array index

**Impact: Medium** | **LoE to fix: Low**

**Evidence**
- `tools/internal/clusterspec/types.go:255-262` — `BroadPATDeployments` is a **space-separated string** listing which deployments' `infra-<d>` `LINODE_API_TOKEN` gets the rotated value. `llz env add` does not append to it.
- Four account-scoped jobs resolve their GitHub Environment by array index: `create-linode-pat` (`llz-secret-rotation.yml:298`), `revoke-linode-pat` (`:418`), `create-tf-state-key` (`:525`), `revoke-tf-state-key` (`:572`) — all `infra-${{ fromJSON(needs.discover.outputs.deployments)[0] }}`. The list is **sorted** (`llz-discover-deployments.yml:15-19`), so `[0]` is whichever deployment sorts first alphabetically.

**Why it matters** — everything else in the instance derives the deployment set dynamically from tfvars, and `docs/workflows/llz-scheduled-checks.md:41-46` makes that coupling the explicit design goal: it "makes 'checked but unrotated' and 'rotated but unchecked' deployments structurally impossible rather than merely unlikely." `BroadPATDeployments` is the one place that guarantee is broken by hand — a new deployment silently stops having its `LINODE_API_TOKEN` rotated, invisible until the daily reaper revokes the old PAT. Separately, adding a deployment named `alpha` to a `dev`/`staging`/`prod` instance silently moves account-wide PAT minting into `infra-alpha`; if that is a scratch environment or gets torn down, monthly rotation breaks while the daily revoke reaper keeps running. `[0]` is an array index, so there is no error to surface.

**Recommendation** — add an `llz ci` guard asserting `set(BroadPATDeployments) == set(llz env list --json)`, wired into `llz lint`; this is the same two-sides-of-a-contract class as `TestReaperRecognisesRelabelerOutput` (`docs/lessons-learned.md:121-129`), and safer than defaulting because the rotator's blast radius is an account-wide PAT. For the index, add an explicit `spec.platform.accountRotationEnv` (defaulting to current behaviour) and have `llz doctor` fail when it names a deployment that does not exist.

## M-12. Platform-component sizing is fixed and does not scale with cluster size; the reconciler is a singleton on the day-2 signal path

**Impact: Medium** | **LoE to fix: Medium**

**Evidence** — a repo-wide grep for `HorizontalPodAutoscaler|VerticalPodAutoscaler|autoscaling/v2` across `*.yaml/*.yml/*.tf/*.go/*.md` returns **zero matches**; no first-party HPA or VPA ships from this repo (apl-core's and Argo CD's own internal sizing is upstream and out of scope). What ships is fixed: OpenBao `replicas: 3` with `requests 250m/256Mi`, `limits 1000m/1Gi` (`kubernetes-charts/llz-openbao-platform/values.yaml:271, 467-473`); cert-automation JetStream `replicas: 3` (`llz-cert-automation/values.yaml:53`); `llz-reconciler` `replicas: 1` (`platform-apl/components/llzReconciler/llz-reconciler/deployment.yaml:31`); otel-collector `replicas: 1` (`platform-apl/components/observability/otel-collector.yaml:50`). The spec exposes only four sizing fields, covering two components (`clusterspec/types.go:246-251`).

**Why it matters** — a 3-node dev cluster and a 50-node prod cluster get identical platform sizing. `llz-reconciler` at `replicas: 1` is a singleton on the path that produces the credential metrics the daily single-pane job gates on (`llz-scheduled-checks.yml:305-324`) — and it *has* leader election (`tools/cmd/llz/reconcile_leader.go`), so it is built to run more than one but deployed at one. OpenBao's `replicas: 3` is correct for quorum, but changing it is two-file surgery, not a value change: `llz-openbao-platform/values.yaml:262-271` records that the `retry_join` blocks are written per-pod for exactly 3 peers and the `openbao-tls` cert SANs enumerate `platform-openbao-0..2`.

**Recommendation** — (1) expose `resources` and `replicas` for `llz-reconciler` and otel-collector as chart values, following the repo's own rule for map-valued defaults an operator may need to clear: `docs/lessons-learned.md:48-61` establishes these **must be gated by a scalar toggle** because `{}` and `null` both fail silently through Argo ApplicationSet values coalescing. Ship `resources.enabled: true` plus the map behind it, not a bare overridable map. (2) For OpenBao, leave `replicas: 3` and add a chart unit test asserting that `ha.replicas`, the `retry_join` peer count, and the `openbao-tls` cert `dnsNames` agree — so a future edit fails at build time rather than at raft-join.

## M-13. Shared VPC — the only mechanism that relieves the hardest scaling ceiling — is the one feature with no live proof

**Impact: Medium** | **LoE to fix: Medium** | *(the caveat is documented)*

**Evidence** — bring-your-own VPC is implemented end-to-end: `terraform-modules/llz-cluster/main.tf:6-9` (`create_vpc = var.vpc_id == ""`), `:17-22` (`count`), `:35-42` (subnet attaches either way), `:58-64` (cluster bound with **both** `vpc_id` and `subnet_id` — the comment records that passing `subnet_id` alone makes LKE-E silently provision its own `lke<id>` VPC, the confirmed root cause of the quota leak at `docs/lessons-learned.md:211-218`). Spec surface at `clusterspec/types.go:315-324`, rendered per-network with its own state key (`render.go:265-275`), applied by a dedicated `apply-vpc` job (`llz-terraform.yml:265-277, 421-427`). **The gap is stated in the repo:** `terraform-modules/llz-cluster/variables.tf:32-42` — "multiple LKE-E clusters sharing one VPC is **unverified**"; `docs/landing-zone-spec.md:427-434` — "What remains is a real `plan`/`apply` against Linode to confirm the `data.linode_vpcs` lookup + attach end-to-end."

**Why it matters** — shared VPC is the only way M same-region clusters avoid consuming M account VPCs, and account VPC quota is the confirmed cause of the 30-minute silent cluster-create hang (M-10). The feature that relieves the hardest ceiling is the one with no live proof.

**Recommendation** — add a shared-VPC lane to `release-e2e`: two same-region deployments referencing one `spec.networks` entry with non-overlapping `/14`s, asserting that `data.linode_vpcs` resolves, that both clusters report the same `vpc_id` output, and that exactly one `linode_vpc` exists on the account for the pair. Until that lane is green, keep the caveat in `variables.tf` — do not let a doc promote it to "supported".

## M-14. Three daily scheduled jobs per environment race the same LKE-E control-plane ACL, with unbounded matrix fan-out

**Impact: Medium** | **LoE to fix: Low** | *(documented; the fold is written up but unlanded)*

**Evidence** — three jobs in `llz-scheduled-checks.yml` share an identical `if: … github.event.schedule == '0 6 * * *'` gate, depend only on `discover`, and matrix over the same deployment list: `lke-admin-rotation-health` (`:157, 167, 171`), `credential-single-pane` (`:222, 232, 236`), `prometheusrule-health` (`:343, 353, 357`). Each runs `cluster-access` (opening the runner's egress IP in the LKE-E control-plane ACL — `:96-109, 182-195, 368-381`) and `lke-runner-acl` with `mode: revoke` (`:145-155, 206-215, 429-438`). Nothing sequences them: no `needs:` between them, `fail-fast: false`, and no `max-parallel` on any of the three — so **6 concurrent read-modify-write mutations of one shared per-cluster ACL object per deployment per day**. `max-parallel` appears exactly once in the entire instance workflow tree (`llz-secret-rotation.yml:469`). The fold is already designed: `docs/designs/instance-slimming.md:155-157` names it, and `docs/workflows/llz-scheduled-checks.md:56-62` records the identical weekly fold's rationale ("three fewer read-modify-write mutations of the shared LKE-E ACL object, which the four concurrent jobs were racing on every single week").

**Why it matters** — the ACL contention is per-cluster, so it does not worsen with environment count; what worsens is the aggregate Linode API call rate at a single instant (H-12) and the wall-clock cost of retry storms across the fan-out. Daily job count is `3N+1` for N deployments, each pulling a container. The existing mitigation — jittered ACL retry, `aclRetryDelay = 3s`, `aclMaxAttempts = 4` (`tools/cmd/llz/runner_acl.go:75-89`) — bounds the damage without removing the contention.

**Recommendation** — land the documented fold, using the existing weekly job (`:74-155`) as the template including its `if: always() && steps.kubeconfig.outputs.available == 'true'` per-step pattern and the `tee -a` rule at `:119-123` that becomes live once steps share a job. That takes daily from `3N+1` to `N+1`. Then add `max-parallel` (≈5) to the remaining per-env matrices; keep `fail-fast: false`, since the point of these matrices is that one bad deployment does not hide the others.

## M-15. A runbook documents a restore path that has no implementation

**Impact: Medium** | **LoE to fix: Low**

**Evidence** — `docs/runbooks/bootstrap-openbao.md:207`: "Use this if the cluster is already initialized and unsealed but configuration steps were missed or need to be re-applied — for example **after a fresh cluster replacement with data restored from a snapshot**." Per C-5, no snapshot mechanism exists anywhere in the repo. The runbook's "Re-configure" mode is real and works (`:205-218`), but it re-applies *configuration* (auth methods, policies, seeds) onto an already-live OpenBao; the clause promises a preceding data-restore step the platform cannot perform.

**Why it matters** — a runbook is executed verbatim during an incident. An operator reading this line during a real regional loss looks for the restore step, does not find it, and burns incident time discovering the sentence describes a capability that does not exist. This is the same class as the audit-pipeline regression at `docs/lessons-learned.md:112-120`: a document internally consistent with itself and with nothing in the cluster. It also violates the standard `docs/runbooks/README.md:3-5` sets for this directory.

**Recommendation** — until C-5 lands, replace the clause with an explicit statement of the gap so the runbook stops implying a path: *"There is no OpenBao data-restore path today — a cluster replacement loses OpenBao's contents and requires a first-time bootstrap plus re-seeding every credential."* Once C-5 lands, make it true and link the procedure.

## M-16. Managed Postgres ships with no declared backup posture, and nothing asserts one

**Impact: Medium** | **LoE to fix: Low**

**Evidence** — `terraform-modules/llz-databases/main.tf:26-59` provisions `linode_database_postgresql_v2` with `label`, `engine_id`, `region`, `type`, `cluster_size`, `private_network{…}`, and `updates{}`. **There is no backup, retention, PITR, or fork configuration in the resource, and no variable exposing one** (`variables.tf`: `name`, `region_suffix`, `region`, `engine_version`, `db_type`, `cluster_size`, `vpc_id`, `subnet_id`, `public_access`, `label_prefix`, `maintenance`). `cluster_size` defaults to `2` — "2 or 3 = high availability with standbys" (`variables.tf:44-45`), which is **availability, not backup**: a standby protects against node loss, not a dropped table, a bad migration, or a deleted cluster. Grepping `docs/designs/shared-managed-postgres.md` (366 lines) for `backup|restore|snapshot|fork|pitr|retention|recover` returns only `pg_dump`/`pg_restore` migration prose (`:213-216`) and one ADR 0007 reference (`:349`). `llz ci assert-database` exists (`tools/cmd/llz/ci.go:313`) but is a connectivity/health lane, not a backup assertion.

**Deliberately not asserted:** what Linode Managed Databases do provider-side. That is not knowable from this repository. The finding is that **the platform declares nothing, configures nothing, and verifies nothing.**

**Recommendation** — (1) establish the provider-side default from Linode's own documentation and record it in `docs/designs/shared-managed-postgres.md` — whether automatic backups exist, their retention, and whether PITR/fork is available. (2) Surface whatever knobs the provider exposes as module variables so the posture is *declared* rather than inherited, and extend `assert-database` to assert the declared posture is live. If the provider exposes no knobs, that is itself the finding and belongs in the DR record as an accepted residual — the form ADR 0009 and ADR 0012 already use.

## M-17. The OpenBao audit log lives on an `emptyDir`, and the predictive alert its own comment prescribes does not exist

**Impact: Medium** | **LoE to fix: Low**

**Evidence** — `auditStorage.enabled: false` with a 2Gi `emptyDir` named `audit` (`kubernetes-charts/llz-openbao-platform/values.yaml:459-466, 654-657`). The rationale is sound and explicit — a Promtail sidecar ships to Loki "within seconds; the local file is only a buffer," saving 30GB across the raft (`:459-463`) — and the risk is named in the same comment: "OpenBao audit devices are blocking — if the emptyDir fills, OpenBao stops serving. Monitor `kubelet_volume_stats`, alert at 75%." **That alert does not exist.** The rule group (`platform-apl/components/observability/prometheus-rules/openbao-alerts.yaml`) has `OpenBaoMetricsTargetDown` (`:22`), `OpenBaoSealed` (`:32`), `OpenBaoNoActiveLeader` (`:42`), `OpenBaoRaftQuorumDegraded` (`:57`), `OpenBaoLeaseExhaustion` (`:76-81`), `OpenBaoAuditLogFailure` (`:90-94`) — and no volume-usage rule. `OpenBaoAuditLogFailure` fires on `vault_audit_log_request_failure`, i.e. *after* writes are already failing, which for a blocking audit device means OpenBao has already stopped serving.

**Why it matters** — a filled buffer stops OpenBao serving, which stops every ExternalSecret in the cluster. The chart commits to a predictive control and ships without it, so the documented mitigation is prose only. Separately, during exactly the incident this review concerns, the audit log is the record of what was read and by whom — and it lives in a pod-local buffer destroyed with the pod, whose only surviving copy is in Loki, in the same cluster.

**Recommendation** — implement the rule the chart comment already commits to, in the existing `openbao-alerts.yaml` group. One caveat worth resolving first: `kubelet_volume_stats` is a PVC-scoped metric series and may not cover an `emptyDir` at all — if so the prescribed control is not merely unimplemented but unimplementable as written, and needs `container_fs_usage_bytes` or equivalent. Confirm against a live cluster before writing the rule. This is the cheapest closable item in this review.

## M-18. The break-glass path is well built and exercised by no live gate

**Impact: Medium** | **LoE to fix: Medium**

**Evidence** — the design is genuinely good: `breakglass-openbao.yml` → `llz ci bao-breakglass` regenerates a root token from the recovery quorum and returns it **encrypted to the operator's RSA public key**, never cleartext in a run log (`docs/runbooks/bootstrap-openbao.md:281-291`), with `generate`/`rotate`/`revoke` actions (`:296-302`) and a shared concurrency group with bootstrap (`:357-358`). But the only test is `tools/cmd/llz/ci_bao_breakglass_test.go`, a **unit** test. None of the 27 `assert-*` verbs (`tools/cmd/llz/ci.go:43-386`) exercises quorum regeneration against a live cluster, and none exercises the **seal-key restore** path: `OPENBAO_SEAL_KEY` appears in `instance-template/.github/workflows/llz-bootstrap-openbao.yml:19, 21, 86, 345` only as an input to `bao-seed-seal-key` on the normal bootstrap path, so the namespace-rebuild restore branch (`ci_bao_seed_seal_key.go:158-169`) is driven by no workflow at all. Note the recovery quorum cannot substitute: `docs/runbooks/bootstrap-openbao.md:353-356` is explicit that break-glass needs no `OPENBAO_SEAL_KEY` "(that is only for a namespace/data rebuild)".

**Recommendation** — extend the existing `llz-wedge-gameday` pattern (`tools/cmd/llz/ci_wedge_gameday.go`, `docs/workflows/llz-wedge-gameday.md`), which already injects a controlled fault and asserts containment, with a **seal-key restore rehearsal**: delete the `openbao-unseal-key` Secret, delete the OpenBao pods, re-run `llz ci bao-seed-seal-key --region <env>`, assert all three pods return to unsealed. This is the one DR path fully exercisable on the e2e cluster without destroying it, and it is the path a real incident takes.

## M-19. The cluster-rebuild path is documented for infrastructure and silent on what does not survive

**Impact: Medium** | **LoE to fix: Low**

**Evidence** — the rebuild itself is a single dispatch (`gh workflow run terraform.yml --field region=<env> --field action=apply --field module=all`) walking vpc → cluster → object-storage → in-cluster bootstrap → OpenBao (`docs/runbooks/bootstrap-openbao.md:105-121`), described as "the supported path on a fresh-cluster rebuild" (`:121`). The teardown side is unusually thorough (`docs/workflows/llz-terraform.md:522-575`). What is not written down is what the rebuild *loses*:

| Asset | On rebuild | Evidence |
|---|---|---|
| OpenBao **data** (all secrets) | **LOST** — no snapshot to restore | C-5 |
| OpenBao seal key | Recoverable iff `infra-<env>::OPENBAO_SEAL_KEY` survived — and only helps if the raft Volumes also survived | `ci_bao_seed_seal_key.go:158-169` |
| Terraform state | Recoverable iff the passphrase survived; bucket unversioned | H-7, H-14 |
| Loki chunks / Harbor images | Buckets survive a cluster destroy — but unversioned, and the destroy path drains them | `terraform-modules/llz-object-storage/main.tf:17-20` |
| Block Storage Volumes | `Retain` protects them from CSI, but the destroy job's sweep deletes them by design | `block-storage-class.yaml:79-84`; `docs/workflows/llz-terraform.md:553-575` |
| Harbor robot credentials | Re-minted by the in-cluster provisioner within ~5 min | `docs/runbooks/bootstrap-openbao.md:147` |
| `openbao-tls`, client CAs | Re-issued by cert-manager from a stable self-signed CA | `docs/runbooks/bootstrap-openbao.md:197-199` |

The rebuild's duration is stated nowhere and is not derivable from the repo.

**Recommendation** — add `docs/runbooks/cluster-rebuild.md` whose spine is that table: what survives, what does not, and in what order to restore. It is worth writing before C-5 lands, for two reasons: it is the document an operator opens at hour zero of a regional incident, and writing it forces the RTO/RPO statement M-1 is missing.

## M-20. OpenBao's `updateStrategyType: OnDelete` is inherited, correct, and undocumented

**Impact: Medium** | **LoE to fix: Low**

**Evidence** — inherited from the vendored subchart (`openbao/values.yaml:391`); a grep for `updateStrategy` in `kubernetes-charts/llz-openbao-platform/values.yaml` returns nothing, so the wrapper does not override it. A StatefulSet spec change — new image tag, new config — therefore does **not** roll pods. The only thing that deletes OpenBao pods today is `openbao-cert-watcher` on leaf renewal (`docs/runbooks/bootstrap-openbao.md:199`; `platform-apl/components/openbao/openbao-cert-watcher.yaml`).

**Why it matters** — `OnDelete` is *defensible* for a raft store, since uncontrolled rolls break quorum. But it is undocumented, so an operator bumping `openbao.server.image.tag` (`values.yaml:241`) sees Argo report Synced/Healthy while every pod keeps running the old image — for up to ~80 days, until the next cert renewal happens to restart them. That is a silent no-op on a security-relevant upgrade.

**Recommendation** — add a comment at `values.yaml:241` naming the `OnDelete` consequence and the supported way to roll (delete pods one at a time, waiting for raft to re-converge between each), in the repo's scars-as-defaults style. This is exactly the class of non-obvious value the convention exists for.

---

# Low

## L-4. The node autoscaler is wired and adopter-exposed but off by default, and the default is not stated

**Impact: Low** | **LoE to fix: Low**

**Evidence** — `tools/internal/tfroots/roots/cluster/main.tf:87-93` emits a `dynamic "autoscaler"` block with `node_count = var.autoscaler_enabled ? null : var.node_count` (`:74-75`). Defaults are `autoscaler_enabled = false`, `min = 3`, `max = 6` (`roots/cluster/variables.tf:100-116`). The adopter surface exists (`clusterspec/types.go:284-295`, mapped at `tfvars_map.go:45-53`, settable via `llz env set`), and merge semantics correctly honour an explicit `false` over an inherited `true` (`clusterspec/merge.go:40`, tested at `instance_test.go:224-243`). `docs/adopter-guide.md:146` groups the autoscaler with "keep unless sizing differs", which reads as guidance to leave it off.

**Why it matters** — with the autoscaler off, every cluster is a fixed `node_count`, so an org's only elasticity lever is a Terraform apply. That is a defensible landing-zone default (predictable cost, deterministic e2e) but it should be a *stated* default.

**Recommendation** — state the default and its tradeoff in `docs/adopter-guide.md` with a one-line "turn it on" snippet, and add an `llz doctor` advisory (not a failure) when `autoscaler_enabled = false` and `promotion_rank > 0` — a cluster in a real promotion pipeline pinned to a static node count.

## L-5. One node pool per cluster, with hardcoded labels

**Impact: Low** | **LoE to fix: Medium**

**Evidence** — `tools/internal/tfroots/roots/cluster/main.tf:69-93` declares exactly one `linode_lke_node_pool "this"` with literal labels `environment = "shared"`, `role = "observability"`. The spec models a single `NodePool` struct, not a list (`clusterspec/types.go:272, 284-295`). The module README states the intent — "This module intentionally creates no node pools. Add them as separate resources in the calling configuration" (`terraform-modules/llz-cluster/README.md:86`) — but the shipped calling configuration adds exactly one.

**Why it matters** — there is an incident behind this: `docs/lessons-learned.md:48-61` records gsap-apl burning three iterations trying to clear a `nodeSelector: {workload: builds}` "that the cluster had no pool for". A single-pool cluster cannot separate platform from workload, host a build/GPU/memory-optimised pool, or drain one class of node independently — so "more clusters" becomes the only answer to a workload needing different hardware, multiplying control planes, VPCs, OpenBao instances and CI fan-out for what should be a second pool.

**Recommendation** — promote `cluster.nodePool` to a keyed map (accepting the existing scalar as the `default` key) and emit `for_each` over `linode_lke_node_pool`, moving the two hardcoded labels into the per-pool spec with current values as defaults. The `moved {}` block at `main.tf:63-66` is the in-repo precedent for doing this without a destroy/recreate, which matters because `node_type` is ForceNew.

## L-6. No documented tested limit for nodes, environments, clusters, or apps

**Impact: Low** | **LoE to fix: Low**

**Evidence** — searches across `docs/` and `README.md` for `max (nodes|envs|environments|apps|clusters)`, `tested (up )?to`, `scales? to`, `at most [0-9]+`, `limit of [0-9]+` return no capacity statement anywhere; every hit concerns Linode account quotas or e2e lane sequencing. The nearest stated constraint is the honest negative at `docs/lessons-learned.md:198-199`: "There is no Linode compute-quota API, so any 'quota exceeded' claim is unverifiable from automation."

**Recommendation** — add a short "Scale and limits" section to `docs/adopter-guide.md` stating what is *known* rather than inventing a tested number: the per-deployment job math (M-14), the per-deployment artefacts, the account-quota ceiling and the `PREFLIGHT_*` variables that catch it (M-10), and the shared-VPC caveat (M-13). Say explicitly that no upper bound has been load-tested, and pair it with M-10's `llz doctor` advisory so the ceiling is enforced rather than only documented.

## L-7. `llz upgrade` has no transactional guarantee, a near-vacuous `--dry-run`, a dirty-tree `git add -A`, and gates that skip silently

**Impact: Low** | **LoE to fix: Low** (four independent small fixes)

**Evidence**
- **Not transactional:** `overwriteManagedFromScaffold` (`upgrade_policy.go:218-244`) returns mid-loop on the first copy failure; files already copied stay copied, and the pin has already advanced because copier rewrote `.copier-answers.yml` in step 3.
- **Owned snapshot lost on copier failure:** `commands.go:784-786` returns on copier error, but the restore lives in `applyUpgradeManifestPolicy`, only reached at `:788` on success; `defer owned.cleanup()` (`:782`) then removes the temp copy. The code's own comment at `:748-749` says "a failed `llz upgrade` is mid-flight" — and nothing acts on it. (Blast radius is bounded: `checkCopierFencing` (`ci_template_manifest.go:215-265`) ensures shipped `owned` files are in copier's `_skip_if_exists`/`_exclude`.)
- **`--dry-run` cannot preview the diff:** `run()` returns before exec on dry-run (`commands.go:196-202`), so `copier update` never runs; `applyUpgradeManifestPolicy` prints two counts and returns (`upgrade_policy.go:154-159`), and `runUpgrade` returns at `commands.go:800-802` before the answer gate, conflict gate, re-render and diffstat. `printUpgradeSummary` (`commands.go:957-971`) — the one place churn is visible — never runs.
- **`git add -A` with no clean-tree precondition:** `commands.go:1012-1030`; the only guard is "is the tree empty" (`:1013`). Unrelated in-flight edits get folded into a commit labelled a template upgrade — the exact commit whose stated purpose (`:878-880`) is "the operator reviews ONE diff".
- **Gates skip silently:** `ci_upgrade_test_gate.go:276-279` ("SKIPPED — copier not installed") and `:301-307` ("SKIPPED — no vX.Y.Z tag to upgrade from (shallow clone?)") both return nil, so a green `make lint` can mean "the upgrade path was not checked".

**Recommendation** — (1) stage the managed overwrite into a temp dir and rename in, or name the already-overwritten files in the error. (2) Move the `owned` restore into a `defer` that also runs on the error path, and print the recovery command on any post-copier failure. (3) Make `--dry-run` render the target scaffold (`renderUpgradeScaffold`, `upgrade_policy.go:177-194`, already does this) and print a real diffstat for the `managed` set — no copier update needed. (4) Capture `git status --porcelain` before step 3 and refuse `--commit` if the tree was dirty, or stage only the paths the upgrade touched. (5) Make the skips fail when `CI=true` and the reason is fixable, or at minimum emit `::warning::` so the skip is visible in the run summary.

---

# What's DONE WELL

This deserves to be stated plainly, because the findings above are corrections at the margin of a system whose centre is strong.

1. **The convergence contract is real engineering, not documentation theatre.** Four exit codes with a stated caller obligation for each, an explicit rule that "I don't know" is exit 2 and never exit 0 (`convergence-contract.md:64-66`), and an explicit rule that Phase 0 is exit 2 rather than exit 0 (`:63`). The named anti-patterns at `:122-131` are the kind of thing most teams learn and never write down.

2. **Fail-closed on vacuity is implemented, not just asserted.** `sectionItems` records `CatPending` with the reason "treating as inconclusive rather than 'none found'" when a list fails (`ci_health.go:545-552`), and `checkOpenBao` refuses to judge leader count on measurements that never happened (`ci_health.go:937-956`). `kubectl_probe.go:1-40` is an unusually clear statement of why "the resource is not there" and "we never got an answer" are different claims.

3. **Scars carry their incidents.** `IsGitAuthError` names the run that burned a 1200s budget (`argo.go:174-195`); the apl-pipeline stage budgets record that 6600s of ceilings sat inside a 70m job (`ci_wait_apl_pipeline.go:73-88`); `healthNamespaces` records that three stale namespace names silently disabled whole sections (`ci_health.go:21-36`). A future maintainer cannot remove these safely-looking values without reading why they exist.

4. **Self-healing is bounded and justified.** The argocd-redis realign and the 256KB-annotation strip each run **once per converge run** (`ci_health.go:227-243`), each with an explicit "if it doesn't clear, the budget still bounds the poll" argument. This is the correct shape for an automatic remediation — and it is consistent with the contract's own rule 6 against CI-imperative nudges, because these repair a specific diagnosed wedge rather than papering over reconcile latency.

5. **The exit codes are treated as an ABI.** The deliberate `os.Exit` at `ci_health.go:74-82` and `ci_health_incluster.go:49-58`, each with a comment explaining that returning an error would collapse 2 and 3 into cobra's exit 1, shows the contract is understood as load-bearing at the process boundary — including the subtlety that `--fail-on-unhealthy=false` suppresses 1 and 2 but deliberately does *not* suppress 3.

6. **One predicate library, two callers.** `llz ci health` (kubectl) and `llz ci health-incluster` (REST via ServiceAccount) share `health.ClassifyArgoApp` rather than re-implementing the classification (`ci_health_incluster.go:1-21`). This is exactly the "assert at the consumer, call both sides' real functions" doctrine from `docs/e2e-gates.md` applied to the system's own structure.

7. **Test discipline matches the stakes.** Mutation tests exist for the converge budget arithmetic specifically because "the deadline is budget SECONDS from now" is invisible to a single-poll test (`ci_health_mutation_test.go:24-35`), and `withConvergePoll` fails the test if the loop requests more scans than scripted (`ci_health_test.go:657-672`) — turning wasted work into a test failure. The gap in C-1 is a missed case, not an absence of rigour.

8. **Documented tradeoffs are genuinely documented.** The static seal key (`docs/secrets.md:734`), the rejected cross-region raft topology (`:108`), the absence of OBJ state locking (`docs/workflows/llz-terraform.md:261`), and the phase1 downgrade (`converge.go:28-41`) are all recorded with their reasoning and their accepted cost. Several findings above are gaps in the *mitigation* of an acknowledged tradeoff, not in the acknowledgement.
