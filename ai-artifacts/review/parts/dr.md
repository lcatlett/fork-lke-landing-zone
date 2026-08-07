# Disaster Recovery posture — LKE-Enterprise landing zone template

**Scope:** read-only review of the DR posture of `fork-lke-landing-zone`.
**Method:** every claim below cites `file:line`. Absence claims are grepped, not
assumed — the grep scopes are stated inline so they can be re-run.

**16 findings:** 2 CRITICAL (F-1, F-2), 6 HIGH (0a, F-3, F-4a, F-4b, F-6, F-7,
F-14 — see note), 6 MEDIUM (0b, F-4c, F-5, F-8, F-9, F-10, F-11), 2
INFORMATIONAL (F-12, F-13). Counted as 16 discrete findings: F-1, F-2, F-3, F-4a,
F-4b, F-4c, F-5, F-6, F-7, F-8, F-9, F-10, F-11, F-12, F-13, F-14, plus
sub-findings 0a (High) and 0b (Medium) attached to the topology table. Severity
buckets overlap the sub-findings, which is why the bucket totals exceed 16.

**Documented-accepted-tradeoff check:** `docs/adr/` (14 records, index at
`docs/adr/README.md:12-27`) and `docs/lessons-learned.md` (323 lines, read in
full) were checked against every finding. **One** finding is already a documented
accepted tradeoff (F-4a, ADR 0012); **one more is a documented deliberate
exclusion** outside the ADR set (F-7's rotation drill,
`docs/workflows/llz-bootstrap-openbao.md:836-838`). **No DR ADR exists** — see F-13.

**Reading note.** `docs/landing-zone-spec.md`, `docs/lessons-learned.md`,
`docs/runbooks/bootstrap-openbao.md`, `docs/runbooks/openbao-team-login.md`,
`docs/playbooks/openbao-accounts.md`, `docs/workflows/llz-bootstrap-openbao.md`,
`docs/workflows/llz-breakglass-openbao.md`,
`docs/designs/shared-managed-postgres.md`, the OpenBao wrapper chart values and
the vendored subchart were each read in full. **`docs/secrets.md` is the one
exception**: this session's tooling refused to open it directly (a
permission/hook block on that path), so it was read through targeted
`grep -A/-B` extraction of every section relevant here — Topology, Why
operator-side dual-write, Initial bring-up, Writing/rotating, Regional failover,
In-cluster TLS, Unseal automation, Cross-references. Claims cited to `secrets.md`
below are from that extracted text and are quoted verbatim; a reader wanting
whole-file assurance should re-read it directly.

---

## 0. Effective OpenBao HA topology — the part that is CORRECT

Stating this first because the wrapper chart's `values.yaml` is silent on
anti-affinity and PodDisruptionBudget, and reading only the wrapper would produce
a false gap. The vendored subchart
(`kubernetes-charts/llz-openbao-platform/charts/openbao-0.13.0.tgz`, extracted
read-only for this review) supplies both as non-empty defaults, and the wrapper
overrides neither.

| Property | Effective value | Evidence |
|---|---|---|
| Storage backend | Integrated **Raft** | `kubernetes-charts/llz-openbao-platform/values.yaml:382` (`storage "raft" { path = "/openbao/data" }`) |
| Replicas | **3** (quorum tolerates 1 loss) | `values.yaml:271`; rationale `values.yaml:264-270` |
| Pod anti-affinity | `podAntiAffinity` / `requiredDuringSchedulingIgnoredDuringExecution` on `kubernetes.io/hostname` — **inherited**, wrapper does not override | subchart `openbao/values.yaml:623-631`; no `affinity:` key under `openbao.server` in `kubernetes-charts/llz-openbao-platform/values.yaml` |
| PodDisruptionBudget | **enabled**, `maxUnavailable` computed = **1** for 3 replicas | subchart `openbao/values.yaml:977-982` (`disruptionBudget.enabled: true`, `maxUnavailable: null`); template `openbao/templates/server-disruptionbudget.yaml:9,23`; helper `openbao/templates/_helpers.tpl:132-140` → `div (sub (div (mul 3 10) 2) 1) 10` = 1 |
| Data PVC | `10Gi`, `ReadWriteOnce`, class `block-storage-retain` | `values.yaml:453-457` |
| Reclaim policy | **`Retain`** — a PVC delete does not delete the Linode Volume | `tools/cmd/llz/manifests/block-storage-class.yaml:135`, rationale `:79-84` |
| Volume encryption at rest | yes (`linodebs.csi.linode.com/encrypted: "true"`) | `block-storage-class.yaml:139` |
| Peer discovery | `retry_join` to all 3 pod FQDNs over mTLS | `values.yaml:410-427` |
| Audit log | `emptyDir` 2Gi, tailed by Promtail → Loki | `values.yaml:459-466, 654-657` |

`Retain` is a genuine and load-bearing DR control: it means the raft data Volume
survives a namespace teardown or an accidental PVC delete, and it is the *only*
mechanism protecting OpenBao's data from deletion today (see F-1).

**Sub-finding 0a (High) — 3 required-anti-affinity replicas on a 3-node pool has
zero scheduling headroom.** The anti-affinity is `required…`, not `preferred…`
(subchart `openbao/values.yaml:625`). The minimal spec example and the staging
example both provision `nodePool.count: 3`
(`docs/landing-zone-spec.md:219`, `:158`). Lose one worker node and the evicted
OpenBao pod is **permanently `Pending`** — there is no fourth node it may land
on. Raft still has quorum at 2/3, but the cluster is then one more node loss from
total unavailability, with no self-healing path. The `spec.defaults` example uses
`count: 5` (`docs/landing-zone-spec.md:102`), which is fine; the *minimal* and
*staging* examples are not.

**Why it matters:** the failure is silent from the health tree's perspective —
`OpenBaoRaftQuorumDegraded` (`platform-apl/components/observability/prometheus-rules/openbao-alerts.yaml:57-58`,
`count(vault_core_unsealed) < 3`) will fire, but the remedy (add a node) is not
documented anywhere and the operator will read it as a seal problem.

**Recommendation:** the required outcome is that a spec which enables
`components.openbao` cannot describe a node pool whose *maximum* size leaves no
spare node for an evicted OpenBao pod. State it that way and leave the predicate
to the implementer, because the naive form false-positives: a `count: 3` pool
with `nodePool.autoscalerEnabled: true` and `autoscalerMax` at its default of 6
(`docs/landing-zone-spec.md:334-336`) *would* provision a replacement node, and a
hard error on `count < 4` alone would reject a correct spec. The check therefore
has to read the effective ceiling — `autoscalerMax` when autoscaling is on,
`count` when it is not — and only then fail. This is a spec-level static guard,
same class as the existing HA-pair/CIDR-overlap validators
(`docs/landing-zone-spec.md:408-412`), so `llz render --check` is the right home.
The cheaper half, worth doing regardless: raise the minimal and staging examples
off `count: 3`, or annotate them with the constraint.

**Sub-finding 0b (Medium) — `updateStrategyType: OnDelete`.** Inherited from the
subchart (`openbao/values.yaml:391`); the wrapper does not override it (grep for
`updateStrategy` in `kubernetes-charts/llz-openbao-platform/values.yaml` returns
nothing). A StatefulSet spec change — new image tag, new config — does **not**
roll pods. The only thing that deletes OpenBao pods today is
`openbao-cert-watcher` on leaf renewal
(`docs/runbooks/bootstrap-openbao.md:199`;
`platform-apl/components/openbao/openbao-cert-watcher.yaml`). This is defensible
for a raft store (uncontrolled rolls break quorum) but it is **undocumented**:
an operator bumping `openbao.server.image.tag` (`values.yaml:241`) will see Argo
report Synced/Healthy while every pod runs the old image, for up to ~80 days
until the next cert renewal. Recommend a comment at `values.yaml:241` naming the
`OnDelete` consequence, in the repo's scars-as-defaults style.

