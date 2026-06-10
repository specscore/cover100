---
format: https://specscore.md/feature-specification
status: Approved
---
# Feature: Language profile contract

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/cover100/spec/features/language-profile-contract?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/cover100/spec/features/language-profile-contract?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/cover100/spec/features/language-profile-contract?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/cover100/spec/features/language-profile-contract?op=request-change) |
**Status:** Approved
**Source Ideas:** cover100
**Grade:** A

## Summary

Defines the interface between cover100's language-agnostic orchestrator and everything language-specific: a profile is a self-contained directory of one manifest, two deterministic scripts, and three prompt fragments. Profile authors (initially the cover100 maintainers for Go and Python, later community contributors) implement this contract; the orchestrator consumes it blindly.

## Problem

The proven go-coverage-100 workflow hard-wires Go specifics (go test commands, the `var x = expr` seam policy, gofmt/golangci-lint, a Go fakes guide) into the orchestrator script. Without a contract boundary, every new language means forking the orchestrator — and every orchestrator improvement must be replicated per fork (a single tuning session produced ~15 improvements; multiplying that per language is unsustainable). The contract makes languages data, not code.

## Behavior

### Profile structure

#### REQ: profile-layout

A language profile MUST be a directory `profiles/<lang>/` containing exactly these required artifacts: `profile.yaml` (manifest), `scripts/collect.sh` and `scripts/salvage.sh` (executable, deterministic), and `prompts/seam-policy.md`, `prompts/fakes-guide.md`, `prompts/test-idioms.md` (prompt fragments). The orchestrator MUST source all language-specific behavior from the selected profile directory and MUST NOT contain language-conditional logic in its core.

#### REQ: manifest-fields

`profile.yaml` MUST declare: `language` (identifier), `detect` (list of repo-root marker files, e.g. `go.mod`), `preflight` (list of commands that must succeed before a run, e.g. `go version`, `golangci-lint --version`), `commands` (templates for test, lint, fmt, and checkLoop — the single batched command yielding both the test result and the remaining-uncovered detail in one shell call, prescribed verbatim in the engineer's main loop — each scoped to a package/directory list), `coverage` (metric: `statement` | `line` | `branch`, and the profile's definition of 100%), `testFilePattern` (glob distinguishing test files from production files), `productionChangePolicy` (`seam` | `tests-only`), and `kindHints` (delimiters/conventions for affinity grouping). The orchestrator MUST fail fast with a named-field error when a required field is missing or malformed.

### Deterministic scripts

#### REQ: collect-contract

`scripts/collect.sh <worktree> <pathFilter>` MUST run the language's coverage tooling and emit, as the last line of stdout, a single JSON object: `{"totals": {"coveragePct", "uncoveredStatements"}, "packages": [{"importPath", "preCoveragePct", "uncoveredStatements", "prevAttempt"}], "failedPackages": [{"importPath", "reason"}]}`. Counts MUST include packages/modules with zero tests. `prevAttempt` MUST be true exactly when the package's gap register (e.g. TEST-COVERAGE.md) already exists in its directory — the deterministic marker that a previous run worked the package, consumed by the orchestrator's cross-run model escalation. The script MUST exit 0 on success and non-zero only when collection itself failed (individual package failures belong in `failedPackages`).

#### REQ: salvage-contract

`scripts/salvage.sh <repoPath> <stamp> [worktreeBase]` MUST recover committed-but-unintegrated work from an aborted run: commit dirty unit-worktree state as marked WIP commits, merge each leftover unit branch into the integration branch gated by the profile's build+test commands, abort red merges leaving their branches intact, and emit a final-line JSON summary `{"resumed", "wipCommitted", "merged", "failed", "skipped"}`. On a fresh run (no integration branch) it MUST no-op with `resumed: false` and exit 0.

### Prompt fragments

#### REQ: prompt-fragments

The orchestrator MUST inject the profile's three prompt fragments verbatim at fixed points: `seam-policy.md` into engineer and verifier prompts, `fakes-guide.md` into researcher (rescan mode), helper-builder, engineer, and verifier prompts, and `test-idioms.md` (table-test/parametrize preferences, naming, structure) into engineer prompts. Fragments MUST be self-contained prose that makes no reference to orchestrator internals, other profiles, or absolute paths.

#### REQ: production-change-policy

The verifier's production-change gate MUST be driven by the manifest's `productionChangePolicy`: under `seam`, the only permitted production change is the seam form defined in `seam-policy.md` (for Go: a package-level `var name = expr` plus call-site swaps), enforced within the unit's assigned packages; under `tests-only`, ANY production-file change (per `testFilePattern`) is a violation. Violations MUST fail the unit's `seamOnly` gate and be listed file-and-line.

### Detection and selection

#### REQ: language-detection

At run start the orchestrator MUST auto-select the profile whose `detect` markers match the repo root. An explicit `lang=<id>` argument MUST override detection. When zero or multiple profiles match and no explicit argument was given, the run MUST stop and ask the user rather than guess.

#### REQ: preflight-gate

Before any phase runs, the skill MUST execute the selected profile's `preflight` commands plus the global codegraph check, and on any failure MUST stop with the failing command and installation guidance — never degrade silently.

### Conformance

#### REQ: conformance-suite

