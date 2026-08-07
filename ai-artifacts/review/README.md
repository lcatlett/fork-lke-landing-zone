# LLZ Enterprise Architecture Review — 2026-08-07

End-to-end review of `lke-landing-zone` at `41cd28a`, cross-checked against the live
e2e instance `lke-landing-zone-example` (pinned to `e0fbc5c`).

**How to read this:** findings are owner-mode work items for Lindsey + Adam + other AS contibutors — LLZ is the template for all AS engagements, so a template fix
multiplies across every future instance. Each finding should carry (or will carry in
the consolidated backlog) a cause class:

- **repo-gap** — the primitive exists and isn't used → direct fix, with the gate that
  proves it stays fixed (per the repo's own name-the-gate doctrine) and an LoE.
- **product-gap** — Linode / LKE-E / managed apl-core lacks the primitive → the repo's
  posture is a workaround; the item becomes an escalation-ready product ask AS files
  and drives (e.g. OBJ versioning/replication, OpenBao backup primitives on managed,
  API quota transparency — cluster-create queues opaquely and no quota is
  discoverable, a confirmed known constraint).

## Files

| File | Lens | Status | Tiers (C/H/M/L) |
|---|---|---|---|
| [SUMMARY.md](SUMMARY.md) | Consolidated backlog: 8 Criticals, 3 sequenced waves, product-ask list, verification status | complete — **start here** | 8/62/105/51 total |
| [reliability.md](reliability.md) | Convergence machinery, failure modes, DR, state, scale, upgrades | complete (+ deep-dives in [parts/](parts/)) | 5/15/19/7 |
| [cicd.md](cicd.md) | Workflow security, release gates, publishing integrity, e2e robustness | complete | 0/5/10/4 |
| [code-quality.md](code-quality.md) | Go module structure, test posture, exit-code discipline | complete | 0/2/6/3 |
| [instance-delivery.md](instance-delivery.md) | Template claims vs the real delivered instance | complete | 0/3/7/4 |
| [security.md](security.md) | Terraform, charts, secrets lifecycle, multi-tenancy, supply chain (6 sub-reviews consolidated; repo-gap/product-gap classified) | complete | 1/17/30/25 |
| [docs.md](docs.md) | Docs completeness/accuracy vs the adopter + operator journeys | complete | 2/20/33/8 |
| [tooling-assessment.md](tooling-assessment.md) | Claude tooling: project / workspace / global recommendations | complete | n/a |

## Headlines confirmed firsthand (main-thread verification, not agent-only)

1. **OpenBao recovery quorum + root token transit the GitHub Actions step summary**
   (`tools/cmd/llz/ci_openbao_init.go:99-121`) — masked for logs, but the summary is
   the designed operator handoff channel; Actions-read on an instance repo can
   reconstitute root (3-of-5 threshold, all 5 shares shown). The sibling break-glass
   path already implements the fix pattern (RSA-OAEP artifact, 1-day retention).
2. **`docs/architecture/convergence-contract.md` §1 describes a retired gate** —
   `wait-apl-pipeline` has zero callers (no workflow, script, Makefile, or template
   reference; `ci_bootstrap_cluster.go` uses the weaker `waitManagedArgoReady`,
   lines 320, 402-406).
3. **Trivy config with no consumer; signing with no verifier** — `.trivyignore`
   exists at repo root while no workflow in either tree mentions trivy; no
   `cosign verify`/`verify-attestation` invocation exists in `.github/`,
   `instance-template/`, `tools/`, or `dockerfiles/`.
4. **Inert spec blocks with delivered guidance promising otherwise** — `spec.dns.*`,
   `spec.defaults.platform.*`, and `spec.alerting.*` are genuinely inert
   (`docs/landing-zone-spec.md:377-384` is the only accurate description;
   `RenderValues` retired per `tools/internal/clusterspec/values.go:3-10`;
   symbol-traced: zero renderer consumers of `AcmeEmail`, `Spec.Alerting` read only
   by `validate.go:56`). Stale comments in `render.go:217-219`/`types.go` and four
   files delivered to adopters — including the scaffold's seeded `landingzone.yaml`
   — still instruct operators to set them; `health/certs.go:83` then grades the
   silent no-op as an expected deferral. See instance-delivery.md for the full seam.

## Caveats

- Firewall/CIDR tooling moved to the private `lke-landing-zone-internal` repo and was
  out of scope.
