# Review part — Terraform/OpenTofu state management, locking, and state isolation

Repo: `/Users/lcatlett/projects/akamai/lke/fork-lke-landing-zone` @ `41cd28a` (main, clean)
Scope: `tools/internal/tfroots/`, `tools/internal/terraform/`, `instance-template/terraform-iac-bootstrap/`,
`instance-template/.github/workflows/`, `instance-template/.github/actions/{terraform-init,tf-encryption-env}`,
ADRs 0002 / 0007 / 0008, `docs/workflows/llz-terraform.md`, `docs/lessons-learned.md`.
Method: read-only. Every absence below was established by grep over the repo, not by inference.

**Finding count: 12** — 4 High, 4 Medium, 4 Low/Informational.

---

## 0. Answers to the seven questions, up front

| # | Question | Answer |
|---|---|---|
| 1 | Where does state live? | Linode Object Storage via the Terraform **S3 backend**. Bucket from `TF_STATE_BUCKET` (default `<repo-name>-tfstate`, `tools/cmd/llz/tokens.go:160`). Key layout `<module>/<deployment>/terraform.tfstate`, except `vpc` which CI sets to `vpc/<network>/terraform.tfstate`. Backend block: `tools/internal/tfroots/roots/cluster/backend.tf:25-32` (+ 3 siblings). |
| 2 | **Locking** | **There is NO state locking of any kind.** `use_lockfile` appears nowhere in the repo. Serialization is attempted purely with GitHub Actions `concurrency:` groups, and **the groups do not cover the actual write paths** — see F-01. |
| 3 | State encryption | OpenTofu native `encryption {}`, `pbkdf2` → `aes_gcm`, passphrase from repo secret `TF_STATE_ENCRYPTION_PASSPHRASE`. Loss = permanent unrecoverability, documented. Rotation **is designed and well built** (`tf-encryption-env/action.yml`, `secret-rotation.yml` scope `state-passphrase`) — but see F-12, the job may not run at all. Currently **phase 1 — `fallback { method.unencrypted.migrate }`, `enforced` NOT set** (documented, accepted). |
| 4 | State isolation | Per-instance isolation is real (bucket name derives from repo slug). Per-env isolation is real (`<module>/<deployment>/`). **But the `vpc` root has a documented-vs-actual key mismatch that a unit test pins to the wrong value** — F-02. |
| 5 | State loss/corruption | **No bucket versioning, no backup, no recovery runbook.** ADR 0008 twice tells the reader to restore "the pre-migration state snapshot from the bucket" — nothing in the repo creates or retains one. F-04. |
| 6 | Concurrency groups | Present and thoughtfully commented in the **reusable** workflows, but **incomplete**: the PR plan lane and the passphrase-rotation lane both write state under groups disjoint from the apply group (F-01, F-03). The **vendored thin callers** (`terraform.yml`, `bootstrap-openbao.yml`, `breakglass-openbao.yml`, `secret-rotation.yml`) declare **no `concurrency:` at all** — verified by `grep -rn 'group:' instance-template/.github/workflows/*.yml`, which returns 12 declarations, none in a thin caller. **That omission is correct and deliberate**: `bootstrap-openbao.yml:52-56` documents that a caller repeating the group its own `call` job needs deadlocks GitHub outright ("a deadlock was detected for concurrency group … between a top level workflow and 'call'"), and `breakglass-openbao.yml:52-56` repeats the rule. Serialization belongs on the reusable, and it is there. |
| 7 | `prevent_destroy` | **Zero occurrences repo-wide** (`grep -rn 'prevent_destroy' --exclude-dir=.git .` → no output). The only `lifecycle` blocks are `ignore_changes` (`terraform-modules/llz-cluster/main.tf:92-94`, `firewall.tf:101-103`). F-08. |

---

## HIGH

### F-01 — A PR plan job runs `tofu import`, which WRITES state, in a concurrency group disjoint from the apply group. With no state locking, this is a concrete lost-update race on live state.

**Evidence**

- Workflow-level group: `instance-template/.github/workflows/llz-terraform.yml:89-91`
  ```yaml
  concurrency:
    group: terraform-infra-${{ inputs.region || 'pr' }}
    cancel-in-progress: false
  ```
  On a `pull_request` event `inputs.region` is empty (`llz-terraform.yml:33-37`, default `''`), so **every PR run lands in the single group `terraform-infra-pr`**, while an apply for deployment `primary` lands in `terraform-infra-primary`. Different groups → **they can run at the same time**.
