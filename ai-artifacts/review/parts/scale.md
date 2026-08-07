# Scalability review — multi-env / multi-cluster / multi-instance

Read-only review. Scope: how one org runs N environments and M clusters, and where
the model breaks as N and M grow. All paths are repo-relative to
`/Users/lcatlett/projects/akamai/lke/fork-lke-landing-zone`.

**Finding count: 15** (S-01 … S-15). Three are labelled **[DOCUMENTED]** — an
accepted tradeoff or a known-and-written-down gap, reported as such rather than as
novel.

**Coverage caveat, stated rather than papered over:** `docs/secrets.md` could not be
opened — both `Read` and `grep` on that exact path are refused by a local permission
rule in this session (`File is in a directory that is denied by your permission
settings` / `Permission to use Bash … has been denied`). Every OpenBao-path claim
below is therefore sourced from `docs/adr/0009-unmeasurable-credential-coverage.md`
and from code, never from `secrets.md`. Nothing here asserts what `secrets.md` does
or does not say.

---

## 0. The model in one page (answers Q1)

### What is per-env vs shared

An instance repo describes any number of **deployments** (`<env>`), discovered
dynamically from `terraform-iac-bootstrap/cluster/<env>.tfvars` — there is no
hardcoded env list (`docs/environments-and-promotion.md:16-20`,
`instance-template/.github/workflows/llz-discover-deployments.yml:45-51`).

| Axis | Per-deployment | Shared across the instance |
|---|---|---|
| Terraform | own state key; `cluster/`, `object-storage/`, `databases/` tfvars (`tools/cmd/llz/render.go:279-285`) | the generated TF roots themselves — env-identical, all variation in tfvars (`render.go:259-263`) |
| Platform values | a **thin** `apl-values/<env>/` overlay (`tools/cmd/llz/main.go:334-337`) | `apl-values/values.yaml` base (`instance-template/.template-manifest:105-113`); the heavy `platform-apl/` manifests live **once** at repo root and are pulled as pinned kustomize remote refs — not cloned per env |
| Network | dedicated VPC by default (`terraform-modules/llz-cluster/main.tf:6-9`) | optional shared VPC per `spec.networks`, own state key `vpc/<name>` (`render.go:265-275`) |
| CI identity | GitHub Environment `infra-<env>` + its env-scoped secrets (`llz-scheduled-checks.yml:82,165,230,351`) | repo variables `TF_IMAGE`, `TF_STATE_BUCKET`, `TF_STATE_ENDPOINT`, `PREFLIGHT_*`; repo/instance-wide GitHub PATs (`llz-scheduled-checks.yml:37-45`) |
| Credentials | `LINODE_API_TOKEN` is per-env (`llz-scheduled-checks.yml:217-221` — "Linode PATs are per-env while GitHub PATs are instance-wide") | `APL_VALUES_REPO_TOKEN`, `OPENBAO_SECRETS_WRITE_TOKEN`, `GHCR_READ_TOKEN`, `E2E_DISPATCH_TOKEN` |
| OpenBao | one OpenBao per cluster; HA pairs via `ha_role`/`ha_group` (`tools/internal/clusterspec/types.go:330-334`) | — |

**OpenBao path fan-out** (asked in Q1; `docs/secrets.md` is unreadable here, so this
comes from code + ADR 0009). `credPaths` declares **15** tracked KV paths
(`tools/cmd/llz/reconcile_openbao.go:104-145` — loki/harbor object-store keys, the
narrow and broad Linode PATs, grafana/otel generated secrets, four GitHub-PAT copies,
alert webhooks, an opt-in firewall token), and the sampler additionally discovers
per-cluster paths from the KV collection rather than the static list, "so a cluster
added later is" picked up automatically (`reconcile_openbao_test.go:155-160`).
`docs/adr/0009-unmeasurable-credential-coverage.md:18-19` reports the coverage as
"every OpenBao path a platform policy knows about is age-tracked — 16 of 16."
OpenBao is **per cluster**, not shared, so the path set scales linearly with M —
which is fine, because it is a fixed ~15-path template per cluster rather than a
per-deployment authoring burden. This is the one credential axis that does *not*
degrade with N.

That split is architecturally sound: adding a deployment adds tfvars + a thin
overlay, not a copy of the platform. The manifests-live-once property is the single
most important scaling decision in the repo and it is correct.

**Where the model breaks as N grows** is not in the render layer — it is in the five
places catalogued below: scheduled-workflow fan-out (S-04/S-05/S-06), the Linode API
client (S-07), account quotas (S-08), secret fan-out (S-09/S-10), and fixed
platform-component sizing (S-13/S-14).

---

## S-01 — Shared VPC is implemented end-to-end, but the multi-cluster case is explicitly **unverified**

**Answers Q2.** Bring-your-own / shared VPC *is* supported — this is not a
"every env creates its own" situation.

**Evidence**
- `terraform-modules/llz-cluster/main.tf:6-9`
  ```hcl
  create_vpc = var.vpc_id == ""
  vpc_id     = local.create_vpc ? linode_vpc.this[0].id : var.vpc_id
  ```
  `linode_vpc.this` is `count = local.create_vpc ? 1 : 0` (`main.tf:17-22`), and
  `linode_vpc_subnet.nodes` attaches to `local.vpc_id` either way (`main.tf:35-42`).
- The cluster is bound with **both** `vpc_id` and `subnet_id` (`main.tf:58-64`) —
  and the comment there records why: passing `subnet_id` alone makes LKE-E silently
  provision its own `lke<id>` VPC. That is the confirmed root cause of the
  VPC-quota leak in `docs/lessons-learned.md:211-218`.