---

## F-1 (CRITICAL) — There is no raft snapshot. There is no backup of OpenBao's data at all.

**Evidence — the absence, grepped:**

```
grep -rni 'raft.snapshot|operator raft snapshot|snapshot save|snapshot restore|bao operator raft' \
  docs/ tools/ kubernetes-charts/ instance-template/ terraform-modules/ platform-apl/ Makefile
```

returns exactly **two** hits, neither of which is a snapshot mechanism:

- `tools/cmd/llz/manifests/kyverno-pvc-deny-untaggable-clone.yaml:8` — a policy
  that *denies* snapshot-sourced PVCs (see F-2).
- `kubernetes-charts/llz-openbao-platform/values.yaml:387` — prose in a comment
  noting that `retry_join` removed the manual `bao operator raft join` step.

A broader `grep -rni 'snapshot'` over the same tree (60+ hits, reviewed) finds
only: Terraform *state* snapshots (`docs/adr/0008-opentofu-migration.md:69,134`),
apl-values git branch snapshots
(`docs/designs/apl-core-values-branch-isolation.md:37,104,114,184`), Argo
Application health snapshots in the wedge gameday
(`tools/cmd/llz/ci_wedge_gameday.go:39-40,148,187,238`), an upgrade file-mode
snapshot (`tools/cmd/llz/upgrade_policy.go:17-58`), and a Linode Volume-**ID**
capture before teardown (`docs/workflows/llz-terraform.md:532` — see below).
**None of these is a data backup of OpenBao.**

The one that reads most like a backup is not one:
`docs/workflows/llz-terraform.md:531-533` says the destroy job "must snapshot, by
id, exactly the `pvc-*` Volumes attached to this cluster's nodes while the
cluster still exists" — read in context (`:553-575`) that is recording Volume
**identifiers** so the post-destroy sweep can **delete** them. It is the opposite
of a backup.

**What this means concretely.** OpenBao holds the platform's entire credential
set — `secret/linode/api-token`, `secret/harbor/robot`,
`secret/loki/object-store`, `secret/harbor/registry-s3`,
`secret/cert-automation/github-token`, `secret/infra/github-dispatch-token`, every
`spec.teams` subtree (`docs/runbooks/bootstrap-openbao.md:146-161`,
`docs/landing-zone-spec.md:344-367`). The **only** thing standing between that
and total loss is three `ReadWriteOnce` Linode Block Storage Volumes in **one
region**, protected by `reclaimPolicy: Retain`. There is:

- no periodic `bao operator raft snapshot save`,
- no destination for one (the object-storage module provisions exactly four
  buckets — harbor-registry and three Loki buckets —
  `terraform-modules/llz-object-storage/main.tf:51-77`; there is no backup
  bucket, and the one bucket that ever existed for backup purposes,
  `gitea_backup`, was **removed**, `main.tf:79-87`),
- no restore procedure, tested or documented (see F-3),
- no gate. `tools/cmd/llz/ci.go:43-386` registers 27 `assert-*` verbs, and the
  e2e assert suite's full lane inventory is enumerated with per-lane rationale at
  `docs/workflows/llz-bootstrap-openbao.md:717-882` — loki, openbao-audit,
  scrape+reconciler, log-ingestion, eso-roundtrip, alert-delivery, grafana-dash,
  admission, net-enforcement, credentials, health-workflow, broad-pat, wave-vap,
  instance-custom, metric-surface, alert-eval. **None concerns snapshots, backup,
  or restore.**

**Blast radius.** A regional Linode Block Storage incident, a fat-fingered
`terraform destroy` on the wrong `--field region`, or the documented
destroy-time Volume sweep firing against the wrong cluster
(`docs/workflows/llz-terraform.md:553-575`) destroys every credential the
platform holds, with no recovery path other than re-bootstrapping from scratch
and re-minting every credential by hand.

**Recommendation.** Two pieces, in order:

1. **Build the snapshot.** A `CronJob` in `llz-openbao` (or a lane of the
   existing `llz-reconciler`, which already holds an OpenBao Kubernetes-auth
   token — `kubernetes-charts/llz-openbao-platform/values.yaml:135-141`) running
   `bao operator raft snapshot save`, writing to a new
   `<objLabelPrefix>-openbao-snapshots-<env>` bucket added to
   `terraform-modules/llz-object-storage/main.tf`. The snapshot is
   **seal-encrypted** — it is only readable with the static seal key — so
   storing it in object storage does not widen the credential blast radius
   beyond what F-4 already establishes. It does, however, make the seal key
   strictly *more* load-bearing, so ship it with F-4's gate.
2. **Name the gate** (AGENTS.md requirement; `docs/e2e-gates.md:169` — "only
   exists once something is running … an `llz ci assert-*` verb wired into the
   e2e lane battery"). The correct archetype here is the round trip, not a
   static check, for exactly the reason `docs/e2e-gates.md:21-39` gives about
   the audit pipeline: a CronJob that exists and a snapshot that lands are
   different facts. Propose **`llz ci assert-raft-snapshot`** — list the
   snapshot bucket, assert an object newer than N hours, and assert it parses
   as a raft snapshot archive (header check, not a full restore). Wire it into
   the e2e assert battery alongside `assert-obj-roundtrip`.

---

## F-2 (CRITICAL) — The obvious offline path for the raft data is foreclosed by an admission gate, for an unrelated reason.

The platform **actively rejects** clone/snapshot-sourced PVCs, at two layers:

- Kyverno `ClusterPolicy` `pvc-deny-untaggable-clone`,
  `validationFailureAction: Enforce`, denying any PVC with a `dataSource` or
  `dataSourceRef` — `tools/cmd/llz/manifests/kyverno-pvc-deny-untaggable-clone.yaml:39-84`.
- Its in-apiserver twin, a `ValidatingAdmissionPolicy` with
  `failurePolicy: Fail`, applied by `llz ci bootstrap-cluster` so it enforces
  **before Kyverno exists** —
  `tools/cmd/llz/manifests/vap-pvc-deny-untaggable-clone.yaml:32,40-46`;
  applied at `tools/cmd/llz/ci_bootstrap_cluster.go:294-295`.
- And it is **gated**: `llz ci assert-admission-enforcement` proves the deny is
  live, not merely present — `tools/cmd/llz/ci_assert_admission_enforcement.go:65,127`.

**The stated rationale has nothing to do with DR.** Verbatim, from the policy
header:

> "The Linode Block Storage CSI driver (v1.0.11) clones via the Linode
> CloneVolume API, which takes only (source id, label) — it does NOT accept tags,
> and the clone does NOT inherit the source Volume's tags. So a PVC created FROM
> a source … produces a Volume with NO `lke<id>` ownership tag … it dangles as an
> orphan after teardown"
> — `kyverno-pvc-deny-untaggable-clone.yaml:4-12`

and the load-bearing premise:

> "The platform uses NO clone/snapshot PVCs (grep: no dataSource/VolumeSnapshot
> in any chart or apl-values), so denying them has zero functional impact today"
> — `kyverno-pvc-deny-untaggable-clone.yaml:19-22`

**Why it matters.** That premise is true *only because no DR mechanism was ever
built.* The moment anyone attempts the most natural offline backup of a raft
store on this platform — VolumeSnapshot the data PVC, restore it into a rebuilt
cluster — they hit a hard deny at two layers, one of which cannot be turned off
without an apiserver-level change, and a green CI gate asserting the deny still
works. This is the difference between "nobody built it" and "a gate blocks it,"
and it materially changes the cost of closing F-1.

Note the policy's authors **anticipated** this and left the door labelled:

> "If clones are ever genuinely needed, relax BOTH denies together — the
> reconciler already provides the post-create tagging that makes that safe
> (modulo an up-to-an-hour untagged window, which reap's fail-safe keep-untagged
> default already tolerates)."
> — `kyverno-pvc-deny-untaggable-clone.yaml:29-32`