- The PR job is not read-only. `llz-terraform.yml:180-187`:
  ```yaml
  - name: Import VPC and subnet if not in state
    working-directory: terraform-iac-bootstrap/cluster
    run: llz ci tf-import --region "$REGION"
  ```
  `llz ci tf-import` shells out to `tofu import` — `tools/cmd/llz/ci.go:729`
  (`tfCommandContext(ctx, "import", "-var-file="+varFile, addr, id)`) and `ci.go:874`.
  `terraform import` **persists a new state serial**.
- The job inits against the *live* per-deployment key: it calls the shared composite with no `state-key` override (`llz-terraform.yml:162-174`), and the composite defaults to `KEY="${{ inputs.module }}/$REGION/terraform.tfstate"` (`instance-template/.github/actions/terraform-init/action.yml:128-129`), with `REGION` = `matrix.region` from `discover` (`llz-terraform.yml:152-156`) — i.e. `cluster/primary/terraform.tfstate`, the same object `apply-cluster` writes.
- Nothing serializes them. `grep -rn 'use_lockfile'` over the whole repo returns **nothing**; the only DynamoDB-equivalent is absent by platform (Linode OBJ). Confirmed no out-of-band injection either: `grep -rn 'TF_CLI_ARGS\|tfbackend\|backend.tfvars\|\.tf\.json'` finds only prose in comments (`roots/*/backend.tf:5`, `terraform-iac-bootstrap/AGENTS.md:94`) — no file, no env var.
- The repo *knows* locking is absent and says so: `docs/workflows/llz-terraform.md:261-264` — *"Linode OBJ has no state locking, so a single concurrency group (`llz-shared-vpc-apply`) serializes all VPC applies to avoid concurrent same-state corruption."* That reasoning was applied to the VPC root and **not** to the cluster root's PR lane.

**Why it matters**
With no lock, two writers each GET the state object, mutate in memory, and PUT. The later PUT wins wholesale. A PR opened while a release apply is in flight silently discards whatever the apply recorded — including resources that now exist in Linode but no longer exist in state. The next apply then proposes to *create* them, hits `Label must be unique among your Cloud Firewalls` / duplicate-VPC errors, and the operator is in the wedge class this repo has an entire skill (`.claude/skills/e2e-triage`) for. Blast radius is a 20–30 minute cluster apply (`terraform-modules/llz-cluster/main.tf:101-103`).

**Recommendation**
Two fixes, both cheap, and the first is the real one:

1. **Determine whether S3-native locking (`use_lockfile`) works against Linode OBJ, and record the measurement.** What is established here: `use_lockfile` appears nowhere in this repo, and CI runs OpenTofu **1.12.5** (`docs/adr/0008-opentofu-migration.md:62`) — new enough that the backend option exists. What is *not* established from this repo is whether Linode OBJ satisfies whatever the S3 backend's locking implementation requires; **do not assume it does because the backend is "S3-compatible."** ADR 0007's own probe table is the counter-example — `PutBucketEncryption` returns `501 NotImplemented` on this platform (`docs/adr/0007-terraform-state-encryption.md:32`). **Probe it the way ADR 0007 probed SSE** (`0007:23-42`): scratch bucket, real requests, results in a table, conditions that would retire the finding. That measurement discipline is already the house standard here. If it works, add `use_lockfile = true` to the static partial in all four `roots/*/backend.tf` — they are embedded, so one edit per root reaches every instance (`tools/internal/tfroots/tfroots.go:1-13`). If it does not, record the measured result alongside ADR 0007's table and treat (2) as the *documented* mitigation rather than the accidental one.
2. **Make the PR lane non-writing, or put it in the region's group.** Either drop `tf-import` from `plan-cluster-pr` (a plan that needs an import is arguably not a PR-time concern), or set a job-level `concurrency: group: terraform-infra-${{ matrix.region }}` on `plan-cluster-pr` so it contends with the apply it can corrupt. The latter is a two-line change and is the same pattern `apply-vpc` already uses (`llz-terraform.yml:271-273`).

---

### F-02 — The `vpc` root's documented state key does not match the key CI actually uses, and the unit test written to catch exactly this class of drift pins the *wrong* value.

**Evidence** — three sources, three different keys:

