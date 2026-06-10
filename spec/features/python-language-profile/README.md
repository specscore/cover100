---
format: https://specscore.md/feature-specification
status: Approved
---
# Feature: Python language profile

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/cover100/spec/features/python-language-profile?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/cover100/spec/features/python-language-profile?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/cover100/spec/features/python-language-profile?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/cover100/spec/features/python-language-profile?op=request-change) |
**Status:** Approved
**Source Ideas:** cover100
**Grade:** A

## Summary

The Python implementation of the language-profile contract — pytest + coverage.py, with a tests-only production-change policy (monkeypatch/mock make seams unnecessary). It is the proof that the contract flexes: a *simpler* policy than Go's, expressed purely as profile data.

## Problem

Python repos are the second-largest audience and the architecture's credibility test: if Python requires orchestrator changes, the "languages are data" claim fails. It must ship as nothing but a profile directory.

## Behavior

### Manifest

#### REQ: py-manifest

`profiles/python/profile.yaml` MUST declare: detection markers `pyproject.toml` / `setup.cfg` / `setup.py` (any); preflight `python3 --version`, `pytest --version`, `coverage --version` (or `pytest --cov` plugin presence), `ruff --version` (lint+fmt; degrades with warning when absent), `codegraph` presence; coverage metric `line` — reported as "line coverage" everywhere, never conflated with Go's statement metric; test-file pattern `test_*.py` / `*_test.py` plus `tests/` directories; `productionChangePolicy: tests-only`; kind hints favoring parent-package grouping.

### Policy

#### REQ: py-tests-only

Under `tests-only`, ANY change to a non-test `.py` file (or any non-test file at all) MUST fail the unit's policy gate, listed file-and-line. Branches reachable only through patching MUST be covered via `monkeypatch`/`unittest.mock.patch` at test level — never via production edits. The gap register remains available for genuinely unreachable code (e.g. `if TYPE_CHECKING`, platform-specific branches).

### Fakes guide

#### REQ: py-fakes-guide

`prompts/fakes-guide.md` MUST direct engineers to, in order: pytest builtins (`monkeypatch`, `tmp_path`, `capsys`, `caplog`), `unittest.mock` (patch/MagicMock/sentinel), then well-known libraries ONLY if already declared in the project (`responses`/`requests-mock`/`respx` for HTTP, `freezegun`/`time-machine` for time, `fakeredis`, `moto`); adding a new test-only dependency is allowed when it clearly beats a hand-rolled fake and MUST surface in dependenciesAdded. Hand-rolled fakes duplicating these MUST be reported as [reuse] observations.

### Collection and checks

#### REQ: py-collect

`scripts/collect.sh` MUST run pytest under coverage with the repo's own configuration when present (`pyproject.toml`/`.coveragerc` `[tool.coverage]` settings, including `omit`/`exclude_lines` pragmas), aggregate per-module line totals, count modules with no tests at full weight, classify collection-erroring modules into `failedPackages`, and emit the contract JSON. `# pragma: no cover` lines are excluded from the denominator and MUST be reported in totals as pragma-excluded.

#### REQ: py-test-idioms

`prompts/test-idioms.md` MUST prefer `pytest.mark.parametrize` for multi-case coverage of one function, fixtures over setup duplication, plain asserts over assertEquals-style, and require consolidation of copy-paste tests the engineer touches.

#### REQ: py-end-to-end-validation

Before the profile is announced as supported, a full run on a real mid-size external pytest repo MUST complete end-to-end (all phases, ≥1 partial merge exercised, REPORT.md produced) using only profile-directory changes — zero orchestrator edits. This validates the Idea's first must-be-true assumption.

## Acceptance Criteria

### AC: conformance-pass (verifies REQ:py-manifest)

**Given** the bundled Python profile
**When** `tools/profile-conformance profiles/python` runs
**Then** it exits 0

### AC: production-edit-rejected (verifies REQ:py-tests-only)

**Given** an engineer diff touching `src/app/util.py` (non-test)
**When** the verifier applies the tests-only policy
**Then** the unit fails the policy gate with that file listed

### AC: respects-coverage-config (verifies REQ:py-collect)

**Given** a fixture repo whose pyproject omits `migrations/*` from coverage
**When** collect.sh runs
**Then** migration files do not appear in the uncovered totals

### AC: zero-orchestrator-diff (verifies REQ:py-end-to-end-validation)

**Given** the validation run on the external pytest repo
**When** the run completes
**Then** `git diff` over the orchestrator workflow file(s) between run start and end is empty

## Rehearse Integration

collect/conformance ACs get `_tests/` scenarios at first code drop; the end-to-end validation AC is a manual release-gate checklist item, not an automated stub.

## Open Questions

- Branch coverage: enable `--cov-branch` by default (stricter, slower) or offer as a profile option with line-only default?
- Monorepos with multiple `pyproject.toml`: scope to the nearest root or ask?
- Async tests: require `pytest-asyncio` strict mode guidance in test-idioms?

---
*This document follows the https://specscore.md/feature-specification*
