---
format: https://specscore.md/idea-specification
status: Draft
---

# Idea: Java language profile — the second corporate big fish

**Status:** Draft
**Date:** 2026-06-10
**Owner:** trakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we bring cover100 to Maven/Gradle Java codebases — the largest corporate test-coverage market — while their build systems and module structure differ from everything we ship at launch?

## Context

Java corporate repos are huge, often have contractual coverage thresholds (JaCoCo gates already in CI), and teams accustomed to paying for quality tooling. Structurally Java is FRIENDLIER to cover100 than C#: Maven/Gradle modules map to Go-module-like scopes, and java packages mirror 1:1 between src/main/java and src/test/java — the engineer-unit mapping is nearly native. The pain is build mechanics: slow JVM builds, per-module test invocation, and worktree-parallel builds competing for daemons/caches.

## Recommended Direction

Map module = subtree scope (/cover100 --module core), java package = engineer unit (mirrored test package src/test/java/<same path> — tests-only policy needs no structural allowance at all), class = region. Coverage via JaCoCo XML per module (line metric, branch available), invoked through the repo's own build tool (mvn -pl <module> test / gradle :module:test) so repo build config is honored. Mocking: Mockito-first (ubiquitous), constructor-injection idiom, java.time.Clock for time; mockito-inline for statics when already present; PowerMock usage is rescan-mode fodder, not something we adopt. Lint/fmt via the repo's own spotless/checkstyle when configured. Same validation-spike-first approach as .NET, sharing its hierarchical-scoping design.

## Alternatives Considered

- **Gradle-first** — larger share of modern repos. Lost for the spike: Maven's CLI is more deterministic (no daemon/config-cache interplay with parallel worktrees); Gradle ships in v1 or v1.1 based on spike data, not assumption.
- **Method-level units (JaCoCo can report per-method)** — finer than packages. Lost: java packages already map 1:1 to mirrored test packages and fit context windows; finer granularity buys nothing but scheduling overhead.
- **Adopt PowerMock for static-heavy legacy code** — would raise reachable coverage on old codebases. Lost: PowerMock is deprecated-culture tooling that fights modern JVMs; static-heavy branches become documented gaps and rescan-mode flags the debt instead.

## MVP Scope

A validation spike: profiles/java passing conformance + one real mid-size OSS Java (17+, Maven first) repo taken through a single-module run end-to-end with zero orchestrator changes. Output: go/no-go on codegraph Java support and per-unit build wall-clock; a decision whether Gradle support ships in profile v1 or v1.1.

## Not Doing (and Why)

- Java 8/11 support — target 17+; legacy JVMs are a services engagement, not a product target
- Gradle in the first spike — Maven first (more deterministic CLI); Gradle decided by spike data
- PowerMock/static-heavy legacy test stacks — flagged by rescan as debt, never emulated
- Android — entirely different toolchain and test pyramid

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | codegraph (or replacement indexer) handles Java at researcher/engineer-useful quality | Index a mid-size OSS Java repo; benchmark before profile work |
| Must-be-true | Per-unit wall-clock (mvn -pl module test) is acceptable despite JVM startup; parallel worktrees don't fight over daemons/caches | Measure in the validation spike with parallel units |
| Must-be-true | Mirrored-package mapping (src/main ↔ src/test) holds across real corporate layouts (generated sources, annotation processors) | Spike on a repo using Lombok/annotation processing |
| Should-be-true | JaCoCo line metric communicates honestly alongside Go statement coverage in REPORT.md | Report-wording review (shared problem with Python profile) |
| Might-be-true | Maven-first doesn't alienate the Gradle majority among modern shops | Count Gradle requests after the spike write-up |


## SpecScore Integration

- **New Features this would create:** Java-language-profile Feature (after the validation spike answers go/no-go)
- **Existing Features affected:** language-profile-contract (hierarchical scoping: subtree-scope concept must support module-within-build; the test-location mapping must be declarable in profile.yaml)
- **Dependencies:** codegraph Java support (HARD blocker — validate first); orchestrator-core unchanged by design

## Open Questions

- codegraph Java support — same reframing as .NET: codegraph is tree-sitter-based with a Java grammar bundled, so support likely exists today; the spike validates PRECISION empirically. Fallback ladder if insufficient: upstream PR (MIT) → fork/vendor → cover100-owned SCIP query CLI (`scip-java` is official). jdtls-as-bridge carries the same single-server/integration-worktree-only constraint.
- Build-tool invocation: honor repo wrappers (mvnw/gradlew) always? (Almost certainly yes — corporate repos pin versions there.)
- JaCoCo exclusions (generated code, Lombok) — respect existing build-config exclusions like the Python profile respects coverage config?
- Multi-module reactor builds: can collect run once at root and attribute per module, or must it iterate modules?