| Source | Key |
|---|---|
| `tools/internal/tfroots/roots/vpc/backend.tf:10` | `key = "vpc/terraform.tfstate"` — "shared VPCs are instance-wide (not per-region)" |
| `tools/internal/tfroots/roots/vpc/main.tf:7` | "This root is applied per-network (state key `vpc/<name>`)" |
| `instance-template/.github/workflows/llz-terraform.yml:396` | `state-key: vpc/${{ steps.net.outputs.net }}/terraform.tfstate` |
| `instance-template/.github/actions/terraform-init/action.yml:64-65` | "The shared-VPC root sets `vpc/<network>/terraform.tfstate`" |

`tools/internal/tfroots/backend_key_test.go:28` asserts the **`backend.tf` comment**, i.e. the wrong one:
```go
"vpc": `key      = "vpc/terraform.tfstate"`,
```
and its own header (`backend_key_test.go:13-21`) states the exact hazard:

> *"`terraform init` against another root's state key loads that root's state, and every resource in it is absent from this configuration — so the next plan proposes DESTROYING them… this only bites a by-hand apply — which is what the runbooks describe."*

**Why it matters**
An operator following `backend.tf:10` by hand inits `vpc/terraform.tfstate` — an object **no CI run ever writes**. That state is empty, so `tofu apply` creates a *second* VPC (or 409s on the label), and a subsequent `tofu destroy` against that key does nothing while the operator believes it did. Worse, if a legacy single-network instance ever did write `vpc/terraform.tfstate`, the by-hand path and the CI path manage two different objects for one real VPC. The test that exists specifically to catch cross-root key drift is currently *enforcing* the drift.

**Recommendation**
Fix `roots/vpc/backend.tf:10` to `key = "vpc/<network>/terraform.tfstate"`, update `backend_key_test.go:24-28` to match, and extend the test to assert the documented key against the **workflow's** `state-key:` expression rather than against a hand-maintained second copy — that is what makes it a coupling test rather than a comment-linter. (`.claude/skills/gate` calls out exactly this class: a gate that pins a restatement instead of the coupling.)

---

### F-03 — `rotate-state-passphrase` declares itself "EXCLUSIVE against every other Terraform job" but uses a concurrency group no other Terraform job shares.

**Evidence**

- `instance-template/.github/workflows/llz-secret-rotation.yml:459-460`:
  > `# EXCLUSIVE against every other Terraform job — a concurrent apply would write state with one key while this rewrites it with another.`
- Its actual group, `llz-secret-rotation.yml:479-481`:
  ```yaml
  concurrency:
    group: terraform-state-${{ matrix.region }}
    cancel-in-progress: false
  ```
- The apply group is `terraform-infra-<region>` (`llz-terraform.yml:90`). **`terraform-state-primary` ≠ `terraform-infra-primary`.** The stated exclusivity does not exist.
- Contrast `llz-secret-rotation.yml:173-175` (`rotate-lke-admin`) and `:619-620` (`rotate-db-admin`), which *do* use `terraform-infra-${{ matrix.region }}` and whose comment at `:619-620` says so explicitly — so the correct pattern is present in the same file, two jobs away.
- The state writes happen in the `llz` verb, not in the composite's init: `llz-secret-rotation.yml:514` → `llz ci rotate-state-passphrase --roots-dir terraform --apply` (`tools/cmd/llz/ci_rotate_state_passphrase.go`). The re-key mechanism is `tofu state pull | tofu state push -` — `docs/adr/0009-unmeasurable-credential-coverage.md:135-137`: *"`tofu state pull | tofu state push -` re-encrypts with no provider API calls."* `state push` is a full state write, once per root, across every deployment.
- **ADR 0009 names an exclusive concurrency group as one of three mitigations for this exact hazard, and it is not implemented.** `docs/adr/0009-unmeasurable-credential-coverage.md:173-177`:
  > *"A rollover is all-or-nothing across an instance. A half-completed one leaves roots split across two keys. Mitigated by retaining the old passphrase until every root verifies, by re-runs converging, and by **an exclusive concurrency group** — but it is the sharp edge of this decision."*

  Two of the three mitigations are real (`llz-secret-rotation.yml:449-457`). The third is the group at `:480`, which is exclusive against nothing.
- Same file, `:467-468`, contains a second incorrect belief: *"two deployments re-keying at once would contend on the shared VPC root's **state lock**."* There is no state lock. The `max-parallel: 1` at `:469` is doing the work the comment attributes to Terraform.

**Why it matters**
A re-key rewrites every state object with a new encryption key. An apply racing it writes the *old* key's ciphertext over the new. With `TF_STATE_ENCRYPTION_PASSPHRASE_OLD` still set the rollover appears to converge; once the operator deletes the old secret (which the runbook gates on `rotate-state-passphrase` exiting 0 — `llz-secret-rotation.yml:449-453`), the clobbered root is **permanently unreadable**. This is the one failure mode ADR 0007 calls out as unrecoverable (`docs/adr/0007-terraform-state-encryption.md:123-127`).

