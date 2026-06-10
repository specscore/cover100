---
format: https://specscore.md/feature-specification
status: Approved
---
# Feature: Rescan mode

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/cover100/spec/features/rescan-mode?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/cover100/spec/features/rescan-mode?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/cover100/spec/features/rescan-mode?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/cover100/spec/features/rescan-mode?op=request-change) |
**Status:** Approved
**Source Ideas:** cover100
**Grade:** A

## Summary

An opt-in mode (`rescan`) that extends a run to packages already at 100% coverage: researchers audit the EXISTING tests for quality problems, and packages with findings get an engineer pass whose mission is to improve tests without losing a single point of coverage. The "clean up AI slop tests" feature.

## Problem

Coverage tools stop at 100%, but a 100%-covered suite can still be slop: hand-rolled fakes duplicating stdlib (a real `fakeHttpRecorder` reimplementing `httptest.NewRecorder`), copy-paste test functions that should be one table/parametrized test, dead helpers, redundant tests. This debt is invisible to every coverage metric.

## Behavior

### Audit

#### REQ: rescan-inclusion

With `rescan` enabled, Collect MUST keep fully-covered packages as candidates instead of quarantining them, and Research MUST audit their existing tests against the profile's fakes guide and test idioms, reporting findings as regions with whyType="test-quality" (location + concrete refactor approach). A covered package with no findings returns an empty region list.

#### REQ: rescan-triage

Only covered packages WITH test-quality findings proceed to engineering; audited-clean packages MUST be skipped at zero engineer cost, with the triage counts logged. Covered packages MUST receive a nominal pack weight so unit packing stays balanced.

### Refactor

#### REQ: rescan-engineer-rules

For rescan packages the engineer's mission MUST be: apply the test-quality annotations — replace custom fakes with standard tools, consolidate copy-paste tests, delete dead helpers and redundant tests — under hard rules: coverage MUST NOT drop (measured before and after), tests stay green, assertions stay equally meaningful; a refactor that cannot preserve coverage and meaning MUST be reverted.

#### REQ: rescan-verifier-rules

For rescan packages the verifier MUST require: coverage remained at its pre-run level, and the diff NET-SIMPLIFIED the tests (same verification, fewer or clearer lines). A "refactor" that grew test code without adding verification or weakened assertions MUST fail the meaningful gate with a concrete issue.

#### REQ: rescan-default-off

Default-mode runs MUST be byte-identical in agent prompts to a build without rescan support (the flag adds conditional content only), so rescan's existence never perturbs default behavior or resume caches.

## Acceptance Criteria

### AC: clean-package-costs-nothing (verifies REQ:rescan-triage)

**Given** a rescan run where research finds no issues in a covered package
**When** units are packed
**Then** that package appears in no unit and the skip is logged

### AC: coverage-drop-rejected (verifies REQ:rescan-verifier-rules)

**Given** a rescan refactor that lowered a package from 100% to 97%
**When** the verifier evaluates the unit
**Then** the unit fails with the coverage regression named

### AC: duplicate-fake-replaced (verifies REQ:rescan-engineer-rules)

**Given** a covered package with a hand-rolled HTTP response recorder flagged by research
**When** the rescan engineer completes and verification passes
**Then** the custom fake is gone, the standard tool is used, tests pass, and coverage is unchanged

### AC: default-mode-unperturbed (verifies REQ:rescan-default-off)

**Given** the same repo and stamp
**When** prompts are generated with rescan disabled, before and after rescan support exists in the codebase
**Then** the prompts are byte-identical

## Rehearse Integration

ACs need the orchestrator + a fixture repo with seeded slop tests; stubs follow the orchestrator-core implementation.

## Open Questions

- Should rescan also flag slow tests (>N s) for optimization, or stay scoped to structure/duplication?

---
*This document follows the https://specscore.md/feature-specification*
