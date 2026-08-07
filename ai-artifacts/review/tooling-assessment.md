# Tooling Assessment — Claude skills/agents/plugins for enterprise review of lke-landing-zone

Date: 2026-08-07 · Scope: what tooling exists, what to add, and at which layer.

## What the repo already ships (baseline — this is unusually strong)

| Asset | What it covers |
|---|---|
| 12 project skills (`.claude/skills/`) | Lifecycle-shaped: `preflight`, `gate`, `add-ci-guard`, `branch-base`, `credential-change`, `delivered-surface`, `docs`, `e2e-triage`, `netpol-change`, `onboard-adopter`, `release`, `rotate-credentials` |
| `template-hygiene-reviewer` agent | Read-only pre-PR diff review of the conventions CI can't machine-check (org-identity hardcoding, prefixes, scars-as-defaults, nested AGENTS.md rules) |
| Hooks | `block-sensitive-files.sh` (PreToolUse on writes), `format-on-edit.sh` (PostToolUse) |
| Permissions | Curated read-only/lint allowlist in `.claude/settings.json` — no broad Bash grant |
| `.mcp.json` | Terraform MCP server, docker-run, **SHA-pinned** — consistent with the repo's pinning doctrine |
| `instance-template/.claude/settings.json` | Delivered to adopters — Claude tooling is part of the delivered surface |

The skills are failure-class-shaped (each encodes scars from a real regression class), not generic how-tos. This is the right pattern; recommendations below extend it rather than replace it.

## Recommended additions — project level (`.claude/` in the repo, PR to Adam)

1. **`spec-change` skill** (highest value). The LandingZone spec is the widest-fan-out surface in the repo after credentials: one edit touches the schema structs, `Validate`, three render targets (tfvars, kustomization overlay, apl-overlay `apps.yaml`), `llz render --check` drift guards, `llz components`' live table, and the docs field reference — and drift on this surface is proven: the spec carries a deprecated field (`templateVersion`) that only ever drifted, and the `spec.dns.*`/`spec.alerting.*` blocks are genuinely inert (docs/landing-zone-spec.md:377-384 is the only accurate description) while stale code comments (`tools/cmd/llz/render.go:217-219`, `types.go:119-135`) and four delivered files — including the scaffold's seeded `landingzone.yaml` guidance — still promise rendering that was retired with `RenderValues` (`tools/internal/clusterspec/values.go:3-10`; symbol-traced this session: zero consumers of `AcmeEmail`, `Spec.Alerting` read only by `validate.go:56`). `credential-change` exists for exactly this reason on the credential surface; the spec surface deserves its twin.
2. **`security-reviewer` agent** — a read-only counterpart to `template-hygiene-reviewer` with a security lens: NetworkPolicy deltas (seeded from the Cilium/LKE-E traps in `netpol-change`), RBAC/secret-flow changes (seeded from `credential-change`'s fan-out map), and workflow-permissions deltas. Hygiene and security are different review questions; keeping them as separate agents keeps each prompt sharp.
3. **`convergence-change` skill (optional, medium value)** — anyone touching `ci_health`/`ci_converge`/`ci_bootstrap_cluster` must preserve the 4-exit-code contract; today that doctrine lives in `docs/architecture/convergence-contract.md` and partially in `gate`/`e2e-triage`. A thin skill that triggers on those files and front-loads the anti-pattern list would prevent the highest-cost class of regression. Could alternatively be folded into `gate`.

## Recommended — Lindsey's workspace level (this fork checkout, NOT PR'd)

1. **`settings.local.json` review allowlist** — add read-only scanner permissions used during evaluation sessions (`checkov`, `trivy config`/`trivy fs`, `kics`, `gh api`/`gh run list` read paths) so future review passes don't stall on prompts. Local because Adam's committed allowlist should stay minimal.
2. **deep-wiki generation (already installed globally)** — run `/deep-wiki:generate` against this checkout once the adoption decision is live; a browsable wiki with onboarding guides is the fastest way to ramp NBA-project collaborators without touching Adam's repo.
3. **Review artifacts location** — keep review outputs in `ai-artifacts/review/` in this checkout (searchable with `rg <pattern> ai-artifacts/review/`); this fork's auto-memory index at `~/.claude/projects/-Users-lcatlett-projects-akamai-lke-fork-lke-landing-zone/memory/MEMORY.md` already points at active workstreams and should gain a line pointing here.

## Recommended — global level

1. **A reusable `platform-review` skill** (write to `~/.claude/skills/platform-review/SKILL.md`) capturing this session's methodology: orient on architecture docs → 5-lens parallel agent fan-out (security, reliability, quality, CI/CD, docs) → consolidated LoE/Impact/sequencing report. You review enterprise platforms repeatedly (this repo now, NBA infra next); the methodology is repo-agnostic and currently lives only in this session's transcript — encoding it as a skill makes it re-runnable with one command. Everything else needed (multi-model second opinions via `pal` consensus/codereview, `/code-review`, `/security-review`, visual-explainer for shareable diagrams) is already installed globally — no new global installs required.
2. **Nothing else.** The gap is not missing global tools; it's that the review methodology exists only in this session's transcript until item 1 is done.

## Explicitly NOT recommended

- A Kubernetes/Helm docs MCP at project level — the repo's lint stack (`kubeconform`, `kube-linter`, `helm lint --strict`) already machine-checks what such a server would advise on; it would add supply-chain surface for marginal value.
- Any marketplace plugin at project level — the repo's in-place skills/agents pattern is deliberate (delivered to adopters via instance-template); marketplace-cache plugins wouldn't ship to instances.