**Recommendation**
Change `llz-secret-rotation.yml:480` to `group: terraform-infra-${{ matrix.region }}`, matching the two sibling jobs in the same file and the comment at `:459-460`. Delete or correct the "state lock" claim at `:467-468` — an incorrect mental model of what serializes these jobs is what produced this bug. Add an `llz ci` guard asserting that every job that runs `tofu apply`/`import`/`rotate-state-passphrase` declares a `terraform-infra-*` group (this repo's `add-ci-guard` skill describes the exact pattern for turning a failure class into a PR-time gate).

---

### F-12 — `rotate-state-passphrase` inits with `module: aws-init-only`, which is not a root and does not exist as a directory anywhere in the repo. The composite `cd`s into it.

**Evidence**

- `instance-template/.github/workflows/llz-secret-rotation.yml:496`:
  ```yaml
  - name: Terraform init (rotation window — both keys)
    uses: ./.github/actions/terraform-init
    with:
      module:                aws-init-only
  ```
- The composite hard-codes the working directory from that input — `instance-template/.github/actions/terraform-init/action.yml:121-123`:
  ```yaml
  - name: Terraform init
    working-directory: terraform-iac-bootstrap/${{ inputs.module }}
  ```
  and derives the state key from it at `:128-129` (`KEY="${{ inputs.module }}/$REGION/terraform.tfstate"`, since no `state-key` is passed).
- **`aws-init-only` is not a root.** `tfroots` embeds and emits exactly four: `cluster`, `databases`, `object-storage`, `vpc` (`tools/internal/tfroots/tfroots.go:64-81` walks `roots/` for `*.tf`; `find tools/internal/tfroots -type f` confirms the four). The composite's own input description enumerates the same four (`terraform-init/action.yml:8`).
- **`aws-init-only` appears exactly once in the entire repository.** `grep -rn 'aws-init-only' --exclude-dir=.git .` → one hit, `llz-secret-rotation.yml:496`. Nothing creates the directory, nothing else references the name.

**Why it matters**
The intent is legible — this job needs `TF_ENCRYPTION` exported (which the composite does at `:102-111`, *before* the init step) and needs AWS credentials in the environment, but has no single root to init against, because `llz ci rotate-state-passphrase --roots-dir terraform` (`:514`) walks all four roots itself. So the init step is meant to be a no-op. **But it is not written as one**: GitHub Actions fails a step whose `working-directory` does not exist, before the `run:` block executes. On that reading, `rotate-state-passphrase` fails at `:493` on every invocation and never reaches the re-key at `:514` — i.e. **the state-encryption passphrase cannot be rotated**, and the mitigation ADR 0009 leans on (`0009:173-177`) is unavailable in practice.

I could not execute the workflow to confirm the failure (read-only review), so state this as: either the job is broken as described, or `aws-init-only` is an intentional sentinel whose no-op behavior is undocumented and unguarded. **Both need fixing, and they need different fixes**, which is why this is High rather than a nit.

**Recommendation**
1. **Determine which it is** — check whether `rotate-state-passphrase` has ever completed on any instance (`gh run list --workflow secret-rotation.yml`). This is a one-command answer and it decides everything below.
2. If broken: call `./.github/actions/tf-encryption-env` **directly** from this job instead of going through `terraform-init`. That action is standalone by design and has no `working-directory` at all (`tf-encryption-env/action.yml:64-74`) — it was extracted precisely so paths that do their own init could use it (`tf-encryption-env/action.yml:11-23`). This job is exactly that case; routing it through `terraform-init` to reach the nested action is the indirection that produced the phantom module.
3. Either way, add a guard: `terraform-init` should fail with a named message when `inputs.module` is not one of the four roots. Today an invalid module produces a directory-not-found from the runner, or — worse, if the directory ever came to exist — a `tofu init` against the state key `aws-init-only/<region>/terraform.tfstate`, which is the cross-root-key hazard `backend_key_test.go:13-17` exists to prevent, arriving through the input rather than through a comment. A three-line `case` in the composite closes it.

---

## MEDIUM

### F-04 — No state bucket versioning, no state backup, no recovery runbook — while two documents instruct the reader to restore from a snapshot that nothing produces.

**Evidence**

- The state bucket is created by `llz tokens` with a bare API call — `tools/cmd/llz/tokens.go:164` → `client.CreateObjectStorageBucket(ctx, clusterID, bucketName)`, whose body is `{"cluster":…, "label":…}` only (`tools/internal/linode/rotate.go:228-231`). **No versioning, no object lock, no lifecycle policy is configured anywhere.** Grep for `versioning` across the repo returns only Renovate config and an unrelated HTTP-header comment.
- The state bucket is deliberately **not** Terraform-managed (chicken-and-egg): the `object-storage` root manages only the Loki/Harbor buckets via `terraform-modules/llz-object-storage` (`tools/internal/tfroots/roots/object-storage/main.tf:11-21`).
- No backup mechanism exists: `grep -rni 'state backup\|backup the state\|snapshot.*state'` finds only prose.
- Yet `docs/adr/0008-opentofu-migration.md:68-70`:
  > *"Rolling back means **restoring a pre-migration state snapshot from the bucket**, not merely reverting this commit."*

  and `:134`: *"the one-way property above is why the pre-migration state snapshot is worth keeping."* Nothing keeps it.
- `instance-template/terraform-iac-bootstrap/AGENTS.md:113`:
  > *"Do not run `terraform state rm`, `terraform state mv`, or `terraform init -reconfigure` without explicit user approval. **State corruption is not recoverable without a backup.**"*

  Correct, and there is no backup.
- No runbook covers it. `docs/runbooks/` contains 12 files (`apl-branch-recreate-wedge`, `bootstrap-openbao`, `first-build-failed`, `orphan-volume-cleanup`, …) — **none about state loss, state corruption, or `terraform import` recovery**. The only `terraform import` recovery prose in the repo is `docs/designs/shared-managed-postgres.md:228`, scoped to one Postgres instance.
- `tools/internal/terraform/import.go` and `heal.go` are **not** a state-recovery path — they are the node-pool-ID selector and the firewall-collision / transient-API-flake parsers for `llz ci tf-import` and the self-heal loop (`import.go:11-28`, `heal.go:46-54`).

**Why it matters**
The repo has correctly identified that state loss is unrecoverable, has written that down in two places, and has then built an instance-provisioning path that leaves the operator with no way to recover. Combined with F-01 and F-03 (two live paths that can clobber state), the probability side of the risk is not zero.

**Recommendation**
1. **Probe whether Linode OBJ implements `PutBucketVersioning` / `GetBucketVersioning`, and record the result in ADR 0007's measurement table** (`0007:23-42`) — that table is the precedent and the right home. Do **not** assume support from S3-compatibility: the same table shows `PutBucketEncryption` returning `501 NotImplemented` on this platform (`0007:32`), which is precisely the inference that ADR disproved. If versioning is implemented, enable it at creation time in `llz tokens` (one additional call after `CreateObjectStorageBucket` at `tokens.go:164`), with a noncurrent-version expiry so it does not grow unbounded. If it is not implemented, that measured `501` is itself the finding, and the fallback is an explicit scheduled `tofu state pull` snapshot to a second bucket — note the machinery already exists (`ADR 0009:135-137`) and the plaintext-handling discipline it requires is already written down (`ADR 0009:182-184`).
2. Add `docs/runbooks/state-loss.md` covering: restore a prior object version; the `tofu import` re-adoption path per root (the resource addresses are already enumerable from the roots); and the interaction with encryption (a restored object still needs the passphrase *and* the key-provider name it was written under — see F-06).
3. Reconcile ADR 0008's two "snapshot" sentences: either they name the mechanism added in (1), or they are changed to state that no snapshot exists. As written they transfer a false assurance to a future reader — the litmus test in `~/.claude/rules/owning-layer-fixes.md` applied to authored artifacts.

---

### F-05 — State encryption is phase 1 (`enforced` not set), so an unencrypted state write is accepted, not refused. *(Documented, accepted tradeoff.)*

**Evidence**

- `tools/internal/tfroots/roots/vpc/encryption.tf:54-66` (byte-identical across all four roots — verified by `md5sum`, all `8ce7d274a9c073e55573c4e3f8435021`):
  ```hcl
  terraform {
    encryption {
      state { fallback { method = method.unencrypted.migrate } }
      plan  { fallback { method = method.unencrypted.migrate } }
    }
  }
  ```
- Rationale and the two-phase plan: `encryption.tf:30-44` and `docs/adr/0007-terraform-state-encryption.md:90-104`. Status line `ADR 0007:3` — *"accepted, phase 1 shipped. Phase 2 (enforcement) is a follow-up."*
- `docs/adr/0012-credential-observability-gaps.md:140` independently flags that the fallback *"is also what makes an unencrypted state file accepted."*

**Assessment: this is a correctly documented, correctly reasoned accepted tradeoff.** OpenTofu genuinely rejects `enforced` + `unencrypted` fallback together (verified at ADR 0007:117), so phase 1 is not optional. The posture-in-code / key-in-env split (`ADR 0007:71-88`) is a good design: it converts "silently writes plaintext" into a hard init failure, and the `tf-encryption-env` action preflights the secret with an explicit message (`tf-encryption-env/action.yml:75-98`).

**Recommendation (schedule, not redesign)**
Phase 2 has no owner or trigger recorded. Add the phase-2 flip to a tracked issue with a concrete precondition ("every deployment's `cluster`, `vpc`, `object-storage`, `databases` state shows `encrypted_data`"), and add an `llz ci` verb that *reports* that precondition per deployment so the flip is a measurement rather than a judgement call. Until then, `enforced` absent means a hand-run `tofu apply` with a *stale but valid* `TF_ENCRYPTION` that omits the state block will migrate encrypted state back to plaintext without complaint.

---

### F-06 — Key rotation is well designed and the key-name constraint is well documented, but the **escrow checklist captures only half the decryption tuple**.

**Evidence**

- The mechanism is genuinely well built. `instance-template/.github/actions/tf-encryption-env/action.yml:131-166` emits the full HCL; during a rollover it emits both key providers plus an **encrypted** fallback (`:140-154`) — legal alongside `enforced`, so a rotation never relaxes the posture (`:147-148`). Injection guards on both passphrases (`:87-92`, `:113-118`) and on the key names as HCL identifiers (`:99-108`). Collision guard at `:123-126`.
- The load-bearing constraint, `tf-encryption-env/action.yml:33-44` / `terraform-init/action.yml:33-44`:
  > *"OpenTofu stores pbkdf2's salt at `meta["key_provider.pbkdf2.<name>"]`, so state can only be decrypted by presenting its passphrase under the name it was WRITTEN with… (verified against OpenTofu 1.12.3: reusing the name for a new passphrase fails with "decryption failed for all attempted"). Defaults to `llz`, the name every existing state file was written under — **changing this default would strand all of them**."*
- The name lives in `vars.TF_STATE_ENCRYPTION_KEY_NAME` (`llz-terraform.yml:171`, `llz-secret-rotation.yml:501`) — a **GitHub repo variable**, mutable by anyone with repo admin, with no guard that it matches what the state was actually written under.
- **The constraint is documented, and well.** `grep -rn 'TF_STATE_ENCRYPTION_KEY_NAME' docs/ instance-template/` (run) returns `docs/quickstart.md:744`, `docs/adr/0009-unmeasurable-credential-coverage.md:131` and `:178`. ADR 0009:178-181 is explicit:
  > *"`TF_STATE_ENCRYPTION_KEY_NAME` is now load-bearing config. If the variable drifts from the name state was actually written under, decryption fails — loudly, not silently, which is the right failure, but it is a new way to break a working instance."*
- **The gap is narrower than "undocumented": it is the escrow checklist.** The two places that tell an operator what to put in offline escrow name only the passphrase — `docs/quickstart.md:913` (*"`TF_STATE_ENCRYPTION_PASSPHRASE` saved offline — printed once by `llz tokens`; lose it and every Terraform state file is unreadable"*) and `docs/secrets.md:275` (*"**ESCROW OFFLINE**: until a rollover completes, losing it makes every state file unrecoverable"*). Neither line mentions the key name. The `TF_STATE_ENCRYPTION_KEY_NAME` grep does **not** hit `quickstart.md:913` or `secrets.md:275`.

**Why it matters**
Before any rotation the name is `llz` for everyone, so passphrase-only escrow is sufficient and the checklist is right. **After the first rotation it stops being sufficient** — the decryption secret becomes the tuple `(passphrase, key-provider name)`, and the checklist still captures one element. An operator who escrows the new passphrase and later loses the repo variable (repo recreated, variable overwritten, instance migrated to a new org) holds a passphrase that decrypts nothing. ADR 0009 predicts the failure precisely ("a new way to break a working instance"); the checklist has not caught up with the ADR.

**Recommendation**
- Add the key name to the escrow checklist (`docs/quickstart.md:913`) and to the secrets table (`docs/secrets.md:275`), stated as "escrow the **pair**". This is a two-line docs change that closes a gap the ADR already identified.
- Have `llz ci rotate-state-passphrase` print the `(key-name, written-at)` pair on success and require it be recorded, the same way `llz tokens` prints the passphrase once.
- Consider a preflight in `tf-encryption-env` that reads the state object's `meta` keys and fails loudly when `key_provider.pbkdf2.<KEY_NAME>` is absent from a state that has `encrypted_data` — converting a silent "decryption failed for all attempted" into a named diagnosis. This is the same move the action already makes for the missing-passphrase case (`:75-78`).

---

### F-07 — `llz ci fetch-kubeconfig-state` is a second, independent `tofu init` path against live cluster state, and the workflows that call it declare no concurrency group.

**Evidence**

- `tools/cmd/llz/fetchkubeconfig_state.go:139` hardcodes `stateKey := fmt.Sprintf("cluster/%s/terraform.tfstate", region)` and inits with bucket/key/region only (`:146-149`) — it does **not** go through `.github/actions/terraform-init`. `tf-encryption-env/action.yml:11-23` documents exactly this: the action was extracted *because* this path bypasses `terraform-init` and broke when ADR 0007 landed.
- The operation itself is read-only (`tofu output -raw kubeconfig_raw`, `fetchkubeconfig_state.go:161`) — `output` does not write state, so this is **not** a corruption path.
- But `instance-template/.github/workflows/llz-cluster-health.yml` consumes `TF_STATE_BUCKET` (`:63`) and declares **no `concurrency:` block at all** — `grep -n 'concurrency' llz-cluster-health.yml cluster-health.yml llz-scheduled-checks.yml llz-wedge-gameday.yml` returns nothing for any of them.

**Why it matters**
Read-only today, so the risk is bounded to a stale read racing a mid-apply state (a health check that reads a kubeconfig for a cluster being replaced). The structural concern is that a second init path exists whose state key is a **Go string literal** rather than derived from the same place CI derives it — if the cluster key layout ever changes (e.g. to add an instance prefix, see F-09), `fetchkubeconfig_state.go:139` is a silent second copy that will not be updated by changing the composite action.

**Recommendation**
Extract the key layout into one exported helper in `tools/internal/tfroots` (it already owns `DefaultVPCSubnetCIDR` for exactly this reason — `tfroots.go:34-46`, "three copies of one literal, each claiming to mirror the HCL, with nothing enforcing it — that is the drift this prevents"). Have both `fetchkubeconfig_state.go` and a unit test against `terraform-init/action.yml:129` consume it. Add `concurrency: group: terraform-infra-<region>` to `llz-cluster-health.yml` only if it ever gains a writing step; today a comment stating it is read-by-design is sufficient.

---

## LOW / INFORMATIONAL

### F-08 — No `prevent_destroy` on the cluster, the VPC, or any bucket.

**Evidence** — `grep -rn 'prevent_destroy' --exclude-dir=.git .` → **no matches**. The only `lifecycle` blocks in the repo are `ignore_changes`: `terraform-modules/llz-cluster/main.tf:92-94` (`control_plane[0].acl`, `pool`) and `terraform-modules/llz-cluster/firewall.tf:101-103` (`inbound`).

**Assessment — this is defensible, and I would not add `prevent_destroy` naively.** Destruction is gated *procedurally* and quite thoroughly: `llz ci assert-destroy-confirm "$REGION" "<module>" "$CONFIRM_DESTROY"` runs at the head of all six destroy jobs (`llz-terraform.yml:618, 693, 749, 1003, 1053, 1283, 1339`), the token is `destroy:<deployment>:<module>` (`llz-terraform.yml:39`), and `infra-*` GitHub Environments carry required reviewers (`docs/workflows/llz-terraform.md:249-251`). A `prevent_destroy` on the cluster would also **break the e2e teardown lane**, which is a first-class supported workflow here.

**Recommendation**
Do not add `prevent_destroy` to `cluster`. **Do** consider it for the `object-storage` buckets: `llz-terraform.yml:48-52` already establishes that "destroying a cluster is not consent to erase them" and defaults `drain_data_buckets: false` — a `prevent_destroy` on the Loki/Harbor buckets in `terraform-modules/llz-object-storage` would make that stated policy structural rather than a default someone can flip. Note the interaction with `force_destroy` already flagged at `terraform-modules/llz-object-storage/main.tf:13-22`.

### F-09 — Cross-instance state collision is prevented by convention, not by construction.

**Evidence** — `tools/cmd/llz/tokens.go:160`:
```go
bucketName := firstNonEmpty(bucket, vars["TF_STATE_BUCKET"], repoSlug(instanceRepo)+"-tfstate")
```
`repoSlug` (`tokens.go:547-552`) returns the **repo name only, lowercased — the owner is discarded**. So `acme/landing-zone` and `globex/landing-zone` both default to `landing-zone-tfstate`. Linode OBJ bucket labels are unique per region — the repo already knows this: `roots/object-storage/main.tf:9-10`, *"OBJ bucket labels are global per region."* Two orgs on different Linode accounts get different buckets (accounts are separate namespaces), so this is only reachable when **two instances share one Linode account**, which is the `--bucket` override case (`tools/cmd/llz/main.go:215`).

Key layout below the bucket carries no instance discriminator (`<module>/<deployment>/terraform.tfstate`), so if the bucket is shared, `primary` collides with `primary` exactly.

**Why it matters** — bounded, but the second instance's `CreateObjectStorageBucket` returns 2xx for an already-owned bucket (`tools/internal/linode/rotate.go:225-227`, "effectively idempotent for the caller's own buckets"), so on a shared account the collision is **silent at provision time** and only manifests as the destroy-everything plan at first apply.

**Recommendation** — either include the owner in the default (`repoSlug` → `<owner>-<name>`), or have `llz tokens` fail when the target bucket already exists *and* already contains `*/terraform.tfstate` objects under a different instance's key set. The `llz ci` guard pattern fits; a one-line HEAD probe would catch it at the only moment it is cheap to catch.

### F-10 — `roots/databases/backend.tf` carries a stale scar comment claiming its workflow jobs do not exist.

**Evidence** — `tools/internal/tfroots/roots/databases/backend.tf:19-21`:
> *"CI is unaffected — `.github/actions/terraform-init` derives the key from the module name — but **the workflow jobs for this root do not exist yet**, so applying it by hand (as the runbooks describe) is currently the only way to run it."*

They do exist: `apply-databases` (`llz-terraform.yml:1197`), `plan-destroy-databases` (`:1261`), `destroy-databases` (`:1316`), each passing `module: databases` (`:1233, 1291, 1347`). `backend_key_test.go:19-21` repeats the same stale claim.

**Why it matters** — the comment is the sole justification given for the hand-apply path, which is the *only* path with zero concurrency protection (F-01/F-03 at least have partial groups). Leaving it in place tells a future operator that hand-applying `databases` is normal.

**Recommendation** — delete the "jobs do not exist yet" sentence from `roots/databases/backend.tf:19-21` and `backend_key_test.go:19-21`; replace with a pointer to `apply-databases`. Keep the destroy-hazard paragraph — that part is still true and load-bearing.

### F-11 — `terraform-iac-bootstrap/AGENTS.md` drifts from the roots it documents in three places.

**Evidence**

| `AGENTS.md` | Reality |
|---|---|
| `:73` — `force_path_style = true` | `roots/*/backend.tf:31` — `use_path_style = true` (`force_path_style` is the removed pre-TF-1.6 spelling; copying this block verbatim fails) |
| `:29-36` — "**Three** generated roots: cluster / object-storage / vpc" | Four. `databases` exists (`tools/internal/tfroots/roots/databases/`) and is generated by the same walk (`tfroots.go:64-81`) |
| `:48-52` — apply order omits `databases` | `apply-databases` exists (`llz-terraform.yml:1197`) |

`:113` ("State corruption is not recoverable without a backup") is **correct** and is the sharpest sentence on this topic in the repo — see F-04.

**Recommendation** — correct all three. The `force_path_style` line is the one that actively costs someone time, because `AGENTS.md:97-100` presents a copy-pasteable `terraform init` immediately below it.

---

## Cross-cutting note

The three High findings share one root cause: **serialization is expressed in GitHub Actions `concurrency:` groups, which are strings with no relationship to the state object being written.** There is no mechanism anywhere that makes "this job writes `cluster/primary/terraform.tfstate`" imply "this job holds `terraform-infra-primary`" — the two are matched by hand. Five `group:` declarations across the instance workflows govern Terraform state writes (`llz-terraform.yml:90`, `:272`; `llz-secret-rotation.yml:174`, `:480`, `:622`), and **two of the five are wrong** (`llz-terraform.yml:90` on the PR path, `llz-secret-rotation.yml:480`). `use_lockfile` (F-01) fixes this at the layer that owns the resource; the group corrections (F-01.2, F-03) are the mitigation while that is being measured. A static guard asserting the job↔group↔state-key correspondence is what stops it recurring, and this repo already has the pattern for building one.