- Spec surface: `spec.networks` (name → region), referenced per env as
  `cluster.network.vpc` (`tools/internal/clusterspec/types.go:315-324`), rendered to
  `vpc/<name>.tfvars` with its own state key (`tools/cmd/llz/render.go:265-275`),
  applied by a dedicated `apply-vpc` job before `apply-cluster`
  (`instance-template/.github/workflows/llz-terraform.yml:265-277, 421-427`).
- Validation enforces same-region, non-overlapping subnets, and distinct HA-peer
  CIDRs, resolving unset CIDRs to `10.0.0.0/13` so a silent collision is caught
  (`docs/landing-zone-spec.md:406-413`; `clusterspec/types.go:308-314`).

**The gap** — `terraform-modules/llz-cluster/variables.tf:32-42`:

> `NOTE: multiple LKE-E clusters sharing one VPC is unverified — see the spec's cluster.network.vpc and the shared-VPC bootstrap-ordering note before relying on it.`

and `docs/landing-zone-spec.md:427-434`:

> **Shared-VPC apply: built; one live check remains.** … What remains is a real `plan`/`apply` against Linode to confirm the `data.linode_vpcs` lookup + attach end-to-end.

**Why it matters** — the shared VPC is the *only* mechanism that lets M same-region
clusters avoid consuming M VPCs against the account quota, and account VPC quota is
the confirmed cause of a silent 30-minute cluster-create hang (S-08). So the feature
that relieves the hardest scaling ceiling is the one feature with no live proof.

**Recommendation** — add a shared-VPC lane to `release-e2e`: two same-region
deployments referencing one `spec.networks` entry with non-overlapping `/14`s,
asserting (a) `data.linode_vpcs` resolves, (b) both clusters report the *same*
`vpc_id` output, and (c) exactly one `linode_vpc` exists on the account for the pair.
Until that lane is green, keep the caveat in `variables.tf` — do not let a doc
promote it to "supported".

---

## S-02 — One node pool per cluster, with hardcoded labels; no workload-pool axis

**Evidence** — `tools/internal/tfroots/roots/cluster/main.tf:69-93` declares exactly
one `linode_lke_node_pool "this"`, and its labels are literals:

```hcl
labels = {
  environment = "shared"
  role        = "observability"
}
```

The spec models a **single** `NodePool` struct, not a list
(`tools/internal/clusterspec/types.go:272, 284-295`). The module README states the
intent — "This module intentionally creates no node pools. Add them as separate
resources in the calling configuration" (`terraform-modules/llz-cluster/README.md:86`)
— but the shipped calling configuration adds exactly one.

**Why it matters** — this is a real scaling axis, and there is an incident behind it.
`docs/lessons-learned.md:48-61` records gsap-apl burning three iterations trying to
clear a `nodeSelector: {workload: builds}` "that the cluster had no pool for". A
single-pool cluster cannot separate platform from workload, cannot host a
build/GPU/memory-optimised pool, and cannot drain one class of node independently.
Growing "M clusters" is the only available answer to a workload that needs different
hardware — which multiplies control planes, VPCs, OpenBao instances and CI fan-out
for what should be a second pool.

**Recommendation** — promote `cluster.nodePool` to `cluster.nodePools` (a keyed map,
with the existing scalar accepted as the `default` key for back-compat) and emit
`for_each` over `linode_lke_node_pool`. Move the two hardcoded `labels` into the
per-pool spec with the current values as defaults. The `moved {}` block at
`main.tf:63-66` is the precedent for doing this without a destroy/recreate — a
`moved` from `linode_lke_node_pool.this` to `linode_lke_node_pool.this["default"]`
keeps existing pools in state, which matters because `node_type` is ForceNew.

---

## S-03 — LKE autoscaler is wired and adopter-exposed, but **off** by default

**Answers Q4.**

**Evidence**
- Terraform: `tools/internal/tfroots/roots/cluster/main.tf:87-93`
  ```hcl
  dynamic "autoscaler" {
    for_each = var.autoscaler_enabled ? [1] : []
    content { min = var.autoscaler_min, max = var.autoscaler_max }
  }
  ```
  with `node_count = var.autoscaler_enabled ? null : var.node_count` (`main.tf:74-75`).
- Defaults: `autoscaler_enabled = false`, `autoscaler_min = 3`, `autoscaler_max = 6`
  (`tools/internal/tfroots/roots/cluster/variables.tf:100-116`), echoed in the
  rendered tfvars (`tools/cmd/llz/testdata/render_golden.txt:1024-1028`).
- Adopter surface: `spec.cluster.nodePool.autoscalerEnabled / autoscalerMin /
  autoscalerMax` (`tools/internal/clusterspec/types.go:284-295`), mapped at
  `clusterspec/tfvars_map.go:45-53`, settable with `llz env set <env>
  cluster.nodePool.autoscalerMin=…`.
- The min/max pointers exist *because* of a fixed gap — the type comment records it
  (`types.go:289-292`): "a spec could turn autoscaling ON but had no way to set its
  range — the rendered tfvars are git-untracked, so hand-editing them was not an
  option either."
- Merge semantics correctly honour an explicit `autoscaler: false` over an inherited
  `true` (`tools/internal/clusterspec/merge.go:40`, tested at
  `clusterspec/instance_test.go:224-243`).

So: **wired, min 3 / max 6, exposed to the adopter, not hardcoded — but disabled by
default.** `docs/adopter-guide.md:146` groups the autoscaler with "keep unless sizing
differs", which reads as guidance to leave it off.

**Why it matters** — with the autoscaler off, every cluster is a fixed `node_count`.
Combined with S-02 (one pool), an org's only elasticity lever is a Terraform apply.
That is a defensible default for a landing zone (predictable cost, deterministic
e2e), but it should be a *stated* default, not an implicit one.

