---
format: https://specscore.md/idea-specification
status: Draft
---

# Idea: .NET (C#) language profile — the first corporate big fish

**Status:** Draft
**Date:** 2026-06-10
**Owner:** trakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we make cover100 work on solution/project-structured .NET codebases whose modularity model differs fundamentally from Go packages?

## Context

Corporate .NET shops have huge repos, real test-coverage mandates, and token budgets to spend — the audience cover100 eventually wants most. The structural challenge: a repo is a solution (.sln) of projects (.csproj), each project can be 100k+ LOC, and tests live in SEPARATE test projects (Foo.Tests.csproj) rather than beside the code. The orchestrator's package concept and the tests-only policy both need a mapping, not a rewrite.

## Recommended Direction

Hierarchical scoping that maps the existing contract onto .NET structure: solution = repo scope, project = subtree scope (the default demo motion: /cover100 --project src/Foo), namespace-directory within a project = engineer unit (Go-package-equivalent granularity), class = region. Coverage via coverlet (dotnet test --collect 'XPlat Code Coverage') producing cobertura XML mapped back to files and namespace-dirs. Policy stays tests-only with ONE structural allowance: creating/extending TEST projects (including .csproj edits and .sln registration for a new Foo.Tests) counts as test infrastructure, not production change; the verifier whitelists exactly that shape. Build gate = dotnet build + analyzers; idiomatic DI (constructor injection) plus .NET 8 TimeProvider make mock-based testing viable (Moq/NSubstitute when already referenced); statics/sealed without seams become documented gaps in v1. Validation spike first: run the profile on one mid-size OSS .NET repo, one project, end-to-end.

## Alternatives Considered

- **Project = unit (no namespace slicing)** — simplest mapping. Lost: corporate projects routinely exceed 100k LOC; a unit must fit one engineer context window, so sub-project granularity is non-negotiable.
- **Seam policy from day one (internal virtual / InternalsVisibleTo)** — closer to the Go profile. Lost for v1: idiomatic modern C# is DI-heavy enough that tests-only covers most branches, and corporate reviewers will accept "no production changes" far more readily than any seam. Revisit with gap-register data from the spike.
- **MSBuild-native orchestration (custom targets)** — deeper integration. Lost: couples us to MSBuild internals; the profile contract only needs CLI calls and parseable output (coverlet cobertura XML).

## MVP Scope

A validation spike, not a launch: profiles/dotnet passing conformance + one real mid-size OSS .NET (8+) repo taken through a single-project run end-to-end (collect -> units -> verified merges -> REPORT.md), with zero orchestrator changes. Output: a go/no-go report on the two named risks (codegraph C# support; wall-clock build times per unit worktree).

## Not Doing (and Why)

- .NET Framework (legacy) — modern .NET 8+ only; corporate legacy migration is not our fight
- Internal-virtual / InternalsVisibleTo seam policy in v1 — tests-only plus documented gaps first; seams are a v2 decision with data
- CI-mode integration (Azure DevOps/Jenkins) — depends on the deferred CI feature; corporate adoption path, not profile v1
- IDE tooling (VS/Rider plugins) — the plugin is Claude Code; IDEs are out of scope entirely

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | codegraph (or a replacement Roslyn/LSP indexer) can index C# well enough for researcher/engineer symbol lookups | Index a mid-size OSS .NET repo and benchmark query quality before any profile work |
| Must-be-true | Per-unit wall-clock (dotnet build + scoped test run in a worktree) stays under ~5 min on a mid-size project | Measure in the validation spike; if violated, explore shared build output / project-scoped restore |
| Must-be-true | The tests-only policy + test-project allowance covers ≥80% of branches without seams in idiomatic DI codebases | Gap-register analysis from the spike run |
| Should-be-true | coverlet cobertura output maps cleanly back to namespace-directories for unit packing | Fixture repo in conformance suite |
| Should-be-true | NuGet package cache is safely shareable across parallel unit worktrees | Parallel-build test during spike |
| Might-be-true | Corporate teams will run a Claude Code plugin locally before any CI integration exists | Early-adopter interviews; CI mode may need to lead for this audience |


## SpecScore Integration

- **New Features this would create:** C#-language-profile Feature (after the validation spike answers go/no-go)
- **Existing Features affected:** language-profile-contract (hierarchical scoping: subtree-scope concept must support project-within-solution; the test-location mapping must be declarable in profile.yaml)
- **Dependencies:** codegraph C# support (HARD blocker — validate first); orchestrator-core unchanged by design

## Open Questions

- codegraph C# support — REFRAMED after inspecting the dependency: codegraph (`@colbymchenry/codegraph`, MIT, npm, third-party single-maintainer) is tree-sitter-based and bundles a C# grammar, so it likely already indexes C# at the same syntactic fidelity our Go runs used. The spike question is therefore PRECISION (name-matching quality in overload/DI-heavy code), validated empirically by indexing a real .NET repo and benchmarking callers/callees usefulness. If precision is insufficient: (a) upstream feature request/PR (MIT permits contribution), (b) fork/vendor (MIT permits), or (c) a cover100-owned thin query CLI over SCIP indexes (`scip-dotnet` is official). LSP (OmniSharp/Roslyn) remains the bridge of last resort: name-based CLI wrapper required, ONE server on the integration worktree only (parallel per-unit servers are operationally untenable — observed with gopls on 14 worktrees). Dependency context: codegraph is a ~46k-star actively-developed MIT project — pin versions per release (policy in plugin-packaging); precision improvements are best pursued as upstream contributions.
- Test-project allowance shape: exactly which .csproj/.sln edits are whitelisted, and how does the verifier distinguish them from production project edits?
- Coverage metric naming: coverlet reports line+branch — present as "line coverage" like Python, or unify reporting language across non-Go profiles?
- Solution filters (.slnf) for partial builds on huge solutions — adopt for unit scoping?
