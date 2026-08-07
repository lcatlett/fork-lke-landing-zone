# LKE Landing Zone — Code Quality & Maintainability Review

Read-only review of `tools/` (Go module), `template-scripts/` (bash),
`instance-template/` composite actions, and the `Makefile` lint machinery.
Date: 2026-08-07. Commit: `41cd28a`. Toolchain: go1.26.5, shellcheck 0.11.0.

---

## Executive summary

This is a well-engineered repository by most measures that can be checked
mechanically: the full Go suite passes in **28.5s** (`go test ./... -count=1`,
exit 0), `go vet ./...` is clean, `gofmt -l .` prints nothing, `shellcheck -S
style` is clean across all 17 `.sh` files, `go.mod` carries **5 direct
dependencies and zero `replace` directives**, and there is **zero deferred-work
debt** — 27 `TODO|FIXME|XXX|HACK` grep hits across 731 `.go` files, none of
which is a deferred-work marker. The "scars as defaults" convention is real and consistently
applied — `main.go:104-130` and `ci_health.go:74-81` are exemplary.

The dominant structural issue is that **`cmd/llz` holds 70,413 of the module's
81,648 non-test lines (86%) in a single `package main` of 241 non-test files**,
while the repo's own stated convention (`tools/cmd/llz/ci.go:7-9`,
`tools/AGENTS.md:8-10`, `.untestable-budget.yaml:6-8`) is that decision logic
lives in `internal/*` behind unit tests with `cmd/llz` as "the thin
orchestration around it." The `internal/*` packages total 11,235 non-test lines. That drift has a
measurable, not theoretical, cost: because Go coverage is per-package, the
`COVERAGE_MINS` ratchet can express exactly **one floor for 86% of the code** —
and that floor is set to **48% while actual is 71.1%**, leaving 23 points of
dead slack. Inside that slack sit the credential-provisioning paths:
`tokens.go` at **26.0%**, `regenroot.go` at **19.1%**, `openbao.go` at
**38.1%**.

Notably, the review brief's assumption that health classification and render
are under-tested is **refuted by measurement**: `internal/health` is at 97.1%,
`internal/clusterspec` (the spec→tfvars engine behind `render`) at 95.9%,
`internal/terraform` at 100%. Credential provisioning is the real gap.

**Findings: 0 Critical / 2 High / 6 Medium / 3 Low.**

---

## Findings

### 1. `cmd/llz` is 86% of the module in one package, against the module's own stated convention

**Impact: High | LoE to fix: High**

**Evidence**

| Tree | Non-test LOC | Non-test files |
|------|-------------:|---------------:|
| `tools/cmd/llz` | 70,413 | 241 |
| `tools/internal/**` (16 pkgs) | 11,235 | ~139 |