**Recommendation** — two small, low-risk moves. (1) State the default and the
tradeoff explicitly in `docs/adopter-guide.md` alongside a one-line "turn it on"
snippet. (2) Add a `llz doctor` advisory (not a failure) when
`autoscaler_enabled = false` **and** `promotion_rank > 0` — i.e. a cluster in a real
promotion pipeline pinned to a static node count. This is the pattern of the existing
warn-only checks (`health-openbao` emits `::warning::` and exits 0 —
`docs/workflows/llz-scheduled-checks.md:96-98`).

---

## S-04 — **[DOCUMENTED]** Three daily scheduled jobs per env race the same LKE-E control-plane ACL

**Answers Q3 (contention).** This is already written down and not yet landed — report
it as unlanded work, not as a discovery.

**Evidence** — three jobs in `instance-template/.github/workflows/llz-scheduled-checks.yml`
share an identical gate `if: … github.event.schedule == '0 6 * * *'`, all depend only
on `discover`, and all matrix over the same deployment list:

| Job | line | `if:` | matrix source |
|---|---|---|---|
| `lke-admin-rotation-health` | :157, :167, :171 | `'0 6 * * *'` | `needs.discover.outputs.deployments` |
| `credential-single-pane` | :222, :232, :236 | `'0 6 * * *'` | same |
| `prometheusrule-health` | :343, :353, :357 | `'0 6 * * *'` | same |

Each one runs `./.github/actions/cluster-access` (opens this runner's egress IP in the
LKE-E control-plane ACL — `:96-109`, `:182-195`, `:368-381`) and
`./.github/actions/lke-runner-acl` with `mode: revoke` (`:145-155`, `:206-215`,
`:429-438`). Nothing sequences them: no `needs:` between them, `fail-fast: false`, and
**no `max-parallel`** on any of the three.

So per deployment per day: **3 concurrent opens + 3 concurrent revokes = 6
read-modify-write mutations of one shared per-cluster ACL object.**

This is precisely the race the *weekly* fold was created to eliminate —
`docs/workflows/llz-scheduled-checks.md:56-62`:

> Folding them into one job saves three of those cycles per region per week. The part that matters more than runner minutes: it makes **three fewer read-modify-write mutations of the shared LKE-E ACL object**, which the four concurrent jobs were racing on every single week.

And the daily version is already named as remaining work —
`docs/designs/instance-slimming.md:155-157`:

> three `llz-scheduled-checks.yml` matrix jobs share an `if:` and preamble → one job, which also removes **2 redundant control-plane ACL open/close cycles per region per run** (the real argument, not the ~75 lines).

**Mitigation that exists** — the ACL RMW path has its own jittered retry:
`aclRetryDelay = 3s`, `aclMaxAttempts = 4`, and `aclSleep` adds up to +50% jitter
"to break lockstep between two runners retrying in parallel"
(`tools/cmd/llz/runner_acl.go:75-89`). That bounds the damage; it does not remove the
contention, and it burns 4 attempts × N envs × 3 jobs of Linode API calls at 06:00.

**Why it matters at scale** — the contention is per *cluster*, so it does not worsen
with N. What worsens with N is the aggregate Linode API call rate at a single instant
(see S-07), and the wall-clock cost of retry storms across the fan-out.

**Recommendation** — land the documented fold: collapse the three `'0 6 * * *'` jobs
into one job with the same step sequence, exactly as the weekly fold did. The
existing weekly job (`:74-155`) is the template, including the
`if: always() && steps.kubeconfig.outputs.available == 'true'` per-step pattern and
the `tee -a` (not `tee`) rule at `:119-123` that becomes live once steps share a job.

---

## S-05 — Cron literals are identical in every instance; no jitter, no stagger, no templating

**Answers Q3 (would 50 instances all fire at :00?).** Yes — verifiably.

**Evidence** — the caller stubs carry bare literals with no copier token:

| File | line | cron |
|---|---|---|
| `instance-template/.github/workflows/scheduled-checks.yml` | :18 | `"0 2 * * 0"` (weekly) |
| same | :21 | `"0 6 * * *"` (daily) |
| same | :23 | `"0 7 1 * *"` (monthly) |
| `instance-template/.github/workflows/secret-rotation.yml` | :23 | `"0 4 1 * *"` (monthly full rotation) |
| same | :25 | `"30 3 * * *"` (daily reapers) |

A repo-wide grep for `cron` across `copier.yml` and `instance-template/` returns only
these five literals plus prose comments — there is no `<@ … @>` substitution near any
of them, and `copier.yml` asks **no** scheduling question (its entire question set is
`upstream_org`, `instance_repo`, `openbao_team`, `llz_version`, `llz_image_ref` —
`copier.yml:156-257`).

The **template repo's own** scheduled workflows sit in the same slots rather than
avoiding them: `.github/workflows/go-vuln-audit.yml:11-12` is `'0 2 * * 0'` — and its
inline comment says so outright, *"same slot the instance job had"* — i.e. identical
to every instance's `weekly-cluster-checks`. (`.github/workflows/build-images.yml:17-18`
is `0 3 * * 1`, template-only image rebuild, no instance contention.) So the
coincident-schedule pattern is repo-wide convention, not an instance-side oversight.

The trigger surface *is* deliberately operator-editable:
`instance-template/.template-manifest:114-127` keeps the stubs `merge` specifically so
"they own the dispatch/trigger surface an operator may legitimately tune (cron
schedules, dispatch defaults)". But `merge` is an *affordance*, not a default — the
manifest itself notes these three stubs are now token-free and held in `merge`
"purely for the trigger-surface affordance". A scaffolded instance that nobody edits
fires at the literal minute.

**Why it matters** — 50 instances scaffolded from this template all fire at exactly
02:00 / 03:30 / 04:00 / 06:00 / 07:00 UTC. Where those instances share a Linode
account (the common case for one org running many system teams), the 06:00 spike is
`50 instances × 3 jobs × N deployments` simultaneous control-plane-ACL RMW cycles
against one account's API — into a client with no rate-limit handling (S-07).