**Recommendation.** Do **not** relax the denies. The escape hatch is real but it
trades a clean, gated invariant for a one-hour untagged window and a
`volumeTagReconciler` dependency, to buy a snapshot mechanism that is inferior to
F-1's: a Volume-level snapshot of a live raft store is crash-consistent at best,
whereas `bao operator raft snapshot save` is application-consistent by
construction. **Build F-1 and leave the denies alone** — but record the coupling,
because the next person to reach for VolumeSnapshot-based DR will otherwise spend
a day discovering it from an admission error. Add a sentence to the policy header
naming OpenBao raft DR as the use case that is *not* a reason to relax it, and
pointing at the `raft snapshot save` path instead.

---

## F-3 (HIGH) — A runbook documents a restore path that has no implementation.

`docs/runbooks/bootstrap-openbao.md:207`:

> "Use this if the cluster is already initialized and unsealed but configuration
> steps were missed or need to be re-applied — for example **after a fresh cluster
> replacement with data restored from a snapshot**."

Per F-1, no snapshot mechanism exists anywhere in the repo. The runbook's
"Re-configure" mode is real and works (`:205-218`), but it is scoped to
re-applying *configuration* (auth methods, policies, seeds) onto an already-live
OpenBao. The clause promises a preceding step — restoring the data — that the
platform cannot perform.

**Why it matters.** A runbook is executed verbatim during an incident. An
operator reading `:207` during a real regional loss will look for the snapshot
restore step, not find it, and burn incident time discovering that the sentence
describes a capability the platform does not have. This is the same class as the
audit-pipeline regression (`docs/lessons-learned.md:112-120`): a document that is
internally consistent and describes something that does not exist.

**Recommendation.** Either delete the clause, or — better, once F-1 lands — make
it true and link the restore procedure. Until F-1 lands, replace the clause with
an explicit statement of the gap, so the runbook stops implying a path:
"*(There is no OpenBao data-restore path today — see the DR gap tracked in
`docs/adr/00NN`. A cluster replacement loses OpenBao's contents and requires a
first-time bootstrap plus re-seeding every credential.)*"

---

## F-4 — Seal/recovery key custody

### F-4a (HIGH, but a DOCUMENTED ACCEPTED TRADEOFF for the *rotation* half)

**The mechanism.** Auto-unseal is a `seal "static"` stanza reading a 32-byte
AES-256 key from a file
(`kubernetes-charts/llz-openbao-platform/values.yaml:293-296`), mounted from the
`openbao-unseal-key` Secret (`values.yaml:706-714`). The key lives only in etcd,
which LKE-E encrypts at rest (`docs/secrets.md:737-739`). There is no managed
KMS on Linode (`values.yaml:277-278`).

**The custody chain, in full:**

| Copy | Where | Evidence |
|---|---|---|
| Live | `llz-openbao/openbao-unseal-key` Secret in etcd | `values.yaml:711-714`; `tools/cmd/llz/ci_bao_seed_seal_key.go:33,208-218` |
| DR | `infra-<region>` GitHub **Environment** secret `OPENBAO_SEAL_KEY` | `ci_bao_seed_seal_key.go:198`; `docs/runbooks/bootstrap-openbao.md:55` |
| Offline | **manual operator action, unenforced** | `docs/runbooks/bootstrap-openbao.md:166-180` |

**The consequence, stated by the code itself:**

> "`current_key_id` is a PERMANENT identifier — OpenBao rejects a changed id for
> the same key material — and the key is never rotated: **losing it loses the
> data** (the recovery keys from `bao operator init` authorize generate-root but
> CANNOT decrypt the root key)"
> — `kubernetes-charts/llz-openbao-platform/values.yaml:288-292`

and enforced in code: an existing Secret is never overwritten
(`ci_bao_seed_seal_key.go:134-137`), a first-ever bootstrap **refuses to proceed**
without a secrets-write PAT so the key cannot exist in only one place
(`ci_bao_seed_seal_key.go:174-176`), and the key is never printed — only written
to the environment secret, which GitHub exposes by name only with no read-back
API (`docs/runbooks/bootstrap-openbao.md:170-180`).

**What is already accepted.** ADR 0012 (`docs/adr/0012-credential-observability-gaps.md`)
explicitly accepts the never-rotating property:

> "**It does not build rotation for the credentials that lack it.** The seal key,
> the recovery quorum and the Harbor robot copies are `static` because nothing
> rotates them … Rotating the seal key means a rewrap of OpenBao's entire raft
> store; that belongs in an ADR that argues it directly, and it is the honest
> residue of this one."
> — `docs/adr/0012-credential-observability-gaps.md:175-180`

ADR 0012 also brought `OPENBAO_SEAL_KEY` onto the credential single pane —
finding 2 at `:33` records that ADR 0009 had left it off "by omission, not
decision," and `:150-151` records the fix. `llz ci credential-coverage-guard`
now exists specifically so this class of omission cannot recur
(`tools/cmd/llz/ci_credential_coverage_guard.go:3-39`, error text at `:228-235`).

**So: the never-rotated seal key is a documented accepted tradeoff. Do not
report it as a novel gap.**

### F-4b (HIGH, NOT accepted anywhere) — nothing verifies the DR copy exists or matches.

What ADR 0012 accepted is that the key is not *rotated*. What no record accepts,
and no mechanism checks, is whether the **DR copy is present and correct**:

- The seed step restores from `OPENBAO_SEAL_KEY` only if the env var is set at
  that moment (`ci_bao_seed_seal_key.go:158-169`) — it never *verifies*, on a
  normal run, that the environment secret still holds the same key the live
  Secret holds.
- The idempotent path (`ci_bao_seed_seal_key.go:134-137`) short-circuits before
  `resolveSealKey` is ever called, so on **every run after the first**, the
  `infra-<region>::OPENBAO_SEAL_KEY` secret is not read, not compared, and not
  validated. Someone deleting or overwriting that environment secret produces
  **no signal anywhere**, until the day a namespace rebuild needs it.
- The offline copy is a checkbox in prose
  (`docs/runbooks/bootstrap-openbao.md:166-181`) with no gate.
- The runbook's own recovery instruction for the persistently-sealed case is to
  "check … that it matches `OPENBAO_SEAL_KEY` on the `infra-<env>` environment"
  (`:201`) — a comparison the tooling could make continuously and does not.
- The alert set covers `OpenBaoSealed`, `NoActiveLeader`, `RaftQuorumDegraded`
  (`platform-apl/components/observability/prometheus-rules/openbao-alerts.yaml:32,42,57`)
  — all of which fire *after* the key is already a problem. None is predictive.

**Why it matters.** This is the highest-consequence single value in the platform,
its DR copy lives in a store (GitHub Environment secrets) that is write-only from
the UI, mutable by anyone with Environment admin, and it is checked exactly once
in the life of the instance. The failure is perfectly silent until the moment it
is fatal — the exact shape `docs/lessons-learned.md:130-133` names as the reason
gates exist.

**Recommendation — name the gate.** Add **`llz ci assert-seal-key-escrow`**:
read `llz-openbao/openbao-unseal-key` from the cluster, read
`infra-<env>::OPENBAO_SEAL_KEY` (presence is checkable via
`repos/{repo}/environments/{env}/secrets/{name}` — the same 404-means-absent
probe `tools/cmd/llz/state_passphrase.go:63-72` already implements), and fail
when the environment secret is absent. Comparing *values* is not possible
(GitHub has no read-back), so assert **presence** and additionally assert the
live Secret is exactly 32 bytes and decodes — the two failure modes
`resolveSealKey` guards against at seed time
(`ci_bao_seed_seal_key.go:161-166`) but nothing re-checks afterwards. Wire it
into the scheduled-checks lane, not only e2e, so it runs against live instances.