The repository MUST ship a profile conformance check (`tools/profile-conformance`) that validates a profile directory without running a full coverage workflow: manifest schema validation, script presence/executability, collect/salvage JSON-shape verification against a fixture repo, and prompt-fragment presence. Both bundled profiles (Go, Python) MUST pass it in CI; a profile that passes is loadable by the orchestrator without orchestrator changes.

## Acceptance Criteria

### AC: orchestrator-is-language-blind (verifies REQ:profile-layout)

**Given** a fictional but conformance-passing profile directory `profiles/fake/`
**When** the orchestrator is pointed at a repo with `lang=fake`
**Then** the run reaches the Collect phase using `profiles/fake/scripts/collect.sh` with zero modifications to the orchestrator script

### AC: missing-manifest-field-fails-fast (verifies REQ:manifest-fields)

**Given** a profile whose `profile.yaml` omits `productionChangePolicy`
**When** a run starts with that profile
**Then** the run aborts before Collect with an error naming `productionChangePolicy` and the profile path

### AC: collect-json-shape (verifies REQ:collect-contract)

**Given** the Go profile and a fixture repo containing one tested and one test-less package
**When** `scripts/collect.sh <worktree> ./...` runs
**Then** the last stdout line parses as JSON with both packages listed and the test-less package's statements counted as uncovered

### AC: salvage-recovers-green-branch (verifies REQ:salvage-contract)

**Given** an integration branch and one unit branch with a committed, passing test file (an aborted run's leftover)
**When** `scripts/salvage.sh <repoPath> <stamp>` runs
**Then** the unit branch is merged into the integration branch, and the JSON summary lists it under `merged`

### AC: salvage-noop-on-fresh-run (verifies REQ:salvage-contract)

**Given** a repo with no integration branch for the stamp
**When** `scripts/salvage.sh <repoPath> <stamp>` runs
**Then** it exits 0 with `resumed: false` and no git state changes

### AC: seam-policy-gates-production-edit (verifies REQ:production-change-policy)

**Given** a Go-profile unit whose engineer changed a production `if` statement (not a seam)
**When** the verifier evaluates the unit
**Then** `seamOnly` is false and the change is listed file-and-line in `seamViolations`

### AC: tests-only-policy-gates-any-production-edit (verifies REQ:production-change-policy)

**Given** a Python-profile unit (`productionChangePolicy: tests-only`) whose engineer edited any non-test `.py` file
**When** the verifier evaluates the unit
**Then** `seamOnly` is false and the edit is listed as a violation

### AC: fragments-reach-prompts (verifies REQ:prompt-fragments)

**Given** a profile whose `fakes-guide.md` contains a unique marker string
**When** an engineer prompt is generated for any unit
**Then** the marker string appears verbatim in that prompt

### AC: detection-picks-go (verifies REQ:language-detection)

**Given** a repo root containing `go.mod` and no other profile markers
**When** a run starts without a `lang=` argument
**Then** the Go profile is selected and named in the run's opening log line

### AC: ambiguous-detection-asks (verifies REQ:language-detection)

**Given** a repo root containing both `go.mod` and `pyproject.toml`
**When** a run starts without a `lang=` argument
**Then** the run stops and asks the user to choose before any phase executes

### AC: preflight-stops-on-missing-tool (verifies REQ:preflight-gate)

**Given** the Python profile selected on a machine without `pytest` on PATH
**When** the skill preflight runs
**Then** the run stops before Salvage with the failing command and install guidance in the message

### AC: conformance-validates-bundled-profiles (verifies REQ:conformance-suite)

**Given** the repository CI pipeline
**When** `tools/profile-conformance profiles/go profiles/python` runs
**Then** it exits 0, and mutating any required manifest field or deleting any required script makes it exit non-zero naming the violation

## Rehearse Integration

Rehearse stubs deferred until the implementation repo has its first code drop: the script ACs (collect-json-shape, salvage-recovers-green-branch, salvage-noop-on-fresh-run, conformance-validates-bundled-profiles) are directly automatable against fixture repos and will get `_tests/` scenarios then; the orchestrator-behavior ACs (fragments-reach-prompts, detection, policy gates) require the orchestrator core Feature to be implemented first.

## Open Questions

- Python coverage semantics: coverage.py reports line (and optionally branch) coverage — does the manifest's `coverage.metric` surface "100% of lines" vs Go's "100% of statements" distinctly in reports, and do we enable `--cov-branch` by default?
- Script runtime: are profile scripts guaranteed bash, or should the manifest declare an interpreter (Windows support is realistically WSL-only either way)?
- Shared script library: collect/salvage share git plumbing across languages — extract `scripts/lib.sh` into the contract, or keep each profile fully self-contained (current lean: self-contained; duplication is small and isolation is worth it)?
- Profile schema versioning: does `profile.yaml` carry a `schemaVersion` from day one so the orchestrator can reject too-new/too-old profiles?
- Code-intel provider: codegraph (third-party, MIT, tree-sitter-based — bundles grammars for all five target languages) is the hard requirement and the only agent-facing interface (query/callers/callees verbs); the contract stays provider-blind. Per-language precision is validated empirically per profile. If a language's precision proves insufficient, the fallback ladder is: upstream contribution → fork/vendor (MIT) → a cover100-owned thin query CLI over SCIP indexes — never per-profile LSP integrations as primary. The single-maintainer dependency risk needs a pinning/vendoring policy (owned by plugin-packaging).

---
*This document follows the https://specscore.md/feature-specification*