**Recommendation** — render a deterministic per-instance offset at scaffold time.
The mechanism already exists: the stubs are `merge`-classified so a rendered value
survives, and copier can derive a stable offset from an existing answer (e.g. a hash
of `instance_repo` → minute 0-59, and for the daily jobs hour 6 or 7). Ship it as two
new hidden copier questions (`when: false`, like `llz_image_ref` at
`copier.yml:248-251`) so adopters are not prompted, and keep the literals as the
documented fallback. Spreading five crons across a 60-minute window converts a
coordinated spike into a flat load with zero behavioural change.

---

## S-06 — Scheduled-check job count is 3N+1 per day with no `max-parallel`

**Answers Q3 (how many, at what cadence, per-env matrix?).** Yes, per-env matrix —
and unbounded fan-out.

**Job math per instance with N deployments** (all from
`instance-template/.github/workflows/llz-scheduled-checks.yml` and
`llz-secret-rotation.yml`):

| Cadence | Cron | Jobs |
|---|---|---|
| Daily 06:00 | `0 6 * * *` | `discover` (1) + 3 matrix jobs × N = **3N + 1** |
| Daily 03:30 | `30 3 * * *` | `discover` + `setup` + `revoke-linode-pat` + `revoke-tf-state-key` ≈ **4** |
| Weekly Sun 02:00 | `0 2 * * 0` | `discover` (1) + `weekly-cluster-checks` × N = **N + 1** |
| Monthly 1st 04:00 | `0 4 1 * *` | `discover` + `setup` + `rotate` × N + `create-linode-pat` + `propagate-linode-pat` × N ≈ **2N + 4** |
| Monthly 1st 07:00 | `0 7 1 * *` | `discover` + `template-drift` = **2** |

Every one of those jobs runs `container: image: ${{ vars.TF_IMAGE }}` — a container
pull per job — and carries `timeout-minutes: 15` (20 for `weekly-cluster-checks`,
`:83`). Upper-bound daily runner time is therefore ~45N minutes.

`max-parallel` appears exactly once in the whole instance workflow tree, on
`rotate-state-passphrase` (`llz-secret-rotation.yml:469`, `max-parallel: 1`). None of
the scheduled-check matrices bound their width; all set `fail-fast: false`
(`:86, :169, :234, :355`).

**Why it matters** — at N=10 deployments the daily 06:00 run is 31 jobs starting
simultaneously, each pulling a container and each opening a control-plane ACL. That
is fine on GitHub-hosted runners for one instance; it is the term that multiplies
against the instance count in S-05.

**Recommendation** — after landing the S-04 fold (which takes daily from 3N+1 to
N+1), add `max-parallel` to the remaining per-env matrices, sized to something like
5. `fail-fast: false` must stay — the point of these matrices is that one bad
deployment does not hide the others. Bounding width costs a little wall-clock and
removes the thundering herd.

---

## S-07 — The `llz` Linode API client has no 429 / rate-limit / backoff handling at all

**Answers Q1 (Linode API rate limits).**

**Scope this claim precisely:** this is about the **CLI's own Go client**. Linode
calls made on the Terraform path go through the Linode provider, which is outside
this repo and has its own retry behaviour. The finding is about every `llz ci …` verb
that talks to Linode directly — ACL open/revoke, credential rotation, the reaper,
discovery, preflight.

**Evidence** — `tools/internal/linode/linode.go`:
- `:34` — `return &Client{token: token, http: &http.Client{Timeout: timeout}, base: APIBase}`. A bare `http.Client` with a per-request timeout. No `Transport` wrapper, no limiter.
- `:46-66` — `do()` is the single funnel for every verb ("Every verb below (and the
  paginated GET in rotate.go) funnels through here"). It builds the request, sets
  `Authorization`, calls `c.http.Do`, and returns. There is **no** status inspection,
  no `429` branch, no `Retry-After` read, no backoff.
- `:78-95` — `ListRaw` returns `(nil, resp.StatusCode, nil)` on any non-2xx, with the
  doc comment "The returned error is non-nil only for transport or parse failures".

A grep for `rate.?limit|429|backoff|TooManyRequests|retryable` across
`tools/internal/linode/**` returns **zero matches** (verified; the same grep across
`tools/` returns only unrelated hits — `ImagePullBackOff` strings, Argo retry specs,
the ACL RMW retry, a log rate-limiter at `reconcile_leader.go:66`).

**The ACL retry loop in S-04 does not cover this**, and the distinction matters. The
`aclRetryDelay` / `aclMaxAttempts` / jittered `aclSleep` machinery at
`tools/cmd/llz/runner_acl.go:75-89` is **read-modify-write conflict retry** — it
re-reads the ACL object and re-writes it when a concurrent runner clobbered the
update. It is not rate-limit retry. A `429` returns from `do()` as an unclassified
non-2xx, so under actual throttling that loop can burn its 4 attempts re-issuing a
call that was never going to be admitted.

**The one place a 429 is silently swallowed** — `ListRaw`'s only production caller is
`tools/cmd/llz/import_linode.go:124`:

```go
if buckets, _, err := client.ListRaw(ctx, "v4", "object-storage/buckets"); err == nil {
```

The status code is discarded (`_`). A rate-limited response therefore yields
`buckets == nil, err == nil` — indistinguishable from "this account has no buckets".
That is the exact failure shape `docs/lessons-learned.md:121-133` names as the class
gates exist for: "reported 'none matched the filter' — which reads exactly like
'nothing to do.'"

**Why it matters** — the coincident-cron design (S-05) plus unbounded per-env fan-out
(S-06) plus a 4-attempt ACL retry loop (S-04) produces the exact traffic shape that
triggers rate limiting, into a client that cannot recognise it. The first symptom
will not be an error; it will be a scheduled check that reports clean.