### F-4c (MEDIUM) — break-glass is well built, and untested by any live gate.

The break-glass path is genuinely good: `breakglass-openbao.yml` →
`llz ci bao-breakglass` regenerates a root token from the recovery quorum and
returns it **encrypted to the operator's RSA public key**, never in cleartext in
a run log (`docs/runbooks/bootstrap-openbao.md:281-291`), with `generate` /
`rotate` / `revoke` actions (`:296-302`) and a shared concurrency group with
bootstrap (`:357-358`). Manual break-glass verbs (`bao-status`, `bao-init`,
`bao-regen-root`) are documented as deliberately caller-less
(`:249-277`).

But it is unit-tested only, and the maintainer rationale says so in its own
words: "The whole break-glass flow — validate, revoke/rotate/regenerate, and
RSA-OAEP-encrypt — is **one unit-tested Go verb**"
(`docs/workflows/llz-breakglass-openbao.md:8-10`), with the encryption path
specifically called out as unit-tested "rather than an untested inline `openssl
pkeyutl` heredoc" (`:93-97`). The test is
`tools/cmd/llz/ci_bao_breakglass_test.go`. The e2e assert-suite lane inventory
(`docs/workflows/llz-bootstrap-openbao.md:717-882`, read in full) contains no
lane that exercises quorum regeneration against a live cluster, and none that
exercises the **seal-key restore** path. Confirmed against the workflows:
`OPENBAO_SEAL_KEY` appears in
`instance-template/.github/workflows/llz-bootstrap-openbao.yml:19,21,86,345` only
as an input to `llz ci bao-seed-seal-key` on the normal bootstrap path — the
namespace-rebuild restore branch (`ci_bao_seed_seal_key.go:158-169`) is never
driven by any workflow, and the maintainer notes describe that step purely as the
first-bootstrap/idempotent path
(`docs/workflows/llz-bootstrap-openbao.md:361-373`).

Note the recovery quorum also **cannot** substitute for the seal key. Three
independent statements agree: `docs/runbooks/bootstrap-openbao.md:353-356`
("No `OPENBAO_SEAL_KEY` is needed (that is only for a namespace/data rebuild)"),
`docs/workflows/llz-breakglass-openbao.md:163` (same, verbatim), and
`kubernetes-charts/llz-openbao-platform/values.yaml:290-292` (the recovery keys
cannot decrypt the root key). So break-glass restores *administrative access to a
live cluster*; it restores no data and unseals nothing.

**One further gap this document surfaces, flagged for the security reviewer as
well.** The break-glass rationale states a hard prerequisite in its own
`⚠️ Required prerequisite` section: "**every `infra-*` GitHub Environment MUST
have protection rules — at minimum required reviewers** … Without them, any
identity that can trigger a `workflow_dispatch` (a leaked write-scope PAT, a
compromised account) can mint cluster root with a clean, quiet exfiltration path"
(`docs/workflows/llz-breakglass-openbao.md:39-54`). The document is explicit that
the summary's `github.actor` record "is a *record*, not a *control*; the
environment reviewers are the control" (`:52-54`). **Nothing enforces or checks
that protection rules exist.** It is DR-relevant because `infra-<env>` is also
where the seal key and the entire usable recovery quorum live (F-14) — the same
environment whose protection is unverified. GitHub exposes environment protection
rules on `repos/{repo}/environments/{env}`, so this is checkable by the same probe
family F-4b proposes.

**Recommendation.** Extend the existing `llz-wedge-gameday` pattern
(`tools/cmd/llz/ci_wedge_gameday.go`, `docs/workflows/llz-wedge-gameday.md`) —
which already injects a controlled fault and asserts containment — with a
**seal-key restore rehearsal**: delete the `openbao-unseal-key` Secret, delete
the OpenBao pods, re-run `llz ci bao-seed-seal-key --region <env>`, assert all
three pods return to unsealed. This is the one DR path that is fully exercisable
on the e2e cluster without destroying it, and it is the path a real incident will
take.

---

## F-5 (MEDIUM/HIGH) — Object storage buckets have no versioning, no lifecycle, no replication, and no deletion protection.

**Evidence — the module, read in full.**
`terraform-modules/llz-object-storage/main.tf` declares exactly four buckets:
`harbor_registry` (`:51-55`), `loki_chunks` (`:61-65`), `loki_ruler` (`:67-71`),
`loki_admin` (`:73-77`). Each is three attributes — `region`, `s3_endpoint`,
`label`. There is nothing else on them.

**Evidence — the absence, grepped:**

```
grep -rn 'prevent_destroy|lifecycle|versioning|force_destroy|replication' \
  terraform-modules/ tools/internal/tfroots/