- `tools/cmd/llz/ci.go:7-9` states the convention: *"the decision logic lives in
  `internal/terraform` (+ `internal/linode`) behind unit tests, and this file is
  the thin terraform/Linode orchestration around it."* Restated twice more:
  `tools/AGENTS.md:8-10` (`cmd/llz` *"Orchestrates the existing setup / upgrade
  flow … it does not reimplement them"*) and `.untestable-budget.yaml:6-8`
  (*"The design principle of this repo … is that decision-making logic belongs
  in unit-tested Go"*).
- `go list ./...` returns 17 packages; `cmd/llz` alone contains 588 `.go` files
  (241 non-test + 347 test).
- Largest files, all in `cmd/llz`: `ci_health.go` (1563), `ci.go` (1396),
  `ci_bootstrap_cluster.go` (1308), `commands.go` (1202), `import.go` (1135),
  `ci_docs_guard.go` (1051), `ci_plaintext_guard.go` (981).
- `tools/internal/apl/apl.go:1-17` documents ADR 0013's intended target package
  map and calls the current state *"a seam, not a migration: the package is
  intentionally near-empty."* So the drift is **known and has a plan** — this
  finding measures the plan's remaining distance, not its absence.

**Why it matters**

Three concrete costs, each measured elsewhere in this report:

1. Coverage is unmeasurable per-area (Finding 2) — Go reports one number per
   package, and 86% of the code is one package.
2. ADR 0013's boundary rule (`internal/apl` MUST NOT import `internal/provider`,
   `internal/apl/apl.go:7-11`) is enforceable by the compiler only for code that
   has already moved. Everything in `package main` is outside the rule.
3. Navigation: `ciCmd()` at `ci.go:29-80+` registers ~60 subcommands spanning
   terraform, OpenBao, Harbor, Keycloak, rotation, teardown and readiness in one
   function body. It is a single append-only point that every new verb touches.

**Recommendation**

Do not attempt a big-bang split. ADR 0013 already names the target map —
convert it into a **ratchet the build enforces**, matching this repo's existing
idiom: add an `llz ci` guard asserting `cmd/llz` non-test LOC only ever
decreases (identical shape to `.untestable-budget.yaml`), seeded at today's
70,413. Then extract in the order coverage says hurts most (Finding 4):
`tokens.go`/`wizard.go` → `internal/credentials`, `openbao*.go` →
`internal/openbao` (which already exists at 90.2%), `reap.go` decision logic →
`internal/linode` (which already holds the heuristics per `reap.go:5-7`). Each
extraction converts an opaque slice of the 48% floor into a gateable package.

---

### 2. The `cmd/llz` coverage floor is 48% against 71.1% actual — 23 points of dead ratchet

**Impact: High | LoE to fix: Low (raise floor) / Medium (add tests)**

**Evidence**

- `Makefile:29-39` — `COVERAGE_MINS := cmd/llz=48 internal/cli=95
  internal/clusterspec=95 internal/health=95 internal/kube=78 internal/linode=80
  internal/metrics=95 internal/openbao=88 internal/preflight=95
  internal/terraform=95`.
- Measured (`go test ./cmd/llz -count=1 -coverprofile=…`; `go tool cover -func`):
  **`cmd/llz` total = 71.1%** (16,633 / 23,382 statements).
- The 48% floor therefore permits deleting **~5,400 covered statements** — more
  than the entire `internal/*` tree's statement count — while staying green.

Per-file coverage inside that slack (statements covered / total), lowest-covered
files ≥80 statements:

| File | Coverage | Stmts | What it does |
|------|---------:|------:|--------------|
| `env_set.go` | 13.3% | 15/113 | spec WRITE side (`env set`/`env edit`/`network add`) |
| `reap.go` | 14.0% | 34/242 | orphan Linode resource deletion (`--yes` deletes) |
| `ci_obj_encryption_harbor.go` | 15.7% | 19/121 | SSE-C encryption proof |
| `regenroot.go` | 19.1% | 27/141 | OpenBao root-token quorum regeneration |
| `components_cmd.go` | 21.2% | 17/80 | component enable/disable |
| `verify.go` | 22.7% | 22/97 | post-bootstrap acceptance snapshot |
| `tokens.go` | 26.0% | 84/323 | **credential provisioning wizard** |
| `ci_keycloak_smoke.go` | 27.8% | 80/288 | OIDC login smoke |
| `openbao.go` | 38.1% | 53/139 | KV v2 get/set (dual-write + rollback) |
| `ci.go` | 43.7% | 219/501 | terraform/Linode CI orchestration |
| `wizard.go` | 48.0% | 95/198 | token gather wizard |

**Why it matters**

The ratchet is the mechanism this repo relies on to keep test debt monotonically
decreasing (`.untestable-budget.yaml:13-16` states the doctrine explicitly:
*"These budgets are a RATCHET: lower them as logic moves into the CLI; never
raise one"*). A floor 23 points below actual is not a ratchet — it is a number
that cannot fail. `reap.go` and `openbao.go` in particular are **destructive**
paths (`reap` deletes Linode resources with `--yes`; `openbao set` dual-writes
with rollback) sitting at 14% and 38%.

**Recommendation**

Two moves, both cheap:

1. Raise `cmd/llz` to **70** today (`Makefile:30`). It costs nothing, it is
   below actual, and it immediately makes a regression in any of the files above
   visible. Repeat on every PR that improves coverage, per the file's own rule.
2. Add the missing per-area floors as the extraction in Finding 1 proceeds —
   each extracted package gets its own line in `COVERAGE_MINS`, which is the
   only way these numbers ever become individually gateable.

---

### 3. Seven of 17 packages are outside `COVERAGE_MINS` entirely

**Impact: Medium | LoE to fix: Low**

**Evidence**

`Makefile:29-39` gates 10 packages. Measured coverage of the 7 it does not:

| Ungated package | Coverage | Notes |
|---|---:|---|
| `internal/forge` | **72.9%** | multi-forge abstraction (github.com / GHE.com / GHES) — `internal/forge/flavors.go:3-12` |
| `internal/apl/overlay` | 82.8% | apl-values overlay rendering |
| `internal/apl/identity` | 90.3% | |
| `internal/tfroots` | 90.0% | embedded TF roots |
| `internal/validate` | 97.9% | |
| `internal/provider` | *(no statements)* | ADR 0013 interface-only seam |
| `internal/apl` | **no test files** | doc-only seam package (`internal/apl/apl.go`) — benign today |

**Why it matters**

`internal/forge` is the lowest-covered package in the whole module and is
load-bearing for the GHES/GHE.com adopter path — precisely the path with the
fewest e2e runs behind it. `internal/apl/overlay` at 82.8% renders what every
instance's GitOps repo consumes. Neither can regress into a red build today.

`ci_coverage_guard.go:86-90` already fails on a package with *no coverage data*
("the package was renamed, removed, or its tests did not run") — so the guard is
designed to notice absence. It just is not pointed at these packages.

**Recommendation**

Add all seven to `COVERAGE_MINS` at their measured value rounded down
(`internal/forge=72 internal/apl/overlay=82 internal/apl/identity=90
internal/tfroots=90 internal/validate=97`). Skip `internal/apl` and
`internal/provider` while they remain statement-free seams, and add a one-line
comment in the Makefile saying so — otherwise the next reader adds them and gets
the `no coverage data` error from `ci_coverage_guard.go:86`.

---

### 4. Credential provisioning is the least-tested load-bearing area (32.8%)

**Impact: Medium | LoE to fix: Medium**

**Evidence** — area rollups computed from the `cmd/llz` coverage profile
(statements covered / total):

| Area (file prefixes) | Coverage | Stmts |
|---|---:|---:|
| `tokens*` / `wizard*` / `secrets*` | **32.8%** | 179/545 |
| `ci_assert_*` | 71.5% | 1,987/2,779 |
| `upgrade*` / `scaffold*` / `template_*` | 72.1% | 431/598 |
| `render*` | 76.5% | 248/324 |
| `openbao*` / `ci_bao_*` | 78.0% | 798/1,023 |
| `credentials*` / `ci_rotat*` / `runner_acl*` | 79.4% | 935/1,177 |
| `ci_health*` | 79.6% | 626/786 |
| `import*` | 81.2% | 1,158/1,426 |

Cross-checked against the packages those orchestrators sit on:
`internal/health` **97.1%**, `internal/clusterspec` **95.9%**,
`internal/terraform` **100%**, `internal/metrics` **100%**,
`internal/preflight` **100%**, `internal/openbao` 90.2%.

**Why it matters**

This **refutes** the common assumption that health classification and render are
the risk. They are the two best-tested things in the module, exactly because
their decision logic was extracted per the `tools/AGENTS.md` convention. The
untested area is the one where that extraction has *not* happened:
`tokens.go:1-15` describes a command that creates an OBJ bucket, mints a scoped
key via the Linode API, gathers GitHub PATs, and writes `.llz/*.env` — all
inside `package main`, at 26% coverage. A silent regression here produces an
instance whose credentials are subtly wrong, and the failure surfaces minutes
later inside a Terraform apply in CI.

**Recommendation**

Highest-value first extraction for Finding 1. Move the *decisions* out of
`tokens.go` / `wizard.go` — which PAT scopes are required, which steps are
already satisfied (the idempotence check described at `tokens.go:9-11`), how
image vars are computed — into an `internal/credentials` package with the
`internal/health` treatment: pure functions over parsed input, transport behind
a package-var seam. `tools/AGENTS.md:49-61` already specifies exactly this shape
and names `ci_assert_scrape.go` / `ci_assert_openbao_audit.go` as the models.

---

### 5. `os.Exit(2)` inside a library package collides with the convergence contract

**Impact: Medium | LoE to fix: Low**

**Evidence**

```go
// tools/internal/cli/cli.go:34-41
// MustUint parses a uint64 flag value, exiting(2) on a malformed number.
func MustUint(s string) uint64 {
	n, err := strconv.ParseUint(s, 10, 64)
	if err != nil {
		fmt.Fprintf(os.Stderr, "invalid unsigned integer %q\n", s)
		os.Exit(2)
	}
	return n
}
```

- Exit **2** is a load-bearing value: `ci_health.go:55` — *"exit 0 converged / 2
  in-progress / 1 hard-failed / 3 unreachable"*; `ci_health.go:74-81` documents
  that `llz ci converge` branches on it and that 2 means *"keep polling."*
- One caller today: `cmd/llz/credentials_lkeadmin.go:91`
  (`clusterID := cli.MustUint(clusterIDArg)`), which is outside the converge
  tree. So this is **latent, not live.**
- Everything else about exit-code discipline is correct: 9 `os.Exit` sites
  module-wide, 5 of them carrying explicit `DELIBERATE os.Exit` rationale
  (`ci_health.go:74-81`, `ci_health_incluster.go:49-56`), plus a contract
  validator at `ci_health.go:291-292` that rejects any code outside 0/1/2/3.
  Exactly **one** `panic()` in non-test code (`internal/tfroots/tfroots.go:77`,
  over `embed.FS` walking — unreachable at runtime). `main.go:43-46` collapsing
  all returned errors to exit 1 is correct here, because the only verbs with a
  richer contract bypass it via their own `os.Exit`.

**Why it matters**

`internal/cli` is a shared package (`internal/cli/cli.go:1-4` says it is used by
*"the credential-rotation commands"*, plural). The first time a converge-adjacent
verb parses a numeric flag through `MustUint`, a typo'd flag becomes "in
progress — keep polling" and the converge loop burns its full `--budget 1800`
before reporting anything. It will look like a cluster that never converged.

**Recommendation**

Change the signature to `func Uint(s string) (uint64, error)` and let
`credentials_lkeadmin.go:91` return the error into cobra. If a `Must` variant is
kept for ergonomics, it must not choose an exit code — libraries return, mains
exit. This is a ~10-line change with one call site.

---

### 6. Package-level mutable global (`gopts`) is read by RunE closures across 241 files

**Impact: Medium | LoE to fix: Medium**

**Evidence**

- `cmd/llz/main.go:32-40` — `type globalOpts struct { dryRun, open, yes bool }`;
  `var gopts globalOpts`, populated from persistent flags at `main.go:60-63`.
- Read directly inside RunE closures throughout, e.g. `main.go:155`
  (`runNew(gopts, …)`), `main.go:196` (`runTokens(gopts, …)`), `main.go:202`
  (`if gopts.yes && !gopts.dryRun`), `main.go:249`, `main.go:267`, `main.go:439`,
  `main.go:454`, `main.go:512`, `main.go:547`, `main.go:577`.
- `gopts.yes` is the gate on **every cloud-mutating command** (`main.go:13`,
  `tools/AGENTS.md:12-13`).

**Why it matters**

Two costs. First, testability: any test exercising a `--yes` path must mutate a
process-global, so those tests cannot use `t.Parallel()` and can leak state into
each other — in a package whose suite already runs 12.5s, that constraint
compounds. Second, safety: the single most consequential boolean in the CLI
("actually mutate cloud resources") is ambient rather than passed, so a future
subcommand that forgets to consult it fails open with no compiler signal.

**Recommendation**

`newRootCmd()` already constructs the tree; have it construct a `*globalOpts`
and close over it, passing the pointer into each `*Cmd()` constructor. Purely
mechanical, no behavior change, and it makes `--yes` a parameter the type system
can track. If a broader change is unwelcome, the narrower fix is to route every
mutation through one `requireYes(gopts) error` helper so the check has a single
call site that a guard can assert on.

---

### 7. kubectl transport is duplicated: 146 call sites, three near-identical helpers in three unrelated files

**Impact: Medium | LoE to fix: Medium**

**Evidence**

- 146 occurrences of the literal `"kubectl"` across 25 non-test files in
  `cmd/llz`. Densest: `ci_diagnose_argocd.go` (21),
  `ci_assert_broad_pat_rotation.go` (12), `ci_assert_network_enforcement.go` (8),
  `ci_wait.go` (7), `ci_assert_health_workflow.go` (7),
  `ci_kick_harbor_provisioner.go` (6), `ci_health.go` (5).
- Three separate wrappers, each defined in a file about something else, each
  with a different signature:
  - `kubectlNames(args ...string) []string` — `ci_diagnose_argocd.go:366`
  - `kubectlCtx(kubeconfig, context string, args ...string) (string, error)` — `import.go:213`
  - `kubectlOut(args ...string) (string, error)` — `verify.go:160`
- Only 8 files use any of them; 59 `exec.Command` sites in `cmd/llz` non-test
  code build invocations inline.
- `internal/kube` exists (785 LOC, 86.8% covered) but is the in-cluster REST
  client, not the shell-out path — `tools/AGENTS.md:45-47` records the deliberate
  "no client-go" decision, so shelling out is the intended pattern; it is the
  *lack of one seam for it* that is the issue.

**Why it matters**

`tools/AGENTS.md:49-54` mandates that transport sit "behind a **package-var
seam** a test replaces," naming `withPrometheus` / `withLoki` as the models.
Those seams exist and work. kubectl — the most-used transport in the module —
has no equivalent, which is a direct contributor to the low coverage on the
kubectl-heavy files in Finding 2 (`verify.go` 22.7%,
`ci_assert_health_workflow.go` 42.9%, `ci_assert_network_enforcement.go` 51.9%).
It also means a cross-cutting change (adding `--request-timeout`, threading a
context, adding retry) is a 146-site edit.

**Recommendation**

Add one `kubectl.go` in `cmd/llz` defining a single `var runKubectl = func(ctx
context.Context, args ...string) ([]byte, error)` seam plus the two or three
shapes callers actually need (raw bytes, JSON-decoded, name list). Fold
`kubectlNames`/`kubectlCtx`/`kubectlOut` into it and migrate call sites
opportunistically — a lane at a time, as each lane's tests are written. Do not
do it as one sweep; migrate a file only when adding its test, so the change pays
for itself in coverage each time.

---

### 8. Three untestable-loc budget categories have ≤1 line of headroom

**Impact: Medium | LoE to fix: Low**

**Evidence** — `llz ci untestable-loc`, run at `41cd28a`:

```
embedded-shell-in-yaml        22 / 23     ok    (1 line headroom)
makefile-recipe               27 / 28     ok    (1 line headroom)
python-scripts                 0 / 0      ok
shell-scripts                366 / 375    ok    (9 lines)
terraform-provisioner-bash     0 / 0      ok
workflow-inline-bash         522 / 526    ok    (4 lines)
```

`.untestable-budget.yaml:57-59` states the intent: *"Budgets carry ~3% headroom
over baseline."* Three categories are now well inside that: 4.3%, 3.6% and 2.4%
of budget respectively for `embedded-shell`, `makefile-recipe` and
`workflow-inline-bash` — but in absolute terms 1, 1 and 4 lines.

**Why it matters**

This is a ratchet working as designed, and the design is sound — `python-scripts
0 / 0` and `terraform-provisioner-bash 0 / 0` are genuine wins the repo should
protect, and `.claude/skills/preflight/SKILL.md` correctly warns that `0 / 60`
means "no Python in the tree," not "60 lines of room." The maintainability
signal is different: at 1 line of headroom, a legitimate two-line addition to a
`ConfigMap` script forces an unrelated conversion in the same PR. That converts
a ratchet into an ambush, and the documented escape (`exclude:` with a
justification) will start getting used for things that are not genuine glue.

**Recommendation**

Do not raise the budgets. Instead, spend the headroom deliberately once: pick
the largest remaining `embedded-shell-in-yaml` and `makefile-recipe` blocks and
convert them to `llz ci` verbs in a dedicated PR, then **lower** the budgets to
the new baseline plus ~3 lines. That restores working room without weakening the
ratchet — and it is the move `.untestable-budget.yaml:13-16` prescribes.

---

### 9. 44 of 241 non-test files in `cmd/llz` have no `_test.go` sibling

**Impact: Low | LoE to fix: Medium**

**Evidence** — 197 of 241 non-test files (82%) have a same-named `_test.go`
sibling; 44 do not. Test-to-source ratio for the module overall is healthy:
75,171 test LOC against 81,648 non-test LOC (0.92:1), and 347 test files against
241 source files in `cmd/llz`.

**Why it matters**

Low severity because the naming convention is not load-bearing — several of
these files are covered by a differently-named test (e.g. `credentials_test.go`,
`build_preflight_test.go`, `commands_push_repo_test.go`,
`objproxy_resign_test.go` each cover more than their prefix). The signal is only
useful as a cross-check against Finding 2's per-file numbers, where the two
overlap.

**Recommendation**

No standalone action. Use the per-file coverage table in Finding 2 as the work
queue instead — it measures the thing this proxy approximates.

---

### 10. `Makefile` is 992 lines / 81 targets

**Impact: Low | LoE to fix: Low**

**Evidence** — `Makefile` 992 lines, 56,889 bytes, 81 rule targets. The `help:`
target alone spans `Makefile:41-118`. Roughly two dozen named guards compose
`make lint`.

**Why it matters**

Large, but genuinely managed rather than sprawling: the `LLZ_CI` macro keeps
guard recipes to one logical line each (which is also why `makefile-recipe`
scores only 27 in Finding 8 — the counter explicitly excludes
single-logical-line glue, `.untestable-budget.yaml:34-37`), `make help`
documents every target, and each guard's comment block records the outage it was
built against. The residual cost is discoverability: 81 targets is past the point
where a newcomer can hold the set in their head, and `make help` is a flat wall.

**Recommendation**

Group `make help` output under the same headings the CLI already uses
(`main.go:76-81` groups cobra commands into "Author & deploy" / "Provision,
build & operate" / "Day-2 & maintenance"). Mirroring that grouping in `help:`
costs a handful of `@echo` lines — and per `.untestable-budget.yaml:38-44`,
pure-`@echo` lines are explicitly not charged to the budget, so this is free
against the ratchet.

---

### 11. `ci.go`'s `ciCmd()` registration is an append-only 60-subcommand function

**Impact: Low | LoE to fix: Low**

**Evidence** — `cmd/llz/ci.go:29-80+` registers roughly 60 subcommands across
terraform, OpenBao, Harbor, Keycloak, rotation, teardown, readiness and wait
primitives in one function body, interleaved with block comments explaining which
are callerless break-glass handles vs. workflow-driven (`ci.go:44-46`,
`ci.go:51-52`, `ci.go:71-76`, and a whole `── BREAK-GLASS VERBS ──` section at
`ci.go:94`). `ci.go` itself is 1,396 lines at 43.7% coverage
(219/501 statements) — the largest low-covered orchestration file in the module.

**Why it matters**

The comments are genuinely valuable — they encode which verbs are live vs.
break-glass, which `make deadcode` (report-only by design, per
`.claude/skills/preflight/SKILL.md`) cannot tell you. But every new `llz ci` verb
edits the same function, making it a persistent merge-conflict point across
concurrent branches, and the grouping comments have no mechanism keeping them
true as verbs come and go.

**Recommendation**

Split registration by concern into `ciCommandsTerraform()`,
`ciCommandsOpenBao()`, `ciCommandsAssert()`, etc., each in its existing topical
file, with `ciCmd()` reduced to calling them. Then the live-vs-break-glass
annotation moves next to the verb it describes, where the person retiring the
verb will see it.

---

## What's done well

These are not filler — each is a measured result, and several are things most
repositories of this size get wrong.

**Test suite speed and health.** `go test ./... -count=1` completes in **28.5
seconds** for 157k LOC and exits 0. `go vet ./...` clean. `gofmt -l .` prints
nothing. A suite this fast is the reason the gate system works at all — a slow
suite gets skipped, and this one cannot be.

**Dependency hygiene — the strongest signal in the repo.** `tools/go.mod` has
**5 direct dependencies** (`spf13/cobra`, `golang.org/x/crypto`,
`golang.org/x/term`, `gopkg.in/yaml.v3`, `sigs.k8s.io/yaml`), 4 indirect, and
**zero `replace` directives**. `go list -m -u all` shows only minor point
releases available on indirect deps. `tools/AGENTS.md:45-47` records the
deliberate decision to hand-roll an in-cluster REST client rather than pull in
client-go — a choice that, on its own, avoids the ~90-module transitive tree that
dominates most Kubernetes tooling. Of every decision visible in this codebase,
this is the one that will still be paying off in five years.

**`internal/*` coverage.** Where the extraction convention has been followed, the
results are excellent: `internal/terraform` 100%, `internal/metrics` 100%,
`internal/preflight` 100%, `internal/validate` 97.9%, `internal/health` 97.1%,
`internal/clusterspec` 95.9%, `internal/cli` 96.0%. These are the packages
Finding 1 argues the rest should look like — the argument is that the pattern
*works*, not that it is unproven.

**Exit-code discipline.** Nine `os.Exit` sites module-wide, five carrying an
explicit `DELIBERATE os.Exit` comment naming the caller that distinguishes the
codes (`ci_health.go:74-81` is the model — it names the exact collapse it
prevents: *"Returning an error would collapse 2 and 3 into cobra's exit 1 and
turn every transient into an immediate hard failure"*). A runtime contract
validator at `ci_health.go:291-292` rejects any code outside 0/1/2/3. Exactly one
`panic()` in non-test code, over an `embed.FS` walk that cannot fail at runtime
(`internal/tfroots/tfroots.go:77`). Finding 5 is the only crack in this.

**Bash quality — clean, and deliberately so.** `shellcheck -S style` (the
strictest non-pedantic level) exits **0** across all 17 `.sh` files with zero
findings. Shebangs are uniformly `#!/usr/bin/env bash`. `set -euo pipefail` is
present in all 13 executed scripts; the three exceptions are each correct and
documented: `template-scripts/lib-common.sh:1` is a sourced library declaring
`# shellcheck shell=bash`,
`instance-template/.github/actions/_lib/git-auth.sh:23-25` documents that it is
*"Sourced (not exec'd)"*, and `template-scripts/ci/with-retry.sh` uses `set -uo
pipefail` because a retry wrapper must survive a failing command. Shared helpers
are genuinely shared — 6 of the 7 CI scripts source `lib-common.sh` rather than
re-implementing `die` / `step` / `fail` / `usage`.

**Composite actions are thin.** All 6
`instance-template/.github/actions/*/action.yml` carry 5 or fewer `run:` steps
(max: `linode-credentials` at 5; `cluster-access` at 0). The logic is in `llz`,
which is what `.untestable-budget.yaml` exists to force and what
`workflow-inline-bash 522/526` confirms is holding.

**Zero TODO debt.** 27 grep hits for `TODO|FIXME|XXX|HACK` across 731 `.go`
files, and **not one is a deferred-work marker**. The breakdown: 23 are the
`MIGRATION-TODO.md` product feature (`import_init.go:116`, `:279`) or the local
variable named `todo` that renders it (`scaffold.go:337-354`); 3 are `✗ TODO`
UI strings printed by the readiness table (`readiness.go:196`, `:208`, `:229`);
1 is a comment *about* how a guard classifies ownership
(`ci_plaintext_guard.go:59`). For a 157k-LOC codebase that is remarkable, and
it is the kind of thing that only stays true if someone is actively keeping it
true.

**"Scars as defaults" is real, not aspirational.** The convention at
`AGENTS.md:37-39` is applied consistently and specifically. `main.go:104-130`
spends 27 lines explaining why cobra's `legacyArgs` validator lets an unknown
subcommand under a group exit 0, which outage that caused (a stale `llz` binary
made a readiness gate "succeed" in 0s and raced Argo CD CRDs into a hard
failure), and why `NoArgs` alone is insufficient. `main.go:346-354` documents a
*deprecated flag that writes nothing* and why it survived — *"It survived as a
flag that echoed the value back in the summary banner — which read exactly like
it had been applied."* `ci_health.go:114-124` explains a removed memoization and
the measurement that justified removing it. These comments answer the question a
future reader will actually have, which is the hard part.

**The gate system is coherent.** `ci_assert_suite.go:72-203` is a single lane
table where each lane carries a `Why` field stating what it proves *and what
stays green without it* — with the structural note that one list means the
"declared but never checked" hazard is gone by construction
(`ci_assert_suite.go:72-74`). The 26 `ci_assert_*` lanes are **not** copy-paste:
each has a distinct pure evaluator (`judgeVolume` at
`ci_assert_volume_encryption.go:138`, `evalCertificates` at
`ci_assert_certificates.go:100`) with the transport behind a seam, which is
exactly the shape `tools/AGENTS.md:49-61` prescribes. Duplication across lanes is
low; the real duplication is the kubectl transport (Finding 7), not the lane
logic.

**The ratchet machinery itself.** `ci_coverage_guard.go:86-95` fails on a package
with *no* coverage data, not just a low number — catching the "package renamed,
tests silently stopped running" failure that most coverage gates miss entirely.
`.untestable-budget.yaml:18-56` documents its counting rules precisely enough to
be reproducible, and explains *why* each exclusion exists (charging for `@echo`
lines "priced DOCUMENTATION at the rate of logic," so documenting a new target
became the expensive choice — a genuinely subtle incentive bug, found and fixed).

---

## Appendix — how the numbers were produced

All commands run read-only from `tools/` at commit `41cd28a`; no repo file was
modified (verified with `git status --porcelain`).

```bash
go test ./... -count=1 -cover                                   # 28.5s, exit 0
go test ./cmd/llz -count=1 -coverprofile=<scratch>/cover.out
go tool cover -func=<scratch>/cover.out                         # 72.9% (this run)
# per-file / per-area rollups aggregated from cover.out statement counts
go vet ./... ; gofmt -l .                                       # both clean
go list -m -u all
find . -name '*.go' -not -name '*_test.go' | xargs wc -l
shellcheck -S style $(find . -name '*.sh')                      # exit 0
go build -o <scratch>/llz ./cmd/llz && <scratch>/llz ci untestable-loc
```

Note: `go tool cover -func` reported 72.9% for a standalone `cmd/llz` run vs.
71.1% aggregated from the profile's statement counts; the small delta is
`-func`'s per-function averaging. The 71.1% figure (16,633/23,382 statements) is
the one used throughout, as it is what `check-coverage` computes.