**Recommendation** — two changes, both small and testable, in
`tools/internal/linode/linode.go`:
1. In `do()`, add a bounded retry on `429` (and `5xx`) honouring `Retry-After`, with
   jittered exponential backoff. `runner_acl.go:79-89` is the in-repo precedent for
   the jitter shape — reuse its rationale rather than inventing a second one.
2. Make `ListRaw`'s contract fail-closed: return a typed error for `429` specifically,
   so a caller cannot spell it `_`. Then fix `import_linode.go:124` to branch on it.

Per `AGENTS.md`'s name-the-gate requirement, the gate is a unit test against an
`httptest` server (the `base` field at `:24-27` exists as exactly that seam): serve
`429` + `Retry-After: 1`, assert the client retries and eventually succeeds, and
assert `ListRaw` on a bare `429` returns a non-nil error.

---

## S-08 — **[DOCUMENTED]** Account-level Linode quota is the real M-cluster ceiling; the preflight that catches it is opt-in and defaults to report-only

**Answers Q1 (where the model breaks) and Q7 (documented limits).**

**Evidence**
- Root cause and symptom are fully written up at `docs/lessons-learned.md:192-218`:
  a fresh LKE-E apply stuck on `Still creating...` to the job timeout, because
  "at the account's VPC quota the create hangs with **no API error**". **Confirmed
  root cause:** VPC-quota exhaustion from a per-cycle leak (the missing `vpc_id`
  bind, now fixed — see S-01). It also records "There is no Linode compute-quota
  API, so any 'quota exceeded' claim is unverifiable from automation."
- The guard exists: `tools/cmd/llz/ci_preflight.go:70-71` defines `--vpc-limit` /
  `--vcpu-limit`, and `:173-177` / `:199-203` fail the apply with an explicit
  `::error::` naming the hang.
- **But both default to `0`, which the flag help defines as report-only**
  (`ci_preflight.go:70-71`), and the workflow wires them from repo variables that
  default to `'0'`: `instance-template/.github/workflows/llz-terraform.yml:484-485`
  ```yaml
  PREFLIGHT_VPC_LIMIT:  ${{ vars.PREFLIGHT_VPC_LIMIT || '0' }}
  PREFLIGHT_VCPU_LIMIT: ${{ vars.PREFLIGHT_VCPU_LIMIT || '0' }}
  ```
