---
format: https://specscore.md/feature-specification
status: Approved
---
# Feature: Go language profile

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/cover100/spec/features/go-language-profile?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/cover100/spec/features/go-language-profile?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/cover100/spec/features/go-language-profile?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/cover100/spec/features/go-language-profile?op=request-change) |
**Status:** Approved
**Source Ideas:** cover100
**Grade:** A

## Summary

The Go implementation of the language-profile contract — extracted from the battle-tested go-coverage-100 workflow. Go is the hardest profile (no monkeypatching), which is why it defines the seam policy; shipping it first proves the contract at its most demanding.

## Problem

Go's lack of runtime patching means some branches are untestable without a production change. An unconstrained agent "fixes" this by rewriting production code; the Go profile constrains it to one auditable change shape and gives it the standard toolbox so it stops reinventing fakes.

## Behavior

### Manifest

#### REQ: go-manifest

`profiles/go/profile.yaml` MUST declare: detection marker `go.mod`; preflight `go version`, `golangci-lint --version` (lint degrades to `go vet` when absent — preflight warns, not stops), `codegraph` presence; coverage metric `statement` computed from `go test -coverprofile` with `-covermode=atomic`; test-file pattern `*_test.go`; `productionChangePolicy: seam`; kind hints for the `<prefix>4<name>` / `<prefix>2<name>` / single-underscore naming conventions; and the `checkLoop` batched command `go test <dirs> -coverprofile=<tmp> -covermode=atomic 2>&1 | tail -15 && go tool cover -func=<tmp> | grep -v '100.0%' | head -40` (test result + remaining uncovered functions in one shell roundtrip).

### Seam policy

#### REQ: go-seam-policy

`prompts/seam-policy.md` MUST permit exactly one production change shape: a new package-level `var name = expr` declaration (including `var name = func(...){...}`) plus call-site swaps replacing a direct call/identifier with the seam variable — within the unit's assigned packages only. Removing or reverting existing seams, altered control flow, changed logic, new error handling, signature changes, and "bug fixes" are violations the verifier MUST list file-and-line.

### Fakes guide

#### REQ: go-fakes-guide

`prompts/fakes-guide.md` MUST direct engineers to standard tooling before any hand-rolled fake, in preference order stdlib > already-in-go.mod > well-known: `net/http/httptest` (NewRecorder/NewServer — never hand-roll a response recorder), `http.RoundTripper` swap for outbound clients, `testing/fstest` + `t.TempDir()`, `testing/iotest`, `bytes.Buffer`/`strings.NewReader`, `t.Setenv`, `io.Discard`, `go-sqlmock` for SQL, `bufconn` for gRPC, a clock seam for time, and mocks SHIPPED by a dependency module (e.g. gomock-based `mocks/` packages) over regenerating them. Hand-rolled fakes duplicating these MUST be reported as [reuse] observations.

### Collection and checks

#### REQ: go-collect

`scripts/collect.sh` MUST compute per-package uncovered statements from the merged cover profile, count packages WITHOUT test files at their full statement weight, classify build/test-failing packages into `failedPackages`, and emit the contract JSON.

#### REQ: go-new-findings-only

Lint checks MUST be scoped to NEW findings only (`golangci-lint run --new-from-rev=<baseline SHA>`), so pre-existing debt never blocks a unit; gofmt MUST be clean on changed files. Engineers run both per-package before committing; the verifier re-runs them independently.

#### REQ: go-test-idioms

`prompts/test-idioms.md` MUST prefer table-driven tests with `t.Run` subtests for multi-branch coverage of one function, require consolidation of copy-paste near-duplicate tests the engineer touches, and forbid forcing a table where cases differ structurally.

## Acceptance Criteria

### AC: conformance-pass (verifies REQ:go-manifest)

**Given** the bundled Go profile
**When** `tools/profile-conformance profiles/go` runs
**Then** it exits 0

### AC: seam-violation-rejected (verifies REQ:go-seam-policy)

**Given** an engineer diff containing a modified `return` statement in a production file
**When** the verifier applies the Go seam policy
**Then** the unit fails the policy gate with that file and line listed

### AC: testless-package-counted (verifies REQ:go-collect)

**Given** a fixture module with a package containing 50 statements and no test file
**When** collect.sh runs
**Then** that package appears with uncoveredStatements=50

### AC: preexisting-lint-debt-ignored (verifies REQ:go-new-findings-only)

**Given** a package with 10 pre-existing lint findings and an engineer change introducing none
**When** the unit's lint gate runs against the baseline SHA
**Then** the gate passes

## Rehearse Integration

Script ACs (conformance-pass, testless-package-counted) get `_tests/` scenarios at first code drop; policy ACs depend on the orchestrator-core verifier and follow it.

## Open Questions

- Build-tagged code (`//go:build integration`) — count toward the denominator or auto-quarantine with reason?
- cgo packages: quarantine by default?

---
*This document follows the https://specscore.md/feature-specification*