```

Every hit is either prose in a comment
(`llz-object-storage/main.tf:13,22,91`; `llz-object-storage/README.md:24`) or
belongs to the cluster module's `ignore_changes`
(`terraform-modules/llz-cluster/main.tf:92-93`,
`terraform-modules/llz-cluster/firewall.tf:101`). **No `prevent_destroy` on any
resource in the repo. No bucket versioning anywhere. No cross-region
replication.** A repo-wide `grep -rni versioning` over `tools/`,
`terraform-modules/` and `docs/` returns only unrelated hits (GitHub REST API
headers, `terraform-modules/RELEASING.md`, `docs/agents.md`,
`docs/designs/instance-slimming.md`).

**This extends to the Terraform state bucket itself.** It is created by the
wizard via a bare Linode API call — `client.CreateObjectStorageBucket(ctx,
clusterID, bucketName)`, `tools/cmd/llz/tokens.go:164` — with no versioning
configuration on that call or anywhere after it. The state backend is a plain S3
backend with no versioning or locking configuration
(`tools/internal/tfroots/roots/object-storage/backend.tf:17-24`), and
`instance-template/terraform-iac-bootstrap/AGENTS.md:113` states plainly:
"State corruption is not recoverable without a backup." **There is no backup.**

**The `objLabelPrefix` destroy-then-create trap is already documented** —
`docs/landing-zone-spec.md:252-258` is explicit that the module "declares no
`create_before_destroy`, so Terraform plans **destroy-then-create** on all four
buckets," that a non-empty bucket fails the apply, and that an empty one "deletes
them and their data with them." That prose is accurate and I confirmed it against
the module (no `lifecycle` block on any bucket resource, `main.tf:51-77`). It is
documented but **not gated** — nothing prevents an `objLabelPrefix` edit from
reaching an apply.

**Why it matters.** Loki chunks are the platform's log retention (and the
destination of OpenBao's audit stream, `values.yaml:769`); the Harbor registry
bucket is every platform image. With no versioning, an errant `s3 rm --recursive`
— which is exactly what the destroy path runs
(`terraform-modules/llz-object-storage/main.tf:17-20`, and the temp-key drain at
`:109-111`) — is unrecoverable. With no `prevent_destroy`, a spec edit plans a
destroy on all four.

**Recommendation.**

1. **`prevent_destroy` on the four buckets** is the cheapest real control, and it
   converts the documented `objLabelPrefix` trap from prose into a hard plan-time
   failure. It does conflict with the destroy workflow, which is why the correct
   form is a module variable (`protect_buckets`, default `true`) that the
   `destroy-object-storage` job sets false — the same shape as the existing
   `assert-destroy-confirm` guard (`tools/cmd/llz/ci.go:80`).
2. **Enable versioning** on the state bucket at creation
   (`tools/cmd/llz/tokens.go:164`) and on the four data buckets — **conditional
   on Linode Object Storage and the Terraform provider actually supporting it,
   which I have not verified and am not asserting.** This module carries the
   precedent for exactly that caution: `force_destroy` was added here and then
   removed because "the Linode provider's `linode_object_storage_bucket` resource
   does NOT support that argument (validated against the deployed provider
   version)" (`terraform-modules/llz-object-storage/main.tf:13-16`, and the same
   scar in `terraform-modules/llz-object-storage/README.md:24`). So the first
   step is to establish, against the deployed provider and the S3-compatible API,
   whether bucket versioning is available at all and by which route (a provider
   attribute, or an out-of-band `PutBucketVersioning` call on the S3 endpoint the
   module already computes at `main.tf:46-49`). If it is available, this is the
   single highest-leverage change in this section — it makes state corruption and
   an errant drain recoverable. If it is not, that unavailability is itself a
   finding and belongs in the DR ADR as an accepted residual, in the same form
   this module already used for `force_destroy`.
3. **Gate whatever the answer is**: `llz ci assert-obj-versioning`, alongside the
   existing `assert-obj-encryption` (`tools/cmd/llz/ci.go:165`) which already
   proves a per-bucket property live. Same lane, same shape, same call pattern.
4. Cross-region replication is **not** recommended — Linode Object Storage
   buckets are pinned to a single endpoint
   (`terraform-modules/llz-object-storage/main.tf:30-45`), so replication would
   be an application-level copy job. Versioning gets most of the value for a
   fraction of the complexity.

---

## F-6 (HIGH) — The cross-region "HA pair" has an unbounded, unmeasured RPO.

**What the HA pair actually is.** `spec.cluster.ha.{role,group}` with the
validator enforcing exactly one `active` and one `standby` per group
(`docs/landing-zone-spec.md:168-190`). But the two clusters run **completely
independent** OpenBao instances:

> "Per-region cluster — the secondary region runs its own independent OpenBao
> cluster. Cross-region consistency is achieved by operator-side dual-write …
> not by OpenBao replication. OpenBao OSS has no Performance Replication
> equivalent — that is a Vault Enterprise feature, intentionally not in OpenBao."
> — `kubernetes-charts/llz-openbao-platform/values.yaml:212-217`

The choice is deliberate and recorded: `docs/secrets.md:103-109` rejects a
stretched Raft cluster and chooses "Two independent HA clusters + operator-side
dual-write — near-zero-write workload makes this trivial." That is a sound
architectural decision and I am not contesting it.

**The gap is that nothing measures whether the dual-write held.**

- `llz openbao set` writes both, compares SHA-256 of the post-write payload, and
  rolls back the primary if the secondary fails (`docs/secrets.md:485-488`). Good
  — but that is a **per-write** check, in-band, at the moment of writing.
- Anything that writes to only one cluster silently diverges. That set is not
  small: the in-cluster `harbor-robot-provisioner`, `linodeCredRotator` /
  `llz ci rotate-linode-creds`, `broad-pat-rotator`, and ESO PushSecrets all
  write into **their own cluster's** OpenBao
  (`docs/runbooks/bootstrap-openbao.md:147,153-154`;
  `kubernetes-charts/llz-openbao-platform/values.yaml:134-160`). The standby's
  Harbor credentials are replicated only at **bootstrap time**, by a one-shot
  workflow step (`docs/runbooks/bootstrap-openbao.md:147,235-239`).
- `docs/secrets.md:601` instructs the operator, after a primary outage, to "run a
  drift check and, if needed, re-apply the last-written values." **There is no
  drift-check command.** Grepping `docs/` and `tools/` for
  `drift check|drift-check|openbao diff|bao-diff|compare.*regions` finds only:
  the same prose at `docs/secrets.md:191-192,601`; template/chart-pin drift
  checks (`tools/cmd/llz/drift.go`, `ci_chart_pin_guard.go:153`) which are
  unrelated; and one **manual** two-line recipe buried in a migration runbook:

  ```
  diff <(llz openbao get active   secret/<project>/keys <app_secret>) \
       <(llz openbao get standby secret/<project>/keys <app_secret>) && echo "in sync"
  ```
  — `docs/apl-core-migration-runbook.md:188-193`

  That checks **one key**, by hand, and lives in a migration runbook nobody reads
  during an incident.
- `docs/secrets.md:603-605` states outright: "This template intentionally does
  not support 'write to secondary only during primary outage' — that would create
  drift the moment primary returns, and **there is no automated reconciliation**."

**So the effective cross-region RPO is: whenever an operator last ran
`llz openbao set` for that particular path — unbounded, unmeasured, and
per-path.** A standby promoted after a primary loss will be missing every
credential written in-cluster since its bootstrap: rotated object-storage keys,
rotated Harbor robots, rotated Linode PATs.

**And the entire HA path is never exercised by CI.** This is the part that makes
the finding urgent rather than theoretical. The maintainer notes state it
directly, while explaining a behaviour change: "That HA path is not exercised by
release-e2e (standalone); deferring peer-CA provisioning until a clean bootstrap
is the intended trade" (`docs/workflows/llz-bootstrap-openbao.md:640-642`).
release-e2e provisions a **standalone** deployment, so every standby-specific
mechanism ships untested against a live cluster: the `resolve` job's role/peer
derivation (`:144-157`), the "Seed standby Harbor robot credentials" step
(`:525-532`), "Extract standby CA cert" (`:572-587`), and all three peer-CA jobs
— `provision-peer-ca` (`:936-947`), `fetch-standby-ca` (`:951-968`), and
`reprovision-peer-ca` (`:972-976`). The `fetch-standby-ca` job exists
specifically "to recover from a failed `provision-peer-ca` without restarting the
entire standby bootstrap" (`:951-957`) — a recovery path for a path that CI never
runs.

**Why it matters.** The standby exists specifically to be promoted during a
regional loss. Its readiness for that role is currently unknown at all times —
neither its secret contents (no parity check) nor its bring-up machinery (no e2e
coverage) is verified — and the failure mode on promotion is a set of credentials
that are silently stale, which reads from the cluster exactly like a working
cluster with an authentication problem.

**Recommendation — this is the strongest gate-shaped finding in the review.**
Add **`llz ci assert-openbao-parity`**: enumerate the KV v2 paths listed in
`credPaths` (the same list `assert-rotation-health` already derives from,
`tools/cmd/llz/ci.go:297`), read each from both clusters via the existing
`llz openbao get active|standby` primitive, and compare the metadata **version
and `rotated_at` stamp** — not the values, which the gate must never handle. Fail
on any path that exists in one cluster and not the other, or whose stamps differ
by more than a configured window. Report **per path**, in the style
`docs/e2e-gates.md:244-247` requires ("Name what IS present, not only what is
missing"). This is exactly the round-trip archetype `docs/e2e-gates.md:99-100`
prescribes, applied one subsystem over, and it converts an unmeasured RPO into a
measured one. Run it on the scheduled-checks lane, not just e2e — the divergence
accrues on live instances, not on the harness.

---

## F-7 (HIGH) — `TF_STATE_ENCRYPTION_PASSPHRASE` is a second irrecoverable-loss key with the same custody story.

Every Terraform root carries an OpenTofu `encryption` block (ADR 0007;
`tools/internal/tfroots/roots/{cluster,vpc,databases,object-storage}/encryption.tf`),
keyed by `pbkdf2` from a single GitHub secret. The code says the quiet part
plainly:

> "Lose it and every state file is permanently unreadable — **same blast radius as
> OPENBAO_SEAL_KEY**."
> — `tools/cmd/llz/state_passphrase.go:278-279`

and ADR 0007 records the same: "Key escrow is now load-bearing. Losing
`TF_STATE_ENCRYPTION_PASSPHRASE` makes every state file unrecoverable — the same
class of risk as [the seal key]" (`docs/adr/0007-terraform-state-encryption.md:123-125`).

**The defences that exist are genuinely careful** and worth naming, because they
are the model F-4b should follow: `statePassphraseExists` probes **both** repo
and every `infra-*` environment scope and treats an indefinite answer as unknown
rather than absent (`state_passphrase.go:74-124`); `dropStatePassphraseIfLive`
re-asks immediately before any push and **refuses** rather than risk a clobber
(`:165-193`); the escrow banner is printed once, to stderr, undimmed, and
explicitly says the local cache "is not escrow" (`:275-282`).

**The gap.** As with F-4b, nothing verifies the passphrase is still escrowed
after run zero, and there is no rotation-with-fallback drill. The code references
`secret-rotation.yml (scope: state-passphrase)` (`:184`, `:212-216`) which
re-keys every root and then deletes `TF_STATE_ENCRYPTION_PASSPHRASE_OLD` — a
correct design, but one whose failure mode is "every state file stranded with no
fallback left to read them" (`:216-217`), and no gate proves a rotation completed
across **all** roots before the OLD value is dropped.

**This exclusion IS documented, outside the ADR set — label it accordingly.** The
e2e `credentials` lane rationale explains why no gate forces a rotation of it:

> "forcing lke-admin (deletes the kubeconfig the job is using), obj-key (cuts TF
> state access), db-admin (Linode resets in place with no overlap) or **the state
> passphrase (a near one-way door)** would break the cluster being measured."
> — `docs/workflows/llz-bootstrap-openbao.md:836-838`

That is a considered, correct decision about **rotation rehearsal** and should not
be reported as an oversight. What it does not cover — and what nothing covers —
is **escrow verification**, which is non-destructive by construction (a presence
probe mutates nothing) and therefore carries none of the reasoning above.

**Compounding effect on Q4 (cluster rebuild).** Every recovery action starts by
fetching a kubeconfig **from Terraform state**:
`docs/runbooks/bootstrap-openbao.md:137` — "Retrieves kubeconfig from the
Terraform S3 state (`cluster/<env>/terraform.tfstate`)". So losing the state
passphrase does not merely lose the ability to `terraform apply`; it locks the
operator out of the recovery path itself.

**The mitigation already exists and is well documented** —
`docs/runbooks/bootstrap-openbao.md:267-270` calls out `llz ci fetch-kubeconfig`
as the Linode-API kubeconfig fetch that "needs no `terraform init`, no S3 backend
and no git auth — the things most likely to be broken when you need a kubeconfig
by hand." That is the right control and it is correctly framed. It should be
named in the DR record (F-13) as the state-independent entry point.

**Recommendation.** Extend F-4b's `assert-seal-key-escrow` to cover
`TF_STATE_ENCRYPTION_PASSPHRASE` — the presence probe is literally already
written (`state_passphrase.go:63-72,91-124`), so this is a re-use, not a new
mechanism. Name it `llz ci assert-escrow-present` covering both keys.

---

## F-8 (MEDIUM) — No RTO or RPO is stated anywhere. Stated affirmatively, having grepped.

```
grep -rni 'RTO|RPO|recovery time objective|recovery point objective|disaster recovery|\bDR\b' \
  docs/ tools/ kubernetes-charts/ instance-template/ terraform-modules/ platform-apl/ \
  README.md AGENTS.md Makefile