- The runbook tells the operator to set them — *after* the first failure:
  `docs/runbooks/first-build-failed.md:77` ("Set `PREFLIGHT_VPC_LIMIT` /
  `PREFLIGHT_VCPU_LIMIT` to your account's limits so the next one fails fast instead
  of hanging").

**Why it matters** — this is the hard ceiling on M. Each dedicated-VPC cluster
consumes one account VPC. A fresh instance ships with the guard disarmed, so the
Nth cluster-create hangs for 30 minutes and then fails with no diagnostic — the
documented worst experience in the repo. The quotas are per-account, so this is also
the coupling between *instances*: two teams' instances on one Linode account share
the ceiling and neither one's preflight knows about the other.

**Recommendation** — this is an onboarding-gap fix, not a code fix. Add
`PREFLIGHT_VPC_LIMIT` / `PREFLIGHT_VCPU_LIMIT` to the credential/variable set that
`llz tokens` and `llz doctor` already assert on, as a **warning** when unset with a
message naming the 30-minute hang. The pattern to copy is the credential-coverage
work in `docs/adr/0009-unmeasurable-credential-coverage.md:104-107`: an *absent*
value is reported explicitly rather than skipped, precisely because "a never-written
credential is indistinguishable from a healthy one". Same argument applies here.

---

## S-09 — Secret fan-out is per-deployment GitHub Environments, and one propagation list is a hand-maintained string

**Answers Q1 (secret fan-out, PAT scope).**

**Evidence**
- Secrets are scoped to a GitHub Environment per deployment: every cluster-touching
  job declares `environment: infra-${{ matrix.region }}`
  (`llz-scheduled-checks.yml:82, 165, 230, 351`;
  `llz-terraform.yml:277, 431, 605, 675, 732, 913, 990, 1040, 1212, 1270, 1326`;
  `llz-secret-rotation.yml:167, 356, 473, 615`). So adding a deployment adds one
  GitHub Environment and its full env-scoped secret set.
- The API shape confirming per-env secret storage is spelled out in
  `docs/adr/0009-unmeasurable-credential-coverage.md:84-86`:
  `GET /repos/{owner}/{repo}/environments/infra-<region>/secrets/<name>`.
- Split of scope: "Linode PATs are per-env while GitHub PATs are instance-wide"
  (`llz-scheduled-checks.yml:219-221`).
- The **manual** part — `tools/internal/clusterspec/types.go:255-262`:
  ```
  // Because the broad PAT is ACCOUNT-wide, the component runs on EXACTLY ONE deployment.
  BroadPATLabel string
  // BroadPATDeployments is a SPACE-separated list of deployment names whose
  // infra-<d> LINODE_API_TOKEN environment secret gets the rotated value —
  // matches the CronJob's `strings.Fields(BROAD_PAT_DEPLOYMENTS)` parse (a
  // scalar so `llz env set` can write it).
  BroadPATDeployments string
  ```

**Why it matters** — everything else in the instance derives the deployment set
dynamically from tfvars via one shared reusable workflow, and
`docs/workflows/llz-scheduled-checks.md:41-46` makes the strength of that explicit:
"the set of deployments these checks verify cannot drift from the set the rotation
propagates into. That coupling is the point: it makes 'checked but unrotated' and
'rotated but unchecked' deployments structurally impossible rather than merely
unlikely." `BroadPATDeployments` is the one place that guarantee is broken — a
hand-typed, space-separated string. `llz env add` does not append to it. The failure
mode is a new deployment whose `LINODE_API_TOKEN` silently stops being rotated, which
is invisible until the old PAT is revoked by the daily reaper.

**Recommendation** — either (a) default `BroadPATDeployments` to `llz env list` at
render time when unset, or (b) add a `llz ci` guard asserting
`set(BroadPATDeployments) == set(llz env list --json)` and wire it into `llz lint`.
Option (b) is the repo's own idiom — this is the same "two sides of a contract
renamed independently" class as `TestReaperRecognisesRelabelerOutput`
(`docs/lessons-learned.md:121-129`), and the `add-ci-guard` pattern exists for exactly
this. Option (b) is also strictly safer, since the rotator's blast radius is an
account-wide PAT.

---

## S-10 — Account-wide rotation jobs pin to `deployments[0]` — the alphabetically-first deployment is load-bearing

**Evidence** — four jobs in `instance-template/.github/workflows/llz-secret-rotation.yml`
resolve their GitHub Environment by array index:

| Job | line |
|---|---|
| `create-linode-pat` | `:298` — `environment: infra-${{ fromJSON(needs.discover.outputs.deployments)[0] }}` |
| `revoke-linode-pat` | `:418` — same |
| `create-tf-state-key` | `:525` — same |
| `revoke-tf-state-key` | `:572` — same |

The list is **sorted** — `llz-discover-deployments.yml:15-19` documents the output as
a "Sorted JSON array of deployment names". So `[0]` is whichever deployment sorts
first alphabetically.

**Why it matters** — this is correct-by-construction for account-scoped operations
(you need exactly one environment's credentials to mint an account-wide PAT), and the
concurrency groups are right (`linode-pat-rotation`, `linode-pat-revoke`,
`linode-tf-state-key-rotation`, `linode-tf-state-key-revoke` — `:306, :426, :533, :580`).
But the *selection* is incidental. Adding a deployment named `alpha` to an instance
whose pipeline is `dev`/`staging`/`prod` silently moves account-wide PAT minting into
`infra-alpha`. If that environment is a scratch cluster with narrower secrets, or is
torn down, the monthly rotation breaks — and the daily revoke reaper keeps running.
Nothing gates this: `[0]` is an array index, so there is no error to surface.

**Recommendation** — make the account-scoped identity explicit rather than positional.
Add an instance-level `spec.platform.accountRotationEnv` (defaulting to the current
`[0]` behaviour for back-compat), surface it as a `discover` output, and have
`llz doctor` fail when it names a deployment that does not exist. The precedent is
`BroadPATLabel`, which `Validate` already enforces when the component is enabled
(`clusterspec/types.go:255-257`) — the same treatment, applied to the environment the
rotation runs *from* rather than the PAT family it rotates.

---

## S-11 — The promotion pipeline scales cleanly; its limits are single-repo and single-in-flight

**Answers Q6.**

**Evidence**
- Order is declared per deployment as `promotion_rank` in `cluster/<env>.tfvars`
  (`docs/environments-and-promotion.md:68-91`): ascending = order, `0` = not in a
  pipeline, ranks must be unique, gaps allowed.
- `.github/workflows/promote.yml` is **generated** from those ranks as a static
  `needs:`-chain (`docs/environments-and-promotion.md:144-178`;
  `instance-template/.github/workflows/promote.yml:68-105`), by
  `llz env pipeline` / `llz env add --promotion-rank`
  (`tools/cmd/llz/main.go:378, 385-408`).
- Drift is gated: `llz env pipeline --check` "exits non-zero when promote.yml has
  drifted from the ranks" (`main.go:396-404`), and it runs as a PR job
  (`llz-terraform.yml:244`, `promote-pipeline-drift`).
- Gating is GitHub-native rather than reinvented — `needs:` for the green gate,
  `infra-<stage>` environment protection rules for approval + wait timer, deployment
  branch policy for "only main promotes", "Re-run failed jobs" for resume
  (`docs/environments-and-promotion.md:149-154`).
- Every stage calls the repo-local vendored body, so `secrets: inherit` is same-repo
  (`promote.yml:78-82`; ADR 0003).

**This is the strongest part of the multi-env story** and I want to say so plainly:
adding a stage is a one-line tfvars edit plus a regeneration, with a CI gate proving
the two agree. It genuinely scales to many stages.

**Two real limits, both stated in the design:**
1. **One pipeline per instance repo.** `concurrency: group: promote,
   cancel-in-progress: false` (`promote.yml:64-67`) is instance-wide and unkeyed. An
   org that wants two independent pipelines (e.g. per product line) in one repo
   cannot run them concurrently. This is a deliberate safety choice — "a partial
   apply is worse than a queued one" — but it is a ceiling.
2. **Version promotion is a manual copy.** `docs/environments-and-promotion.md:226-229`:
   "edit `dev`'s pin → merge → build `dev` → on green, copy the pin into `staging`'s
   tfvars in a follow-up PR". So a 3-stage rollout of a chart bump is 3 PRs, and the
   number of PRs grows linearly with stages. Nothing checks that `staging`'s pin is a
   version `dev` actually converged on.

**Recommendation** — for (1), key the concurrency group on a pipeline identifier
rather than the literal `promote` (a `spec.pipelines` name, defaulting to `default`),
so the generated workflow renders `group: promote-<name>`. For (2), add
`llz env promote-pin <from> <to>` that copies the ranked knobs forward and opens the
PR — the ranks are already the source of truth and `llz env next` (`main.go`, exposed
per `docs/environments-and-promotion.md:129-136`) already computes the target. That
turns "copy the pin by hand into the next tfvars" into a command, and gives a natural
place to assert the source stage was green.

---

## S-12 — All PR Terraform plans in one instance repo serialize into a single concurrency group

**Evidence** — `instance-template/.github/workflows/llz-terraform.yml:89-91`:

```yaml
concurrency:
  group: terraform-infra-${{ inputs.region || 'pr' }}
  cancel-in-progress: false
```

On `pull_request`, the caller passes `region: ${{ inputs.region }}`
(`instance-template/.github/workflows/terraform.yml:99`), which is null for a
non-dispatch event — the comment at `terraform.yml:96` confirms "Empty on push/PR".
So every PR run collapses to the group `terraform-infra-pr`, with
`cancel-in-progress: false`.

Meanwhile the PR plan itself is a per-deployment matrix
(`llz-terraform.yml:142-154`, `plan-cluster-pr`, matrix over
`needs.discover.outputs.deployments`, no `max-parallel`).

`push` to main lands in the same group: `inputs.region` is null there too, per the
comment at `terraform.yml:96` ("Empty on push/PR"). So PR plans and main pushes share
one queue.

**Why it matters** — `cancel-in-progress: false` is unambiguously right for *applies*
(the comment says so: "Never cancel in-flight infra changes — a partial apply is
worse than a queued one"). But it is inherited by the read-only PR plan path, where
the reasoning does not hold. At minimum, plan feedback queues behind unrelated PRs
and a superseded commit's plan still runs to completion at N-deployment width before
the current one starts — latency grows with open-PR count, not change size.

**Possibly worse than latency — confirm before acting.** GitHub's documented
concurrency behaviour is that a group holds one *in-progress* run plus at most one
*pending* run, and a newly-queued run **cancels the existing pending one**. If that
holds here, the failure mode is not slowness but a PR whose plan is silently
cancelled because two other PRs queued behind it — no plan comment posted, and the PR
looks fine. I did not verify this against GitHub's docs in this session (no
non-summarized source available), so treat the mechanism as unconfirmed. The
direction of the finding and the fix below are the same either way, and the
correctness reading makes the fix more urgent rather than less.

**Recommendation** — split the group by event. On `pull_request`, use
`group: terraform-plan-${{ github.event.pull_request.number }}` with
`cancel-in-progress: true` (superseding your own stale plan is desirable); keep
`terraform-infra-<region>` with `cancel-in-progress: false` for every apply/destroy
path. GitHub evaluates `concurrency` per workflow run, so this is a single expression
change guarded by `github.event_name`. The apply-path invariant the comment protects
is untouched.

---

## S-13 — **[DOCUMENTED]** Platform-component sizing is fixed; there is no HPA or VPA anywhere in the repo

**Answers Q5.**

**Evidence — asserted only after grepping, and scoped to this repo.** A repo-wide
grep for `HorizontalPodAutoscaler|VerticalPodAutoscaler|autoscaling/v2|kind: HPA`
across `*.yaml`, `*.yml`, `*.tf`, `*.go`, `*.md` returns **zero matches**. So: no
first-party HPA or VPA ships from this repo. apl-core's and Argo CD's own internal
sizing is upstream and outside this tree — I did not read those charts and make no
claim about them.

What this repo does ship is fixed:

| Component | Setting | Evidence |
|---|---|---|
| OpenBao Raft | `replicas: 3` | `kubernetes-charts/llz-openbao-platform/values.yaml:271` |
| OpenBao | `requests 250m/256Mi`, `limits 1000m/1Gi` | `llz-openbao-platform/values.yaml:467-473` |
| OpenBao sidecar | `requests 20m/64Mi`, `limits 200m/256Mi` | same, `:640-646` |
| cert-automation JetStream | `replicas: 3` | `kubernetes-charts/llz-cert-automation/values.yaml:53` |
| cluster-foundation | `requests 10m/16Mi`, `limits 100m/64Mi` | `kubernetes-charts/llz-cluster-foundation/values.yaml:284-290` |
| cluster-foundation | `requests 50m/64Mi`, `limits 200m/256Mi` | same, `:320-326` |
| `llz-reconciler` | `replicas: 1` | `platform-apl/components/llzReconciler/llz-reconciler/deployment.yaml:31` |
| otel-collector | `replicas: 1` | `platform-apl/components/observability/otel-collector.yaml:50` |
| openbao-cert-watcher | `replicas: 1` | `platform-apl/components/openbao/openbao-cert-watcher.yaml:105` |

**The OpenBao `replicas: 3` is not merely a default — it is structurally pinned.**
`llz-openbao-platform/values.yaml:262-271` records why:

> NOTE: the retry_join blocks in the `config` below are written out per-pod for exactly 3 peers, and the openbao-tls cert SANs (platform.tls.secretName) enumerate platform-openbao-0..2 — changing this number means editing both the retry_join list here AND the cert dnsNames in templates/openbao-tls-cert.yaml.

**Why it matters** — none of this scales with cluster size. A 3-node dev cluster and
a 50-node prod cluster get identical platform sizing. `llz-reconciler` at `replicas: 1`
is a singleton on the path that produces the credential metrics the daily
single-pane job gates on (`llz-scheduled-checks.yml:305-324`) — and it has leader
election (`tools/cmd/llz/reconcile_leader.go`), so it is *built* to run >1 but is
deployed at 1. For OpenBao, 3 is correct for quorum; the finding is that scaling *up*
is a two-file surgery, not a value change.

**Recommendation** — (1) Expose `resources` and `replicas` for `llz-reconciler` and
`otel-collector` as chart/component values. Critically, follow the repo's own rule
when doing so: `docs/lessons-learned.md:48-61` establishes that a map-valued default
an operator may need to *clear* (`resources`, `nodeSelector`, `tolerations`) **must be
gated by a scalar toggle**, because `{}` and `null` both fail silently through Argo
`ApplicationSet` values coalescing — gsap-apl burned three iterations proving it. So
ship `resources.enabled: true` + the map behind it, not a bare overridable map.
(2) For OpenBao, leave `replicas: 3` and instead add a chart unit test asserting that
`ha.replicas`, the `retry_join` peer count, and the `openbao-tls` cert `dnsNames`
agree — so a future edit fails at build time rather than at raft-join.

---

## S-14 — Only two components expose sizing knobs; `copier.yml` asks no sizing question at all

**Answers Q4/Q5 (parameterized vs hardcoded).**

**Evidence**
- `copier.yml` (root) asks **five** questions in total — `upstream_org` (`:156`),
  `instance_repo` (`:165`), `openbao_team` (`:204`), `llz_version` (`:223`),
  `llz_image_ref` (`:248`, hidden via `when: false`). **No sizing question of any
  kind.** All sizing lives downstream in the spec / tfvars, set by `llz env add` and
  `llz env set`.
- Cluster sizing *is* well parameterized at `llz env add`
  (`tools/cmd/llz/main.go:346-379`): `--node-type`, `--node-count`, `--k8s-version`,
  `--subnet-cidr`, `--network`, `--ha-role`, `--ha-group`, `--promotion-rank`, plus
  the autoscaler fields via `llz env set` (S-03).
- Platform-component sizing is a much narrower surface —
  `tools/internal/clusterspec/types.go:246-251` is the complete list:
  ```
  // observability → apps.prometheus.*
  Retention string   // prometheus.retention (e.g. 7d, 30d)
  Storage   string   // prometheus.storageSize (e.g. 10Gi)
  Replicas  *int     // prometheus.replicas
  // harbor → registry image-store PVC
  RegistryStorage string // harbor registry PVC size (e.g. 20Gi)
  ```
  Four fields, covering two components (observability, harbor). Everything in the
  S-13 table is unreachable from the spec.

**Why it matters** — the copier layer is correctly minimal (sizing belongs in the
spec, which is reviewable and re-renderable — hardcoding it in copier answers would
make it un-updatable). The gap is the *spec's* platform-component surface: an adopter
can size Prometheus retention but not the collector that feeds it, and cannot size
the reconciler whose metrics the daily credential gate depends on.

**Recommendation** — extend `ComponentToggle` with a shared, optional
`resources`/`replicas` pair, threaded through `RenderBroadPATEnvPatch`'s existing
"config in the spec, mechanism in the base" pattern (`types.go:252-254`) — i.e. the
base manifest ships the default and the per-env patch fills it. Start with
`llzReconciler` and `observability`'s otel-collector, since those are the two
singletons on the day-2 signal path. Same scalar-gate caveat as S-13.

---

## S-15 — No documented tested limit for max nodes, max environments, or max apps

**Answers Q7.** This is an *absence*, so it is stated only after grepping.

**What was searched** — `docs/` and `README.md` for
`max (nodes|envs|environments|apps|clusters)`, `tested (up )?to`, `scales? to`,
`no more than`, `at most [0-9]+`, `limit of [0-9]+`, `up to [0-9]+ (node|cluster|env)`,
plus `scale|scaling|how many|multiple deployments|many clusters` in
`docs/adopter-guide.md`.

**What came back** — no capacity or tested-limit statement anywhere. Every hit was
about *Linode account quotas* (S-08) or about e2e lane sequencing. The only
scale-adjacent line in the adopter guide is `docs/adopter-guide.md:146`, a table row
telling adopters to keep the node/autoscaler defaults "unless sizing differs".

The nearest thing to a stated constraint is the honest negative in
`docs/lessons-learned.md:198-199`: "There is no Linode compute-quota API, so any
'quota exceeded' claim is unverifiable from automation."

**Why it matters** — an adopter has no answer to "how many deployments can one
instance repo carry?" or "how many clusters can we run on one Linode account?". The
implicit answers are derivable from this review — deployments are bounded by
scheduled-workflow fan-out (S-06) and secret fan-out (S-09); clusters are bounded by
account VPC/vCPU quota (S-08) unless the unverified shared VPC (S-01) is used — but
none of that is written down where an adopter looks.

**Recommendation** — add a short "Scale and limits" section to
`docs/adopter-guide.md` stating what is *known* rather than inventing a tested
number: the per-deployment job math from S-06, the per-deployment artefacts from §0,
the account-quota ceiling and the `PREFLIGHT_*` variables that catch it (S-08), and
the shared-VPC caveat (S-01). Explicitly say no upper bound has been load-tested.
Per the repo's own doctrine on precaution prose — if a future reader would have to
*obey a warning*, prefer the task that removes the warning — pair the section with
the S-08 `llz doctor` advisory, so the quota ceiling is enforced rather than merely
documented.

---

## Summary table

| # | Finding | Class |
|---|---|---|
| S-01 | Shared VPC built; multi-cluster case explicitly unverified | gap, documented caveat |
| S-02 | One node pool per cluster, hardcoded labels | architectural limit |
| S-03 | Autoscaler wired + exposed, but off by default | default choice, undocumented |
| S-04 | 3 daily jobs/env race the same LKE-E ACL | **[DOCUMENTED]**, unlanded |
| S-05 | Identical cron literals, no jitter/stagger | multi-instance risk |
| S-06 | 3N+1 daily jobs, no `max-parallel` | fan-out |
| S-07 | Linode client: no 429/backoff; `ListRaw` status discarded at one caller | correctness + scale |
| S-08 | Account VPC/vCPU quota is the M ceiling; preflight disarmed by default | **[DOCUMENTED]**, onboarding gap |
| S-09 | `BroadPATDeployments` is a hand-typed list breaking the discover coupling | drift risk |
| S-10 | Account-wide rotation pins to `deployments[0]` | fragile selection |
| S-11 | Promotion scales well; one pipeline per repo, manual pin copy | limit, by design |
| S-12 | All PR plans serialize into `terraform-infra-pr` | CI latency |
| S-13 | Fixed platform sizing; zero HPA/VPA in repo; OpenBao 3 structurally pinned | **[DOCUMENTED]** (OpenBao coupling) |
| S-14 | Only 4 sizing fields in spec; copier asks none | surface gap |
| S-15 | No documented tested limit (nodes/envs/apps) | documentation gap |
