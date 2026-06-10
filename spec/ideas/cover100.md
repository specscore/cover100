---
format: https://specscore.md/idea-specification
status: Specified
---

# Idea: cover100 — verified test coverage to 100% as a Claude Code plugin

**Status:** Specified
**Date:** 2026-06-10
**Owner:** trakhimenok
**Promotes To:** cover100-skill-ux, go-language-profile, language-profile-contract, orchestrator-core, plugin-packaging-and-distribution, python-language-profile, rescan-mode, run-report, typescriptjavascript-language-profile
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we turn a battle-tested coverage-to-100% agent workflow into a standalone, multi-language Claude Code plugin that becomes the default way AI-native teams make tests prove their code?

## Context

Extracted from the go-coverage-100 workflow built and battle-tested on sneat-go (18.8% to 100% on a ~38k-uncovered-statement module). The harness engineering is the moat: deterministic phase orchestration (Salvage, Collect, Classify, Research, Helpers, Plan units, Cover with an engineer-verifier worker pool, Finalize, Harvest, Overview), adversarial verification with hard gates (build / meaningful / seam-only / lint / fmt), no-work-lost design (per-package commits, abort salvage, sticky units, resume), kind-affinity unit packing, a standard-fakes guide, and a rescan mode that audits and refactors existing tests. Thesis: tests are executable specs - cover100 proves code honors intent, complementing SpecScore. Audience: AI-native dev teams (clean up AI-generated test slop), OSS maintainers (merge PRs without fear), solo devs and consultants (catch up on TDD; sell a 100%-coverage engagement).

## Recommended Direction

Ship an MIT-licensed Claude Code plugin /cover100 with two marketed modes from day one: drive-to-100 (every package reaches 100% statement coverage or carries a documented, verifier-challenged why-not gap) and rescan (audit existing tests for hand-rolled fakes that duplicate stdlib or shipped mocks, copy-paste tests that should be table tests, and dead helpers - then refactor without dropping coverage). Languages via profiles: Go (done, hardest case - no monkeypatching, so seam policy exists) and TypeScript/JavaScript (vitest/jest, istanbul statement coverage, tests-only policy; scoped support matrix) at launch — the founder's own stack and the largest active-OSS audience; Python (pytest + coverage.py, tests-only) is the stretch third language, included if timing allows, else first fast-follow. The orchestrator core stays language-agnostic. Subtree-first UX: the default motion is /cover100 ./pkg/auth/... - minutes and cents with an upfront cost projection; whole-repo is pro mode behind a confirmation. Every run ends with REPORT.md: executive summary (coverage before/after, tests added, suspected bugs found, documented gaps, spend), code-review observations, and the gap register - the shareable artifact that markets the tool. codegraph CLI is a hard requirement for symbol intelligence.

## Alternatives Considered

- **Coverage-first only (no rescan at launch)** — the conservative cut. Lost because rescan is the differentiator with the sharpest timing: "your AI wrote slop tests — cover100 cleans them" speaks to the AI-native audience's most current pain, and the feature is already built.
- **Go-only polished MVP, Python in v2** — lowest risk, fastest to ship. Lost because a single language leaves the language-profile architecture unproven; a second language at launch is the credibility claim that this is a platform, not a Go script. Python was chosen as the proof because its mock-based testing makes the profile *simpler* than Go's, demonstrating the contract flexes in both directions.
- **CI-first product (GitHub Action opening test PRs)** — biggest 10x potential. Lost for MVP because headless operation removes the human-in-the-loop levers (attempts question, cost confirmation, plan approval) that keep runs trustworthy and affordable; deferred to roadmap.
- **Keep it personal tooling (never productize)** — zero packaging cost. Lost because the harness already outperforms ad-hoc agent test generation in verifiable ways, the extraction cost is modest, and adoption/brand value accrues to the SpecScore ecosystem.

## MVP Scope

