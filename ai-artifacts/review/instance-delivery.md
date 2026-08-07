# Instance-Delivery Review — `lke-landing-zone-example` @ template `e0fbc5c`

Read-only review of the e2e-instantiated instance at
`/Users/lcatlett/projects/akamai/lke/lke-landing-zone-example` (pin:
`.copier-answers.yml:1` → `e0fbc5c1a2949cffda0635f434f7509ba17f6976`) against the template at
`/Users/lcatlett/projects/akamai/lke/fork-lke-landing-zone`. Paths below are relative to
those two roots; `[I]` = instance, `[T]` = template.

## Executive summary

The **mechanical** delivery is excellent. A full recursive diff of `[I].github/` against
`[T]instance-template/.github/` shows drift in exactly 5 files, and every hunk is a copier
`<@ instance_repo @>` → `akamai-consulting/lke-landing-zone-example` substitution — zero
unexplained drift. `.gitignore` coverage of the rendered Terraform build artifacts is exactly
what `render.go:205-214` claims. Every relative Markdown link in the delivered `docs/` tree
resolves to a delivered file. The `managed`/`owned`/`merge` classification in
`.template-manifest` is coherent, and `.template-removals` / `.template-managed.lock` do what
they say.

The **semantic** delivery is where the problems are. The task's premise — that
`docs/landing-zone-spec.md:377-384` is stale — is **refuted**: all three named spec blocks
(`spec.dns.*`, `spec.defaults.platform.*`, `spec.alerting.*`) are genuinely inert. The spec doc
is the *only* place that says so. Six other files, four of which **ship into every adopter's
repo**, tell the operator these settings render — including a code comment at
`render.go:217-219` that names a renderer which does not contain the code it claims. An adopter
who follows the delivered guidance sets `spec.dns.acmeEmail`, re-renders, and gets nothing.

Separately, the entire 23-file delivered `docs/` tree is classified `managed` — the digest-locked
class — yet has **zero entries** in `.template-managed.lock`, and the lock's own header reserves a
"DELIBERATELY UNLOCKED" block that `docs/**` is not in. So the runbooks an on-call engineer edits
under pressure have no drift detection, and nothing declares the gap.

Secondarily: the "delivered scaffold carries no version" claim (`AGENTS.md:88-99`) is **false in
this instance**, and the guard that is supposed to enforce it has pattern holes rather than being
absent. And three of the six `.gitleaks.toml` allowlists are broad `paths` globs pointing at
directories that do not exist in an instance — dead suppressions that contradict the file's own
repeatedly-stated doctrine.

**Counts (14 findings):** 0 Critical · 3 High (F1–F3) · 7 Medium (F4–F10) · 4 Low (F11–F14).

---

## Findings

### F1 — Three spec blocks are inert; six files (four delivered) claim they render
**Impact: High** | **LoE to fix: Medium** (Low if the fix is doc-only; Medium if the renderers are restored)

**Evidence — the claim under test.**
`[T]docs/landing-zone-spec.md:377-384`: "three spec blocks are currently validated but never
rendered: `spec.dns.*` (including `acmeEmail`), `spec.defaults.platform.*` (`externalDNS`,
`externalIDP`), and `spec.alerting.*`". **This statement is accurate.** Verification scope, stated
explicitly:

- `grep -rn "Spec\.Alerting\|lz\.Spec\.DNS\|Spec\.Defaults\.Platform\|\.AcmeEmail" [T]tools/` →
  the only hits are `[T]tools/internal/clusterspec/validate.go:56` (`validateAlerting`) and
  `[T]tools/cmd/llz/import.go:233,398,656-660` (the `llz import` *reader*, a separate struct).
  **No renderer reads any of the three.**
- `grep -rn "HasExternalDNS\|HasExternalIDP" [T]tools/` → 3 files: the definitions at
  `[T]tools/internal/clusterspec/types.go:166,169`, `import_aplvalues.go:34-35,74-75` (a local
  import struct, not the spec type), and `coverage_gaps_test.go:57-69` — a test whose own header
  (`coverage_gaps_test.go:3-5`) says it exists to cover accessors "previously exercised only
  indirectly (or not at all)". **The only production caller of these accessors is nothing.**
- The retirement is documented at `[T]tools/internal/clusterspec/values.go:3-10`: "The apl-core
  value-render pipeline that used to live here — **RenderValues** plus its
  object-store/identity/team wiring — was **RETIRED**… render_test asserts it never does."
  `RenderValues` no longer exists as a function anywhere in `[T]tools/`.
- Ruled out the non-Go path too: `[I]apl-values/values.yaml` is **byte-identical** to
  `[T]instance-template/apl-values/values.yaml` (absent from a per-file diff of all delivered
  top-level files), its `alerts:` block at `[I]apl-values/values.yaml:416-420` is a literal
  `["none"]` (not a `${…}` templatefile token), the ACME email at `:77` is prose describing a
  literal `REPLACE_PER_ENV`, and `otomi.hasExternalDNS/IDP` at `:363-365` are literals. The
  complete set of templatefile tokens in that file is 13 names
  (`grep -oE '\$\{[a-z_]+\}'`), none of which is acme/alerting/otomi-related. And
  `grep -rni "acme\|letsencrypt" [T]terraform-modules/` returns one hit, an unrelated
  `label_prefix = "acme"` example in a README. **Terraform cannot fill them either.**