```

Every hit for "disaster recovery" / "DR" refers to the **seal-key escrow
mechanism** and nothing else: `docs/secrets.md:125,743`;
`docs/runbooks/bootstrap-openbao.md:55,139`;
`docs/workflows/llz-bootstrap-openbao.md:370-371`;
`tools/cmd/llz/ci_bao_seed_seal_key.go:19,110,175`. The single hit for "DR
rehearsal" is an aside in an unrelated design doc about a feature that is built
but not deployed (`docs/designs/obj-sse-c-gateway.md:314`,
status confirmed at `docs/designs/README.md:53`). **`RTO`, `RPO`, "recovery time
objective" and "recovery point objective" return zero hits across the entire
repository.**

**Why it matters.** Every finding above is a design decision that would be
trivially settled by a stated objective. "No raft snapshot" is a defensible
choice under an RPO of "re-bootstrap and re-seed" and indefensible under an RPO
of one hour — but the choice was never framed, so it was never made. The same
applies to F-5 (bucket versioning) and F-6 (cross-region parity). The repo has a
strong convention of recording exactly this kind of decision (ADR 0009's "honest
residue," ADR 0012's "does not do" list) and it has not been applied to DR.

**Recommendation.** State an RTO/RPO per data class — OpenBao secrets,
Terraform state, Loki chunks, Harbor registry, Postgres — in the DR ADR proposed
in F-13. Two columns and five rows would settle six of the findings in this
document.

---

## F-9 (MEDIUM) — Shared managed Postgres: the module configures no backup posture, and nothing asserts one.

`terraform-modules/llz-databases/main.tf` provisions
`linode_database_postgresql_v2` with: `label`, `engine_id`, `region`, `type`,
`cluster_size`, `private_network{vpc_id,subnet_id,public_access}`, and
`updates{}` — `main.tf:26-59`. **There is no backup, retention, PITR, or fork
configuration in the resource, and no variable exposing one** (`variables.tf`,
read: `name`, `region_suffix`, `region`, `engine_version`, `db_type`,
`cluster_size`, `vpc_id`, `subnet_id`, `public_access`, `label_prefix`,
`maintenance`).

`cluster_size` defaults to `2` — "2 or 3 = high availability with standbys"
(`variables.tf:44-45`). That is **availability**, not backup: a standby replica
protects against node loss, not against a dropped table, a bad migration, or a
deleted cluster.

**`docs/designs/shared-managed-postgres.md` (366 lines) was read in full, and it
is a thorough document** — it covers the 0-n cluster shape and why the map key is
identity in three places (`:112-129`), four Aiven-platform behaviours that cost a
downstream team a debugging cycle each (`:157-192`), a detailed migration plan
(`:194-240`), and an unusually careful admin-credential rotation design
(`:291-351`). **It contains no treatment of backup, PITR, restore, or fork.** The
only occurrences of "restore" are `pg_restore` inside the one-off migration
recipe (`:213-216`); the only "recovery" is absent entirely. So the finding is
not "the authors were careless" — it is that a document which thinks carefully
about every other lifecycle property is silent on this one.

**I am deliberately not asserting what Linode Managed Databases do provider-side.**
That is not knowable from this repository, and stating it from memory is exactly
the failure this review is meant to avoid. What *is* knowable and is the finding:
**the platform makes no statement about it, configures nothing, and verifies
nothing.** `llz ci assert-database` exists (`tools/cmd/llz/ci.go:313`) but is a
connectivity/health lane, not a backup assertion.

**Three adjacent facts that sharpen it.**

1. **The admin credential is the documented escape hatch into a cluster, and the
   design protects it deliberately.** `llz ci seed-db-admin` "never prunes":
   removing a cluster from the spec leaves its OpenBao path, because "the admin
   credential is the only way back into it for a final `pg_dump` — so deleting it
   automatically would destroy the escape hatch at exactly the wrong time"
   (`:275-279`). That is a genuine, well-reasoned DR control and belongs in F-12's
   preserve list. It also concedes the point: **`pg_dump` is the platform's actual
   recovery story for Postgres**, and it is manual.
2. **Rotation is irreversible, has no overlap window, and is "Not yet proven
   live."** Linode "exposes exactly one mutation … which regenerates the password
   in place. There is no second credential, no overlap window" (`:301-306`), so the
   invariant flips to "never lose the new credential" (`:309`). The design is
   careful — sequential across clusters, stops at first failure, waits for
   `active`, treats an unchanged password as failure (`:311-323`) — but
   `:360-366` states plainly: "no cluster has been rotated for real." A DR event
   that requires re-seeding admin credentials would be the first live exercise of
   this path.
3. **A documentation inconsistency in the OpenBao path, worth a one-line fix.**
   The module comment says the admin credentials are seeded to
   `secret/platform/db-admin/<name>` (`terraform-modules/llz-databases/main.tf:6`),
   while the design document says `secret/infra/db-admin/<name>` in five places
   (`:15, :119, :219, :261, :293`). One of them is wrong. This matters here
   because an operator recovering a database under time pressure will read one of
   these two and look for a path that does not exist.

**Recommendation.** Three steps. (1) Establish the provider-side default from
Linode's own documentation and record it in
`docs/designs/shared-managed-postgres.md` — whether automatic backups exist,
their retention, and whether PITR/fork is available. (2) Whatever that answer is,
surface the knobs the provider exposes as module variables so the posture is
declared rather than inherited, and extend `assert-database` to assert the
declared posture is live. If the provider exposes no knobs, that is itself the
finding and belongs in the DR ADR as an accepted residual — the form ADR 0009
and 0012 already use for exactly this situation. (3) Reconcile the
`secret/platform` vs `secret/infra` path discrepancy; whichever is correct, the
other is a trap in a recovery path.

---

## F-10 (MEDIUM) — The cluster-rebuild path exists and is documented; the *state re-attachment* half is not, and no rehearsal exercises it.

**What is documented and works.** A full rebuild is a single dispatch:

```
gh workflow run terraform.yml --field region=<env> --field action=apply --field module=all
```

walking vpc → cluster → object-storage → in-cluster bootstrap → OpenBao
(`docs/runbooks/bootstrap-openbao.md:105-121`), described as "the supported path
on a fresh-cluster rebuild" (`:121`). The teardown side is unusually thorough:
Volume and NodeBalancer sweeps scoped by captured id, a VPC delete ordered last
to avoid the 409, and a hard job failure if orphans remain
(`docs/workflows/llz-terraform.md:522-575`). Ordering constraints are stated
(active before standby, `:235-239`), and known wedges have their own runbooks
(`docs/runbooks/first-build-failed.md`, `apl-branch-recreate-wedge.md`,
`e2e-lane-diagnostics.md`).

**What is missing.** The rebuild recreates *infrastructure*. What is
irrecoverably lost, and where the documentation goes quiet:

| Asset | On rebuild | Evidence |
|---|---|---|
| OpenBao **data** (all secrets) | **LOST** — no snapshot to restore | F-1 |
| OpenBao **seal key** | Recoverable *iff* `infra-<env>::OPENBAO_SEAL_KEY` survived — but it only helps if the raft Volumes also survived | `ci_bao_seed_seal_key.go:158-169` |
| Terraform **state** | Recoverable *iff* the passphrase survived; state bucket unversioned | F-7, F-5 |
| Loki chunks / Harbor images | Buckets are separate roots and survive a cluster destroy — but unversioned, and the destroy path drains them | F-5; `terraform-modules/llz-object-storage/main.tf:17-20` |
| Block Storage Volumes | `Retain` protects them from CSI, but the destroy job's sweep **deletes them by design** | `block-storage-class.yaml:79-84`; `docs/workflows/llz-terraform.md:553-575` |
| Harbor robot credentials | Re-minted by the in-cluster provisioner within ~5 min | `docs/runbooks/bootstrap-openbao.md:147` |
| `openbao-tls`, client CAs | Re-issued by cert-manager from a stable self-signed CA | `docs/runbooks/bootstrap-openbao.md:197-199` |

**The honest answer to "how long would a rebuild take" is not stated anywhere**
and cannot be derived from the repo. The one adjacent datum is in this project's
session memory rather than in the repository: the e2e Provision job is a
~40-minute bottleneck. That is a floor for infrastructure, and it excludes the
manual re-seeding of every credential that F-1 makes necessary.

**Recommendation.** Add a **`docs/runbooks/cluster-rebuild.md`** whose spine is
the table above — what survives, what does not, and in what order to restore.
Two things make it worth writing even before F-1 lands: it is the document an
operator opens at hour zero of a regional incident, and writing it forces the
RTO/RPO statement F-8 is missing. Gate it the way the repo gates other live
behavior: extend `llz-wedge-gameday` with the seal-key restore rehearsal from
F-4c, which is the one step of this runbook that is safely rehearsable.

---

## F-11 (MEDIUM) — The OpenBao audit log is on an `emptyDir`, and it is the only forensic record of a DR event.

`auditStorage.enabled: false`, with a 2Gi `emptyDir` named `audit`
(`kubernetes-charts/llz-openbao-platform/values.yaml:459-466, 654-657`). The
rationale is sound and explicit — the Promtail sidecar ships to Loki "within
seconds; the local file is only a buffer," saving 30GB across the raft
(`:459-463`) — and the risk is already named: "OpenBao audit devices are
blocking — if the emptyDir fills, OpenBao stops serving. Monitor
`kubelet_volume_stats`, alert at 75%."

**Two gaps against that stated control.**

1. **The alert prescribed in the comment does not exist.** The OpenBao rule group
   (`platform-apl/components/observability/prometheus-rules/openbao-alerts.yaml`)
   contains `OpenBaoMetricsTargetDown` (`:22`), `OpenBaoSealed` (`:32`),
   `OpenBaoNoActiveLeader` (`:42`), `OpenBaoRaftQuorumDegraded` (`:57`),
   `OpenBaoLeaseExhaustion` (`:76-81`), `OpenBaoAuditLogFailure` (`:90-94`).
   There is **no `kubelet_volume_stats` alert at 75%**. `OpenBaoAuditLogFailure`
   fires on `vault_audit_log_request_failure` — i.e. *after* writes are already
   failing, which for a blocking audit device means OpenBao has already stopped
   serving. The predictive alert the chart comment specifies is absent.
   (Note also that `kubelet_volume_stats` is a PVC-scoped metric series; an
   `emptyDir` may not be covered by it at all — which, if true, means the
   prescribed control is not merely unimplemented but unimplementable as
   written, and needs `container_fs_usage_bytes` or an equivalent instead. Worth
   confirming against the live cluster before writing the rule.)
2. **DR relevance.** During the exact incident this review is about — a wedged or
   partially-lost OpenBao — the audit log is the record of what was read, by
   whom, and when. It lives in a pod-local buffer that is destroyed with the pod,
   and its only surviving copy is in Loki, in the same cluster.

**Recommendation.** Implement the alert the chart comment already commits to, in
the existing `openbao-alerts.yaml` group, after confirming which metric actually
covers an `emptyDir` on this cluster. This is a two-line rule against a documented
requirement — the cheapest closable item in this review.

---

## F-14 (HIGH) — The "3-of-5 recovery quorum" is not distributed. All three usable shares live in one place, alongside the seal key.

`bao operator init -recovery-shares=5 -recovery-threshold=3` implies a split-custody
control: five shares, three needed, no single holder can act alone. **On this
platform that property does not hold**, and two documents say so plainly.

`docs/playbooks/openbao-accounts.md:73-77`, under the heading "*only if you
personally hold 3 of the 5 recovery keys*":

> "The recovery keys are printed **once** to the first bootstrap's job summary and
> are **not distributed to operators**. If you do not have three of them in hand,
> use the workflow above instead — **this path cannot be completed**."

and `docs/runbooks/openbao-team-login.md:68-75`:

> "you almost certainly DON'T hold the recovery keys … they're stored in the
> `infra-<region>` GitHub environment (`OPENBAO_RECOVERY_KEY_1..3`) and printed
> once to the bootstrap job summary. So unless you personally kept 3 of the 5 keys
> offline, use the break-glass workflow, which reconstitutes root from that stored
> quorum with **no operator-held keys**."

**So the effective custody model is:**

| Share | Where it lives |
|---|---|
| 1, 2, 3 (= the threshold) | `infra-<env>` GitHub Environment secrets — **one store** (`docs/runbooks/bootstrap-openbao.md:56-58`; `docs/workflows/llz-bootstrap-openbao.md:72,99`) |
| 4, 5 | Printed once to a job summary; offline copy is a manual, unenforced operator step (`docs/runbooks/bootstrap-openbao.md:181`) |
| `OPENBAO_SEAL_KEY` | **The same `infra-<env>` environment** (F-4a) |

**Why it matters.** Three independent consequences, none of them recorded
anywhere:

1. **The quorum has one custodian, not five.** Anything that can read
   `infra-<env>` secrets holds the full threshold and can mint cluster root. That
   is precisely why the break-glass doc demands required-reviewer protection on
   those environments (F-4c) — but the control is unenforced, so the quorum's only
   real boundary is a setting nobody checks.
2. **The seal key and the quorum share a single loss boundary.** Lose the GitHub
   repository or organisation — account compromise, an org migration, a repo
   deletion — and you lose the seal key (so the raft data is unreadable, F-4a) and
   the recovery quorum (so root cannot be regenerated) **in the same event**. The
   two controls that look like defence in depth are stored in one place. Note also
   that neither is recoverable *from* the other: `values.yaml:290-292` and
   `docs/workflows/llz-breakglass-openbao.md:163` between them establish that the
   quorum cannot decrypt the root key and the seal key is not needed for
   break-glass — they are strictly disjoint capabilities.
3. **Shares 4 and 5 are probably already gone on any live instance.** They exist
   only if an operator manually copied them out of a GitHub Actions job summary at
   first bootstrap. Actions logs and summaries have finite retention, so on a
   cluster bootstrapped some time ago there is realistically no path to those two
   shares — which means the "5 shares" are, in practice, the 3 in the environment.

**Recommendation.** Do not "fix" this by scattering keys — the current design is a
reasonable automation tradeoff and the break-glass workflow is built on it. Record
it and bound it:

- **State it in the DR ADR (F-13)** as an explicit accepted tradeoff: the recovery
  quorum is single-custodian by design so break-glass can be automated, and the
  compensating control is environment protection rules. Right now a reader of
  `bao operator init -recovery-shares=5 -recovery-threshold=3` will reasonably
  infer split custody that does not exist.
- **Make the compensating control checkable.** Fold an environment-protection
  assertion into the `assert-escrow-present` gate proposed in F-4b/F-7 — same
  probe family, same lane. That converts the break-glass doc's `⚠️ Required
  prerequisite` from prose into a verdict.
- **Consider separating the seal key's store from the quorum's.** Even moving the
  offline seal-key escrow to a different custodian than the GitHub org breaks the
  shared loss boundary in (2). This is a policy change, not a code change, and it
  is the cheapest reduction in correlated risk available anywhere in this review.

---

## F-12 (INFORMATIONAL) — Controls that are correct and should be preserved

Recorded so a future change does not remove them believing they are incidental:

- `reclaimPolicy: Retain` on `block-storage-retain` — today the *only* thing
  protecting OpenBao's raft data from an accidental PVC delete
  (`tools/cmd/llz/manifests/block-storage-class.yaml:135`, rationale `:79-84`).
- `bao-seed-seal-key`'s refusal to proceed without a secrets-write PAT on a
  first-ever bootstrap, precisely so the key cannot exist in only one place
  (`tools/cmd/llz/ci_bao_seed_seal_key.go:173-176`).
- Its never-overwrite idempotency (`:134-137`).
- The break-glass token being returned **encrypted to the operator's key**, never
  in a run log (`docs/runbooks/bootstrap-openbao.md:285-289`).
- `llz ci fetch-kubeconfig` as the state-independent, S3-independent,
  git-independent kubeconfig path (`docs/runbooks/bootstrap-openbao.md:267-270`)
  — and the note at `:275-277` that break-glass verbs reporting zero callers is
  expected, not dead code. **Do not let a dead-code sweep retire these.**
- `state_passphrase.go`'s definite-answer-only probe and pre-push clobber guard
  (`:63-72, 165-193`) — the pattern F-4b should copy.
- `llz ci credential-coverage-guard` (`tools/cmd/llz/ci_credential_coverage_guard.go:3-39`),
  which exists specifically so a credential can never again be silently
  unmeasured — the mechanism that would have caught F-4b's class had it covered
  escrow as well as measurement.
- **`llz ci seed-db-admin` never prunes a removed cluster's OpenBao path**
  (`docs/designs/shared-managed-postgres.md:275-279`) — deliberately, because "the
  admin credential is the only way back into it for a final `pg_dump`." A cleanup
  pass that "tidies orphaned OpenBao paths" would delete the Postgres escape
  hatch at exactly the wrong moment.
- **`llz ci rotate-db-admin` is dispatch-only and excluded from both the monthly
  cron and `scope=all`** (`docs/designs/shared-managed-postgres.md:353-358`),
  because an in-place reset with no overlap window breaks every live consumer
  until ESO re-syncs. Do not "improve consistency" by folding it into the
  rotate-all schedule.
- **The break-glass `generate` action never revokes**, which is why the docs say
  to prefer it mid-incident over `rotate` — "a regen failure *after* the revoke …
  leaves you with **no live root token**"
  (`docs/workflows/llz-breakglass-openbao.md:70-75`).

---

## F-13 (INFORMATIONAL, but the recommended first action) — No DR ADR exists, and the house style says there should be one.

`docs/adr/README.md:12-27` lists 14 records. Two of them exist **specifically to
record an accepted gap with its honest residue**: ADR 0009 ("Unmeasurable
credential coverage" — "It stays unmeasured, and is the honest residue of this
ADR," `docs/adr/0009-unmeasurable-credential-coverage.md:158`) and ADR 0012
("Credential observability gaps," whose entire "Consequences" section is a
what-this-does-not-do list, `:175-191`). `docs/lessons-learned.md` (read in full,
323 lines) records operational scars but contains **nothing** on backup,
snapshot, restore, or DR.

So: the repo has a well-exercised convention for writing down a gap it is
choosing to live with — and **that convention has never been applied to disaster
recovery**, despite DR containing the platform's single highest-consequence
unmitigated risk (F-1).

**Why the absence is itself the finding.** Under this repo's own norms, an
undocumented gap is an *omission*, not a decision — which is precisely the
distinction ADR 0012 draws about the seal key: "Not by decision. By omission."
(`tools/cmd/llz/ci_credential_coverage_guard.go:13`). Every finding in this
document is currently in that state.

**Recommendation — do this first.** Write **ADR 0014, "Disaster recovery
posture."** Take the next free number from the index table, not `ls | tail -1`
(`docs/adr/README.md:49-54` — that is how two collisions happened). It should
state:

1. RTO and RPO per data class (F-8) — the decision that settles most of the rest.
2. The OpenBao raft snapshot decision (F-1) — build it, or accept the loss
   explicitly and say what re-bootstrap costs.
3. That the clone/snapshot admission deny is **not** the DR path and why (F-2).
4. Bucket versioning and `prevent_destroy` (F-5), including the provider-capability
   answer.
5. Cross-region parity measurement (F-6) — or an explicit statement that the
   standby is not promotion-ready and what would make it so, given that its
   bring-up path has never run in CI.
6. The two escrow keys and what verifies them (F-4b, F-7).
7. **That the recovery quorum is single-custodian by design, and shares a loss
   boundary with the seal key (F-14).** This is the one a reader will otherwise
   infer wrongly from `-recovery-shares=5 -recovery-threshold=3`.
8. The managed-Postgres backup posture, once established (F-9) — noting that
   `pg_dump` via the never-pruned admin credential is the de facto recovery story.

Then name one gate per accepted item, per AGENTS.md. In priority order the gates
are: **`assert-escrow-present`** (F-4b / F-7 / F-14 / F-4c's protection-rule
check — cheapest, highest consequence, and the GitHub probe code already exists in
`state_passphrase.go:63-72`), **`assert-openbao-parity`** (F-6, the largest
measurement gap), **`assert-raft-snapshot`** (F-1, needs the mechanism built
first), **`assert-obj-versioning`** (F-5, conditional on the provider answer), and
the **seal-key restore rehearsal** in `llz-wedge-gameday` (F-4c). Separately, and
outside the gate framing: F-6's e2e blind spot is only closable by an HA-pair e2e
lane, which is a materially larger investment and should be scoped as its own
decision rather than folded into a gate.