One sitting, stranger-proof: a developer installs the plugin, runs /cover100 on a Go or TypeScript subtree, answers two questions (attempts, cost confirm), and ends with green verified tests, 100%-or-documented coverage, and a shareable REPORT.md. Concretely: extract the orchestrator + Go profile from the workbench workflow, add the TypeScript profile (and Python if timing allows), package as github.com/specscore/cover100 marketplace plugin with preflight (codegraph, language toolchain), and publish the sneat-go case study.

## Not Doing (and Why)

- Python profile in the launch gate — moved to stretch/fast-follow (Go+TS is the launch pair; Go+TS+Python if timing allows)
- Runners beyond vitest/jest in the TS profile (node:test, mocha, bun, deno) — scoped support matrix keeps the claim honest
- CI / GitHub Action headless mode — cost-control and non-interactive UX are a v2-sized problem
- SpecScore REQ cross-linking (tests cite requirements) — integration, not core; target users are not required to use SpecScore
- Community language-profile SDK / marketplace — stabilize the profile contract on two languages first
- Hosted dashboards, paid tiers, telemetry — free OSS; adoption and brand are the goal
- Soft codegraph fallback (grep-only mode) — hard requirement keeps agent quality and cost predictable

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The orchestrator generalizes to a second language with profile-only changes (no rewrite of phases, gates, or no-work-lost mechanics) | Build the TypeScript profile and run it end-to-end on a real mid-size vitest/jest repo before announcing |
| Must-be-true | Users will install an external CLI (codegraph) as a hard prerequisite for a plugin | Measure drop-off between plugin install and first successful run with early adopters; make preflight + install docs one command |
| Must-be-true | The "100% or documented gap" claim survives arbitrary repos (cgo, codegen, build-tag mazes) without embarrassment | Run on 3+ unfamiliar OSS repos before launch; quarantine list must degrade gracefully, never lie |
| Should-be-true | A subtree first-run costs little enough (single-digit dollars, under an hour) that strangers finish their first sitting | Cost-projection telemetry on demo repos; tune default budgets for the first-touch path |
| Should-be-true | The adversarial-verifier gates keep generated-test quality visibly above ad-hoc agent output | Side-by-side comparison on the same package: cover100 vs plain "write tests for this" prompt; publish it |
| Might-be-true | Rescan resonates harder than drive-to-100 in marketing | A/B the two headlines in launch posts; watch which demo gets shared |
| Might-be-true | Consultants adopt REPORT.md as a client deliverable | Watch for inbound "can I white-label this" signals; no build needed up front |


## SpecScore Integration

- **New Features this would create:** language-profile contract; orchestrator core (phases, gates, worker pool, resume/salvage); /cover100 skill UX (preflight, cost projection, attempts); rescan mode; run report (REPORT.md); plugin packaging & distribution
- **Existing Features affected:** none (new repo; SpecScore itself unchanged)
- **Dependencies:** codegraph CLI (hard requirement); Claude Code Workflow tool semantics (agent caps, resume cache); per-language toolchains (go/golangci-lint, python/pytest/coverage)

## Open Questions

- Distribution channel: which marketplace(s) — specscore's own marketplace repo, community lists, or both — and what does install friction look like end-to-end?
- ~~Repo home~~ DECIDED: canonical home is github.com/specscore/cover100, permanently (reuses the org's marketplace/homebrew/scoop distribution infra and family credibility). The github.com/cover100 account is taken by an unrelated dormant personal user, so no defensive org; the repo-level token is unique (0 GitHub repos named cover100). If a dedicated org is ever needed: cover100-dev / getcover100. Grab the domain (cover100.dev) when convenient.
- codegraph distribution: brew tap / scoop bucket / go install — what is the one-command install on each OS, and who maintains it?
- Does the plugin bundle the workflow script per release or fetch a pinned version, and how do users pin/upgrade?
- Naming hygiene: confirm no trademark/package collisions for "cover100" across GitHub, PyPI-adjacent tooling, and plugin marketplaces.
- Case-study licensing: which sneat-go run artifacts (reports, diffs, metrics) can be published verbatim?
- Python profile specifics: minimum supported pytest/coverage versions; how branch coverage (vs Go's statement coverage) is presented without muddying the "100%" claim.
