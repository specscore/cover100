---
format: https://specscore.md/feature-specification
status: Approved
---
# Feature: TypeScript/JavaScript language profile

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/cover100/spec/features/typescriptjavascript-language-profile?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/cover100/spec/features/typescriptjavascript-language-profile?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/cover100/spec/features/typescriptjavascript-language-profile?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/cover100/spec/features/typescriptjavascript-language-profile?op=request-change) |
**Status:** Approved
**Source Ideas:** cover100
**Grade:** A

## Summary

The TypeScript/JavaScript implementation of the language-profile contract — one profile covering both TS and JS, with runner adapters (vitest, jest first) inside the profile's scripts. IN the launch MVP as the second language (the founder's own stack and the largest active-OSS audience); the scoped support matrix (REQ ts-v1-scope) keeps the launch claim honest while the runner adapters live inside the profile's scripts, not in the contract.

## Problem

JavaScript has no compiler: type errors, typos, and wrong-shape data all survive to runtime, so tests are the only safety net — coverage matters MORE here than in compiled languages, and TypeScript only partially mitigates it. At the same time the ecosystem is the most fragmented (runners: vitest/jest/node:test/mocha/bun; module systems: ESM/CJS; transpilers; monorepos), which is why an unscoped "supports JS" claim would be a lie. This profile makes the scoped, honest version.

## Behavior

### Scope (v1 of this profile)

#### REQ: ts-v1-scope

The profile's v1 supported matrix MUST be stated and enforced: runners vitest and jest (detected; others stop with "runner not yet supported" naming the runner), Node ≥ 20, single-package repos and pnpm/yarn/npm workspaces, TS and JS sources. Out-of-matrix repos MUST fail preflight with the matrix in the message — never degrade into guessing.

### Manifest

#### REQ: ts-manifest

`profiles/typescript/profile.yaml` MUST declare: detection marker `package.json` (TS flavor via `tsconfig.json`); preflight `node --version` (≥20), package-manager detection (pnpm/yarn/npm via lockfile), runner presence (vitest or jest in devDependencies or the test script), `codegraph`; coverage metric `statement` (istanbul-format reporting); test-file patterns `*.test.*`, `*.spec.*`, `__tests__/`; `productionChangePolicy: tests-only`; kind hints favoring directory grouping plus name-suffix conventions (`*.controller.*`, `*.service.*`, `*.repository.*`, `use*` hooks).

### Collection

#### REQ: ts-collect

`scripts/collect.sh` MUST detect the runner and produce istanbul-format `coverage-summary.json` (vitest `--coverage` with json-summary reporter; jest `--coverage --coverageReporters=json-summary`), honoring the repo's own coverage configuration (exclude patterns, thresholds ignored), mapping coverage "packages" to workspace packages in monorepos and to top-level source directories in single-package repos, counting source files with no tests at full statement weight, and emitting the contract JSON. `.d.ts`, declared generated files, and config files MUST be excluded from the denominator.

#### REQ: ts-typecheck-gate

For TS repos, the unit build gate MUST include `tsc --noEmit` (or the repo's own typecheck script when present) in addition to the test run — type-clean is the compiled-language-equivalent of buildGreen.

### Testing approach

#### REQ: ts-fakes-guide

`prompts/fakes-guide.md` MUST direct engineers, in order: runner builtins first (`vi.mock`/`jest.mock` module mocks, `vi.fn`/`jest.fn`, fake timers `vi.useFakeTimers`/`jest.useFakeTimers`), then well-known libraries ONLY if already declared in the project (`msw` for HTTP, `supertest` for HTTP handlers, `memfs` for filesystem, `nock`/undici MockAgent for outbound requests, `testing-library` for components); adding a new test-only dependency is allowed when it clearly beats a hand-rolled fake and MUST surface in dependenciesAdded. Hand-rolled fakes duplicating these MUST be reported as [reuse] observations.

#### REQ: ts-tests-only-policy

Under `tests-only`, any change to a non-test source file MUST fail the policy gate. Branches reachable only through module interception MUST be covered via the runner's module-mock mechanism, never via production edits. Known ESM live-binding limitations (spying on same-module internal calls) are documented gap material, not an excuse for production edits.

#### REQ: ts-test-idioms

`prompts/test-idioms.md` MUST prefer `test.each`/`it.each` (or vitest equivalents) for multi-case coverage of one function, `describe` blocks mirroring source structure, explicit async handling (no unawaited promises in tests), and consolidation of copy-paste tests the engineer touches.

## Acceptance Criteria

### AC: out-of-matrix-stops (verifies REQ:ts-v1-scope)

**Given** a repo whose tests run under mocha
**When** `/cover100` preflight runs with the TS profile
**Then** the run stops naming mocha as unsupported and listing the v1 matrix

### AC: conformance-pass (verifies REQ:ts-manifest)

**Given** the bundled TypeScript profile
**When** `tools/profile-conformance profiles/typescript` runs
**Then** it exits 0

### AC: vitest-and-jest-collect (verifies REQ:ts-collect)

**Given** two fixture repos — one vitest, one jest — each with one tested and one untested module
**When** collect.sh runs on each
**Then** both emit contract JSON with the untested module's statements counted as uncovered

### AC: monorepo-packages (verifies REQ:ts-collect)

**Given** a pnpm workspace with packages `core` and `cli`
**When** collect.sh runs
**Then** coverage is reported per workspace package, not as one merged blob

### AC: type-error-fails-gate (verifies REQ:ts-typecheck-gate)

**Given** an engineer's test that passes at runtime but introduces a `tsc --noEmit` error
**When** the unit's build gate runs
**Then** the gate fails with the type error

### AC: production-edit-rejected (verifies REQ:ts-tests-only-policy)

**Given** an engineer diff touching `src/index.ts`
**When** the verifier applies the policy
**Then** the unit fails the policy gate with that file listed

## MVP-inclusion assessment (non-normative)

DECIDED 2026-06-10: IN the launch MVP (Go+TS launch pair; Python moved to stretch/fast-follow). Original assessment kept for the record:

What the contract gives for free: orchestrator, gates, no-work-lost, salvage/collect JSON shapes, tests-only policy (shared with Python), prompt-fragment mechanism. The cost center is exactly one artifact: a robust `collect.sh` across {vitest, jest} × {TS, JS} × {single-package, workspaces} × {ESM, CJS} — fixture-driven work, est. days for the happy matrix, weeks if node:test/mocha/bun are chased. Verdict: keep OUT of launch MVP (two languages already prove the contract), but build against the conformance suite as the first fast-follow (v1.1), scoped by REQ:ts-v1-scope so the supported matrix is honest from day one. Including it in MVP would delay launch by roughly the collect.sh fixture matrix and double the OSS-campaign target pool — a trade the launch plan can revisit if Python lands faster than planned.

## Rehearse Integration

Fixture ACs (vitest-and-jest-collect, monorepo-packages, conformance) get `_tests/` scenarios when profile work starts; policy/typecheck ACs follow orchestrator-core.

## Open Questions

- node:test + c8: cheap enough to include in v1 matrix, or strictly v1.2?
- Branch coverage (istanbul reports it natively): expose alongside statement coverage in REPORT.md for this profile?
- Coverage provider for vitest: default v8 provider's statement mapping vs istanbul provider — pick one for determinism?
- bun/deno: explicitly out until demand is proven?

---
*This document follows the https://specscore.md/feature-specification*