**Evidence — the six contradicting statements.** Two are internal; four ship to adopters.

| # | Location | Ships to instance? | What it claims |
|---|---|---|---|
| 1 | `[T]tools/cmd/llz/render.go:217-219` | no | acmeEmail "rides as a kustomize patch in each env's manifest overlay (`RenderManifestKustomization`)". `grep -rni "acme\|letsencrypt\|clusterissuer\|email" [T]tools/internal/clusterspec/kustomize.go` → **zero hits.** |
| 2 | `[T]tools/internal/clusterspec/types.go:111-116` | no | "`llz render` writes it ONCE into the shared `platform-apl/manifest/dns/letsencrypt-clusterissuer.yaml`". `find [T]platform-apl -iname "*letsencrypt*"` → **no such file.** |
| 3 | `[T]tools/internal/clusterspec/types.go:119-120` | no | Alerting "rendered into every env's `values.yaml` `alerts:` block". No renderer exists. |
| 4 | `[I]apl-values/README.md:89` (byte-identical to `[T]instance-template/apl-values/README.md:89`) | **yes** | "`spec.dns.acmeEmail`, being instance-wide, is applied by a **JSON6902 patch** in the …" (`grep -n JSON6902`). |
| 5 | `[I]apl-values/values.yaml:9-15` | **yes** | "`llz render` resolves every non-secret `$${...}` placeholder below straight from the spec into the committed `apl-values/<env>/values.yaml`: … the platform flags (`otomi.has*`) … all from `spec.environments.<env>` + `spec.defaults.platform`". Also `:360-362`. **No `apl-values/<env>/values.yaml` exists** — `[I]apl-values/e2e/` contains only `apl-overlay/`, `apps/`, `manifest/`. |
| 6 | `[I]landingzone.yaml.example:52-75` + `[I]landingzone.yaml:27-36` (the operator's *real* spec) | **yes** | `platform.externalDNS/externalIDP` "rendered into every env's values.yaml"; `dns.acmeEmail` "render the Let's Encrypt registration contact into every env's overlay from the spec"; alerting "uncomment, `llz render`". |

**Why it matters.** This is the highest-consequence class of delivery defect: config the operator
believes is live, isn't. Concretely, the acme path fails *silently and permanently*.
`[T]tools/internal/health/certs.go:83` already treats a `REPLACE_PER_ENV` ACME email as an
expected deferral ("ACME registration is deferred until the operator provisions
`dns.acmeEmail`"), so an operator who *did* set `dns.acmeEmail` in
`landingzone.yaml` gets a health tree that reports the deferral as normal, forever, with no
signal that the spec value was never consumed. `spec.alerting` is worse in kind: an operator who
sets `receivers: [slack]` reasonably concludes alerts now reach a human. They do not — and
`[I]apl-values/values.yaml:405` states plainly that "CI health probes are the only cluster-health
alerting that reaches a human", which is the true state.

**Recommendation.** Pick one direction and make all seven locations agree.
1. If the blocks stay inert: delete them from `[I]landingzone.yaml.example` and the rendered
   `landingzone.yaml` scaffold comment, correct `[I]apl-values/README.md:91` and
   `[I]apl-values/values.yaml:9-15,360-362`, correct the three internal comments, and have
   `Validate` **reject** (or `llz doctor` **warn on**) a set-but-unrendered value so the failure is
   loud at render time rather than silent at cert-issuance time. That last item is the actual fix —
   prose alone leaves the trap armed for anyone who read an older copy.
2. If they should render: restore the renderers and add a coupling gate (see `gate` skill) that
   fails when a spec field has a validator but no consumer.

Either way, a **static guard is warranted**: "every `json:` tag in `clusterspec.Spec` is read by at
least one non-test, non-import call site" is machine-checkable and would have caught all three.

---

### F2 — `docs/**` is `managed` but carries zero drift detection, and the gap is undeclared
**Impact: High** | **LoE to fix: Low**

**Evidence.** `[I].template-manifest` line 14 classifies `docs/**` as `managed` — the
digest-locked class per `[I].template-managed.lock:4-6` ("Covers every token-free file in a
digest-locked class of `.template-manifest` (today: `managed`)"). **Evidence scope:** I read all 30
lock entries (`.template-managed.lock:21-50`) in full — every one is a top-level dotfile,
`.github/actions/*`, `.github/workflows/llz-*.yml`, `apl-values/*`, an `*.example`, or
`terraform-iac-bootstrap/*`. **Not one `docs/` path appears.** A per-path check confirms
`docs/quickstart.md` and `docs/README.md` are NOT LOCKED. That is **23 delivered files**
(`find docs -type f | wc -l` → 23: `quickstart.md`, `README.md`, 13 runbooks, 8 playbooks) with no
drift detection.

The lock file explicitly reserves a place for this: `:11-18` lists three "DELIBERATELY UNLOCKED"
paths (`.template-manifest`, `AGENTS.md`, `README.md`) with the rationale "Listed so the gap is
auditable rather than **invisible**." `docs/**` is not in that list.

**Why it matters.** `llz ci managed-fresh` "fails when an instance hand-edits a file the next
`llz upgrade` would overwrite" (`:8-9`). For the entire operator-facing documentation set — the
runbooks an on-call engineer edits under pressure — that protection does not exist, and nothing
tells anyone. An operator's incident-time correction to a runbook is silently discarded on the next
upgrade. The exclusion is *justifiable* (deliver-docs rewrites org-substituted links, so the bytes
are per-instance), but it is undeclared, which is precisely the failure mode the lock's own header
argues against.

**Recommendation.** Add `docs/**` to the DELIBERATELY UNLOCKED block with its reason
(deliver-docs link rewriting), or — better — digest-lock the docs *pre-rewrite* so hand edits are
still caught. Then make `llz ci managed-fresh` fail if any `managed` glob has neither lock coverage
nor an explicit exclusion entry, so a future `managed` class can't silently join this gap.

---

### F3 — "The delivered scaffold carries no version" is false in this instance; the guard has pattern holes
**Impact: High** (governance claim vs. reality) | **LoE to fix: Low**

**Evidence — the claim.** `[T]AGENTS.md:88-99`: "The delivered scaffold carries no version at all…
`llz lint`'s upgrade-churn guard enforces this… **Restating the pin in prose counts too**… The
guard runs on both sides of the delivery: in an instance checkout it scans the vendored workflows
and kept docs."

**Evidence — what the instance actually carries.** `grep -rn "e0fbc5c"` + `grep -rEn "v[0-9]+\.[0-9]+\.[0-9]+"`
over `[I]`, sorted into three classes:

*Sanctioned:* `[I]docs/README.md:14` — the generated pinned-docs pointer, the one exception
`AGENTS.md:92-93` names.

*Rendered artifacts (out of scope of the claim, but still churn):* ~15 hits across
`[I]apl-values/e2e/**/kustomization.yaml` (e.g. `manifest/kustomization.yaml:8,17-21`,
`apps/harbor/kustomization.yaml:7,10`). These are `llz render` output, not scaffold, so they do not
refute the claim — but they are **committed**, so they add ~15 lines to every instance's upgrade
diff on every release, which is the exact cost the guard exists to prevent.

*Genuine violations:*
- `[I]terraform-iac-bootstrap/AGENTS.md:43` — the full 40-char sha restated in prose
  (`?ref=e0fbc5c1a2949cffda0635f434f7509ba17f6976`). `AGENTS.md:95-96` says prose restatement counts.
- `[I]docs/quickstart.md:275, 380, 838` — literal `v0.0.39` ×3.
- `[I].template-manifest:137` — `v0.0.24` (a historical failure-mode reference, benign).

**Evidence — why the guard misses them.** `[T]tools/cmd/llz/upgrade_churn_guard.go:43-69` defines
exactly three patterns: `<@ llz_version @>`, `^\s*template-ref:`, and
`lke-landing-zone/(blob|tree)/v\d+\.\d+\.\d+/`. **None matches a bare SemVer in prose, and none
matches a bare sha.** Separately, `churnGuardInstanceRoots` (`:85-90`) is
`.github/workflows`, `docs/quickstart.md`, `docs/runbooks`, `docs/playbooks` —
`terraform-iac-bootstrap/AGENTS.md` is outside the scanned surface entirely. So `quickstart.md` is
*in* the scanned root and still passes (pattern hole), while the tf AGENTS.md is never looked at
(root hole). Two independent gaps.

**Why it matters.** The claim is load-bearing for the release process (`AGENTS.md:100-107`: "There
is nothing to bump first — the template hardcodes no version"). The guard's own header
(`:22-25`) cites gsap-apl's v0.0.33 upgrade leaving 27 stale permalinks *because nothing in the
instance's CI objected*. The instance-side scan was added for exactly this. **By inspection of the
guard source** (I did not execute `llz lint`), none of its three regexes can match a bare SemVer or
a bare sha, and its instance root list excludes `terraform-iac-bootstrap/` — so all four strings
this instance carries are outside what the guard can detect.

**Recommendation.** Add a fourth `churnPattern` for a bare release SemVer / 40-char sha outside
`.copier-answers.yml` (with a narrow allow for `docs/README.md`, already excluded by root
selection), and add `terraform-iac-bootstrap/**` to `churnGuardInstanceRoots`. Then fix the three
sources: `quickstart.md`'s `v0.0.39` examples should read the pin or use a generic
`<version>`; `terraform-iac-bootstrap/AGENTS.md:43` should link `.copier-answers.yml`. Note the
guard must be updated *before* the strings are fixed, or the fix regresses silently.

---

### F4 — Three of six `.gitleaks.toml` allowlists are broad `paths` globs over paths that do not exist
**Impact: Medium** | **LoE to fix: Low**

**Evidence.** `[I].gitleaks.toml` ships 6 allowlists. Three are `paths`-scoped:
- `:6-9` `research-materials/` — "Research prototype certs and test keys"
- `:12-15` `\.github/actions/vendor/`
- `:19-22` `\.github/actions/setup-terraform/hashicorp\.gpg\.asc`

None of these paths exists in the instance. `[I].github/actions/` contains exactly `_lib/`,
`cluster-access/`, `fetch-kubeconfig/`, `linode-credentials/`, `lke-runner-acl/`,
`terraform-init/`, `tf-encryption-env/` — no `vendor/`, no `setup-terraform/`. There is no
`research-materials/` anywhere in the instance tree (verified against the full 103-file listing).

They are also **doctrinally inconsistent with the rest of the same file**, which argues against
`paths` scoping three separate times: `:56-57` ("a `paths` glob would OR-allowlist the whole file
and hide them"), `:74-75` ("A `paths` glob over sealedsecrets/ would OR-allowlist those whole files
and hide a genuine leak pasted alongside"). The three non-path allowlists (`:36-40`, `:46-50`,
`:58-62`, `:76-80`) are exemplary — `regexTarget = "match"` / `"secret"` with narrow patterns and
recorded reasoning.

**Why it matters.** These are inherited from the *template's own* repo layout, not the instance's.
`research-materials/` is the sharp one: it is a plausible directory name an adopter might create,
and the moment they do, every credential in it is silently exempted from scanning — by a rule
delivered by the platform, not written by them. The other two are inert but train readers that
path-globs are the house style, which the file itself disputes.

**Recommendation.** Drop all three from `[T]instance-template/.gitleaks.toml`. They protect
template-repo artifacts that by construction never reach an instance. If the vendored-actions case
ever becomes real in an instance, re-add it then, scoped to the path that actually exists.

---

### F5 — `.checkov.yaml` ships pointing at paths and a build target that do not exist in an instance
**Impact: Medium** | **LoE to fix: Low**

**Evidence.** `[I].checkov.yaml:2` — "Scans: `kubernetes/cluster` (Linode LKE),
`kubernetes/argocd-bootstrap` (Helm + K8s)". Neither path exists; the instance's Terraform lives
under `terraform-iac-bootstrap/{cluster,object-storage}` and is **gitignored/generated**
(`[I]terraform-iac-bootstrap/.gitignore:18-21`). `:3` — "Run via: `make checkov`". The instance
has **no Makefile** (verified against the full file listing — 103 files, no `Makefile`, no
`Taskfile`). The two `skip-check` entries (`CKV_K8S_30`, `CKV_TF_2`) reference an "argocd
namespace created by Terraform" and a `local_sensitive_file` kubeconfig write, both described in
terms of the template's own root layout.

**Why it matters.** The delivered security-scan config is not runnable by the documented command
and describes a tree the adopter does not have. An adopter reasonably concludes IaC scanning is
covered; nothing in the instance runs it. This is not a suppression concern — neither skipped check
is encryption- or network-related, so the *content* is fine — it is a **coverage** concern: the
scan is delivered as a claim, not a control.

**Recommendation.** Either wire checkov into a delivered CI job that runs after `llz render`
materializes the TF roots (which is when there is anything to scan), or drop the file. A config
that cannot be executed by the instructions printed at the top of it is worse than absent, because
it reads as coverage.

---

### F6 — `LINODE_DNS_TOKEN` silently degrades to a sentinel string; nothing downstream detects it
**Impact: Medium** | **LoE to fix: Low**

**Evidence.** `[I].github/workflows/llz-bootstrap-openbao.yml:302`:
```
LINODE_DNS_TOKEN: ${{ secrets.LINODE_DNS_TOKEN || 'placeholder-set-LINODE_DNS_TOKEN-to-enable-dns' }}
```
`grep -rn "placeholder-set-LINODE_DNS_TOKEN" [I]` returns **exactly one hit — this line**. No
consumer, health check, or `llz doctor` path tests for the sentinel.

**Why it matters.** The fallback is deliberate (DNS is optional), but the failure it produces is
indistinguishable from success at bootstrap time: the token is written into OpenBao, ESO syncs it,
cert-manager's webhook authenticates with a garbage credential, DNS-01 fails, and the
`llz-letsencrypt-*` ClusterIssuers sit `Ready=False` — a state
`[T]tools/internal/health/certs.go:83` explicitly grades as an *expected* deferral. So the two
failure modes (deliberately deferred vs. secret-never-set) collapse into the same green-ish
signal. Compounded by F1: the operator's other lever for this (`spec.dns.acmeEmail`) is inert.

**Recommendation.** Have the bootstrap emit a `::notice::` when the sentinel is used, and have
`llz status` / the certs health check report "DNS-01 disabled (LINODE_DNS_TOKEN unset)" as a
distinct state from "ACME registration deferred". One sentinel comparison, and the two states stop
looking alike.

---

### F7 — The two delivered spec examples contradict each other on the `managedApps` model
**Impact: Medium** | **LoE to fix: Low**

**Evidence.** Both files ship to every instance and sit one directory apart.

- `[I]landingzone.yaml.example:41-48`: "**LLZ enables** the OPTIONAL apps it needs in apl-core
  **AT BOOTSTRAP** (default `managedApps: [harbor, loki, grafana, kyverno]`) — apl-core installs
  them, and LLZ layers its matching extras… on top."
- `[I]environments/prod-web-ord.yaml.example:72-78`: "Managed apl-core installs only a MINIMAL
  core; the OPTIONAL apps (harbor, loki, grafana, …) are **enabled by the operator in the App
  Platform Console**. **Declare the ones you enabled** so LLZ layers its matching extras ONLY for
  apps that exist — **render can't discover them**." Default shown: `[harbor, loki, grafana]` (no
  kyverno).

These are opposite causal models: in the first, `managedApps` is an *instruction* LLZ executes; in
the second it is a *declaration* of what the human already did in a web console. The defaults also
differ by one app.

**Why it matters.** This is the single most consequential knob in the spec for whether converge
wedges — `prod-web-ord.yaml.example:76-77` itself says a mismatch means LLZ layers extras for apps
that don't exist. An adopter reading the two files in either order forms a wrong model half the
time. Note the *real* env, `[I]environments/e2e.yaml`, declares no `managedApps` at all while
enabling `objProxy`/`argoWorkflows`/`clusterHealthWorkflow` — so it demonstrates neither shape.

**Recommendation.** One of the two is stale; determine which from
`docs/adr/0006-managed-default-apps.md` and rewrite the other to match, cross-linking rather than
restating. Then have `llz doctor` print the effective resolved `managedApps` set, so the operator
never has to infer it from prose.

---

### F8 — The instance's most privileged credential is an unrotatable GitHub PAT with a misleading name
**Impact: Medium** | **LoE to fix: Medium** (GitHub exposes no PAT-mint API, so rotation cannot be scripted; a GitHub App installation token replaces the credential outright. The `LINODE_API_TOKEN` half is genuinely inherent — Linode has no OIDC federation.)

**Evidence.** `grep -rhoE "secrets\.[A-Z_]+" [I].github/` yields 18 distinct secrets, dominated by
`LINODE_API_TOKEN` (66 refs), `TF_STATE_SECRET_KEY`/`TF_STATE_ACCESS_KEY` (48 each),
`TF_STATE_ENCRYPTION_PASSPHRASE` (30). All are static values in GitHub Environments.
`secrets: inherit` appears in every vendored caller stub
(`[I].github/workflows/{terraform,secret-rotation,bootstrap-openbao,breakglass-openbao,promote,scheduled-checks,wedge-gameday,cluster-health}.yml`),
forwarding the full secret set to the reusable. OIDC appears **once** in the entire instance:
`[I].github/workflows/llz-secret-rotation.yml:354-355` (`id-token: write`, for OpenBao JWT auth) —
verified by `grep -rn "id-token\|oidc" [I].github/`.

**The highest-value credential is misleadingly named.** The `gha-token` input to the credential
rotator (`[I].github/actions/linode-credentials/action.yml:39-42`, "GitHub PAT with
`secrets:write` scope; used for `gh secret set`") is fed, in both call sites, by
`secrets.OPENBAO_SECRETS_WRITE_TOKEN` (`[I].github/workflows/llz-secret-rotation.yml:332, 559` —
the complete result of `grep -rn "gha-token" [I].github/`). That secret's own hint text
(`[I].github/workflows/llz-terraform.yml:326-327`) describes it accurately: "Fine-grained PAT
(github.com) scoped to this repo with **Actions: write + Environments: write**". It is also used
directly as `GH_TOKEN` at `[I].github/workflows/llz-bootstrap-openbao.yml:175`,
`llz-breakglass-openbao.yml:119`, and `llz-terraform.yml:652`.

So the single most powerful credential in the instance — one that can rewrite **every**
`infra-<env>` Environment secret, including `LINODE_API_TOKEN` and both `TF_STATE_*` halves — is
named as though it were an OpenBao token. `[I].github/workflows/llz-secret-rotation.yml:332` reads
`gha-token: ${{ secrets.OPENBAO_SECRETS_WRITE_TOKEN }}`, which looks like a mis-wiring on first
read and is in fact correct.

**Mitigations present and worth crediting:** every workflow declares `permissions: contents: read`
at the top level and re-declares it per job; `environment: infra-${{ inputs.region }}` scopes the
secret resolution per deployment; the reusables are repo-local (`./`), so `secrets: inherit` is
same-repo and cannot cross an org boundary; the rotator masks aggressively
(`action.yml:161-164, 179-184`), never crosses a job boundary with a raw token (emits a sha256
instead), scrubs step summaries (`:277-289`), and defaults to dry-run (`:35-38`).

**Why it matters.** Linode's API has no OIDC federation, so `LINODE_API_TOKEN` as a rotated static
PAT is the correct available design, and the 90-day rotation machinery is real. Two residual
concerns are worth naming precisely:

1. **No rotation lane covers the two GitHub PATs.** `[I].github/workflows/secret-rotation.yml:29,
   57-77` enumerates the rotation scopes: `linode-pat`, `linode-pat-revoke`, `tf-state-key`,
   `tf-state-key-revoke`, `db-admin`, `lke-admin`. Neither `OPENBAO_SECRETS_WRITE_TOKEN` nor
   `APL_VALUES_REPO_TOKEN` has one. This is largely **inherent** — GitHub exposes no API to mint a
   fine-grained PAT, so a rotation job cannot be written — but the consequence is that the
   credential whose compromise defeats the rotation of all the others is the one credential with no
   rotation at all, and nothing in the delivered surface says so.
2. **The name actively misleads.** A reader auditing "which secrets can write GitHub secrets" and
   grepping for GitHub-shaped names will not find `OPENBAO_SECRETS_WRITE_TOKEN`.

**Recommendation.** Replace it with a **GitHub App installation token** (minted per-run,
short-lived, Environments: write scoped to this repo) — the one place in this design where a
long-lived credential can genuinely be eliminated, and it removes the rotation gap rather than
documenting it. If that is out of scope, at minimum rename the secret to something like
`GH_ENV_SECRETS_WRITE_TOKEN` and add it to the credential inventory
`llz ci assert-rotation-health --require-inventory`
(`[I].github/workflows/llz-scheduled-checks.yml:307`) measures, flagged as manually rotated with a
cadence, so an unrotated year-old PAT is visible rather than invisible.

---

### F9 — `landingzone.yaml` carries the template's own identity as the instance name
**Impact: Medium** | **LoE to fix: Low**

**Evidence.** `[I]landingzone.yaml:8` → `metadata.name: instance-template`, while
`[I]landingzone.yaml:12` → `repo: akamai-consulting/lke-landing-zone-example`. The delivered
example is unambiguous about the contract: `[I]landingzone.yaml.example:20` — "instance name ==
repo short name. e.g. `acme-platform`". Downstream, `[I]environments/e2e.yaml:7,12` carry
`clusterLabel: instance-template-e2e` and `bootstrap.name: instance-template-e2e`, so the live
Linode LKE cluster label is literally `instance-template-e2e`.

**Why it matters.** This is a template-provenance string that leaked into a *real* deployment's
identity and reached actual Linode resource labels. `[I]landingzone.yaml:14-19` warns that
`objLabelPrefix` collisions are global-per-region across accounts; a cluster label derived from
`instance-template` is exactly the kind of non-discriminating literal that collides when a second
adopter's e2e runs in the same region. `objLabelPrefix: llz-e2e` was set correctly, so the OBJ
side is safe — the cluster label was not.

**Recommendation.** Have `llz env add` derive `metadata.name` from `.copier-answers.yml`'s
`instance_repo` short name rather than defaulting to a literal, and add a `llz lint` check that
`metadata.name` matches the repo short name (the example already states the invariant, so it is
machine-checkable today).

---

### F10 — `prod-web-ord.yaml.example` ships a deprecated field, a stale path, and an inert claim
**Impact: Medium** | **LoE to fix: Low**

**Evidence.** All three in one delivered file:
- `:82` — `keyRotationDays: 90 # → obj_key_rotation_days (≤120)`, presented as a live setting.
  `[T]docs/landing-zone-spec.md:142` says `keyRotationDays: DEPRECATED/ignored — rotation is owned
  by the …`. `[I].gitleaks.toml:42-50` even carries a dedicated allowlist for that deprecation
  note, so the deprecation is well established.
- `:87` — points at `apl-values/example/values.yaml` as where "mechanism stays". No such path
  exists in the instance (`[I]apl-values/` contains `README.md`, `values.yaml`, `_shared/`, `e2e/`).
- `:88, :92-94` — sizing knobs "render into the env's `values.yaml`" with per-key mappings
  (`→ apps.prometheus.retention`). Per F1, no per-env `values.yaml` is ever rendered.

**Why it matters.** This is the file an adopter copies to create their first real environment. A
deprecated key presented with a validated-looking range annotation (`≤120`) is likely to be set and
then relied on for a rotation SLA that nothing enforces.

**Recommendation.** Mark `keyRotationDays` `# DEPRECATED — ignored; rotation is owned by <X>` or
remove it; fix the `apl-values/example/` path; reconcile `:88-94` with whatever F1's resolution is.
A `llz doctor` warning on a set-but-ignored spec key would cover this and F1 with one mechanism.

---

### F11 — Delivered `.claude/settings.json` provides no guardrails at all
**Impact: Low** | **LoE to fix: Low**

**Evidence.** `[I].claude/settings.json` is 3 lines: `{"includeCoAuthoredBy": false}`. It is
`managed` (`[I].template-manifest` line 8) and digest-locked
(`[I].template-managed.lock:22`), so the template treats it as a deliberate delivered artifact.
`[I].gitignore:15-19` correctly ignores `settings.local.json` while keeping the shared file
committed.

**Why it matters.** An instance repo contains an operator's live credential cache path (`.llz/`,
holding the Linode PAT and OBJ state key per `[I].gitignore:1-4`) and arming flags on real
infrastructure (`llz credentials … --apply`, `llz ci runner-acl open`). The delivered settings file
grants nothing and denies nothing, so full Claude Code defaults apply in a repo where a mis-fired
Bash call rotates or revokes production credentials. The template already ships an `AGENTS.md`
setting conventions — the enforcement surface is simply empty.

**Recommendation.** Ship a `permissions.deny` covering reads of `.llz/**` and Bash matchers for the
arming flags (`llz credentials * --apply`, `llz ci runner-acl open`), plus an `ask` on
`terraform apply`. Cheap, and it converts a documented convention into a control.

---

### F12 — `environments/e2e.yaml` contradicts itself and matches neither delivered example
**Impact: Low** | **LoE to fix: Low**

**Evidence.** `[I]environments/e2e.yaml:16-17` — "components omitted → all default-enabled except
dns. Add a `components:` block to toggle or size them" — is immediately followed at `:19` by a
`components:` block. A scaffold comment that was not removed when the block was added.

Shape comparison against the documented spec: the real env uses a **flat, minimal** form
(`clusterLabel`, `region`, `ha.role`, `bootstrap.{name,appsRepoRevision}`, `objectStorage.cluster`)
and omits everything the example documents as important — `k8sVersion`, `nodePool`, `controlPlane`,
`network.subnetCIDR`, `promotionRank`, `apiServerAllowCIDRs`, `managedAppPlatform`. Most of those
are legitimately inherited from `[I]landingzone.yaml:39-48` `spec.defaults`, so the shape is
**valid** — but `objectStorage.cluster` differs between the two files (`us-ord-10` real vs
`us-ord-1` in `prod-web-ord.yaml.example:81`) and `bootstrap.managedAppPlatform: true` is set only
in defaults, which the example calls "REQUIRED / mandatory" at the env level
(`prod-web-ord.yaml.example:61-70`) while also noting it is "usually set once in
spec.defaults.cluster.bootstrap".

**Why it matters.** Low individually, but it means the one *worked example of a real environment*
an adopter can inspect demonstrates a different (and much thinner) shape than the annotated
example teaches, with no note explaining that the difference is inheritance.

**Recommendation.** Drop the stale comment at `:16-17`; verify which `us-ord-*` OBJ cluster id is
current and align the example; add one line to `e2e.yaml` noting which fields come from
`spec.defaults`.

---

### F13 — `AGENTS.md` sends operators to a guide the instance does not carry
**Impact: Low** | **LoE to fix: Low**

**Evidence.** `[I]AGENTS.md:61-62` — "non-obvious gotchas are in
[docs/adopter-guide.md](https://github.com/akamai-consulting/lke-landing-zone/blob/main/docs/adopter-guide.md)".
`deliver-docs` correctly rewrote the relative link to an upstream URL
(`[T]instance-template/AGENTS.md:62` had `docs/adopter-guide.md`), because
`[T]tools/cmd/llz/ci_deliver_docs.go:42-47` keeps only `quickstart.md`, `runbooks/`, `playbooks/`,
`README.md`.

**Why it matters.** The link hygiene is *correct* — this is not a broken link. But it does mean the
file the instance's own AGENTS.md calls the source of "non-obvious gotchas" is not available
offline or version-matched. Against the "day-2 from what's delivered alone" bar this is a near
miss, not a failure: `[I]docs/quickstart.md` is 929 lines whose numbered walkthrough (`:83-159`)
covers prerequisites, CLI install, scaffold, `env add`, `doctor`, publish, credential provisioning,
the ~40-minute build, the post-bootstrap manual escrow steps, kubeconfig retrieval and convergence
verification end to end — backed by 13 runbooks and 8 playbooks. **Verified scope for that
judgement:** read the section headings of `quickstart.md` (`grep -n "^#\{1,3\} "`, first 60
matches — note this also matched `#`-comments inside its code block, so the heading list is
indicative rather than exhaustive) and extracted every relative Markdown link across `docs/`,
`README.md` and `AGENTS.md` (35 distinct targets), spot-checked against the instance's full
103-file listing. Day-2 is covered.

**Recommendation.** Either add `adopter-guide.md` to `docsKeep`, or reword the sentence to
"upstream, tracking `main`" so the operator knows it may not match their pin.

---

### F14 — Rendered `apl-values/e2e/**` pins add ~15 churn lines per release
**Impact: Low** | **LoE to fix: Medium**

**Evidence.** `[I]apl-values/e2e/manifest/kustomization.yaml:8,17-21`,
`apps/{harbor,observability,externalSecrets,llzReconciler,broadPatRotator,objProxy}/kustomization.yaml:7`
and the four `newTag: sha-e0fbc5c…` lines — ~15 committed occurrences of the template ref. These
are correct and functional (remote kustomize bases must be pinned), and they are `owned` class
(`[I].template-manifest` line 28, `owned apl-values/*/**`), so they are outside the churn guard's
scope by design.

**Why it matters.** It is the precise cost `[T]tools/cmd/llz/upgrade_churn_guard.go:8-11` was
written to eliminate ("45 of 53 changed lines"), reappearing through a channel the guard does not
watch. Not a defect — a residual, worth naming so it is not mistaken for one when someone next
measures upgrade-diff size.

**Recommendation.** No action required. If upgrade-diff size becomes a complaint again, the lever
is a single rendered `ref` variable the kustomizations reference, not a guard change.

---

## What's DONE WELL

**Vendored surface integrity is exemplary.** A full `diff -r` of `[I].github/` against
`[T]instance-template/.github/` produces drift in 5 files
(`bootstrap-openbao.yml:71`, `breakglass-openbao.yml:65`, `promote.yml:79,94,108`,
`secret-rotation.yml:103`, `terraform.yml:93`) and **every single hunk** is
`instance_repo: <@ instance_repo @>` → `akamai-consulting/lke-landing-zone-example`. Zero
unexplained drift across all **16** workflows and **6** composite actions (plus the shared
`actions/_lib/git-auth.sh`). The same holds for the rest of the delivered surface: of **16**
top-level delivered files checked pairwise, only 5 differ, and all 5 are pure copier substitution (`.devcontainer/devcontainer.json:15`, `.template-manifest:147`,
`AGENTS.md:1,4,62`, `README.md:1,4,30,80`, `renovate.json:15,31`).

**The gitignore/build-artifact claim is exactly true.** `render.go:205-214` claims "an instance
commits ZERO Terraform: the roots are gitignored build artifacts regenerated on every render,
exactly like the per-env tfvars." Verified:
`[I]terraform-iac-bootstrap/.gitignore:16` (`*/*.tfvars`) and `:21` (`*/*.tf`), and the instance's
entire `terraform-iac-bootstrap/` tree is 4 files — `.gitignore`, `AGENTS.md`, and two
`.terraform.lock.hcl` provider pins. Nothing else. `[I].gitignore:1-4` correctly excludes the
`.llz/` credential cache, and `:15-19` makes the shared-vs-local Claude Code split explicit rather
than accidental.

**The three-file template governance system is coherent and well-reasoned.**
`.template-manifest` classifies with sensible last-match-wins layering — `merge
.github/workflows/**` (the thin, token-carrying caller stubs adopters may extend) overridden by
`managed .github/workflows/llz-*.yml` (the reusable bodies), which is exactly the split the lock
file reflects. `owned` correctly covers everything an operator authors
(`landingzone.yaml`, `environments/*.yaml`, `apl-values/*/**`, `kubernetes-custom/**`,
`.terraform.lock.hcl`, `.copier-answers.yml`). `.template-removals` solves a real copier gap
(copier never deletes files a template drops) with two well-chosen modes, and its two live rules
each carry the reasoning for *why* the file was retired, not just the fact. `.template-managed.lock`
is generated, format-documented, and — F2's `docs/**` gap aside — declares its own exclusions.

**Supply-chain hygiene in the workflows.** Every external action is pinned by 40-char SHA with the
version as a trailing comment (`actions/checkout@9c091bb… # v7.0.0`,
`actions/upload-artifact@043fb46d… # v7.0.1`) across all 16 workflows — no floating tags found.
Top-level `permissions: contents: read` on every workflow, re-declared per job, with
`id-token: write` granted in exactly one job that actually uses OIDC.

**The credential rotation composite is genuinely good engineering.**
`[I].github/actions/linode-credentials/action.yml` defaults to dry-run, validates its own mode
arguments before doing anything (`:122-134`), emits `::add-mask::` in the same step that mints the
token (`:161-164`), refuses to cross a job boundary with raw material and passes a sha256 instead
with the reason recorded (`:180-184`), scrubs step summaries (`:277-289`), and writes the two OBJ
key halves secret-first so a mid-rotation read fails closed (`:253-256`). The `label` input's
description (`:25-33`) documents a real past incident — a non-discriminating literal that "drained
every LLZ instance on the Linode account" — as the reason the default changed. That is
scars-as-defaults done properly.

**The non-path gitleaks allowlists are a model of how to write suppressions.** Each of the four
(`:36-40`, `:46-50`, `:58-62`, `:76-80`) uses `regexTarget` rather than a path, states the exact
false-positive shape, and — critically — explains why the *broader* option was rejected. The
SealedSecret rule (`:64-80`) additionally documents the blocking impact that motivated it
(`gitleaks detect --source .` scanning all history on all refs).

**No placeholder leaked into a file that should be final.** Explicit verification scope for this
absence claim: `grep -rn "<@" .` and `grep -rn "@>" .` over the instance (excluding `.git/`) each
return **0 hits** — no unrendered copier delimiter anywhere. `grep -rn "REPLACE_"` returns 7 hits,
all correct: `REPLACE_INSTANCE_NAME` / `REPLACE_UPSTREAM_ORG` / `REPLACE_OWNER/REPLACE_REPO` appear
only in `landingzone.yaml.example:20,23,24` (an `.example` file, which is where they belong), and
every `REPLACE_PER_ENV` hit is *prose describing* the deferred state, not an unfilled slot —
`landingzone.yaml.example:62`, `landingzone.yaml:29`, `apl-values/README.md:91`,
`apl-values/values.yaml:77` are all comments. Zero placeholders in a final file.

**`deliver-docs` link rewriting works.** All 35 distinct relative Markdown link targets across
`[I]docs/`, `README.md` and `AGENTS.md` resolve to delivered files; template-only targets were
rewritten to upstream URLs. The pinned-docs pointer (`[I]docs/README.md:14`) and its rationale
(`ci_deliver_docs.go:18-27` — why cross-doc links deliberately track `main` while the pointer is
pinned) is a well-argued, cost-aware decision rather than a default.
