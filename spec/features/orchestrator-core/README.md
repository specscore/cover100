---
format: https://specscore.md/feature-specification
status: Approved
---
# Feature: Orchestrator core

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/cover100/spec/features/orchestrator-core?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/cover100/spec/features/orchestrator-core?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/cover100/spec/features/orchestrator-core?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/cover100/spec/features/orchestrator-core?op=request-change) |
**Status:** Approved
**Source Ideas:** cover100
**Grade:** A

## Summary

The language-agnostic workflow that takes a repo from its current coverage to 100%-or-documented: deterministic phases, an engineer↔verifier worker pool with hard quality gates, serialized integration, and a no-work-lost state model. Battle-tested as go-coverage-100; this Feature is its productized contract.

## Problem

Ad-hoc "write tests for this" agent runs produce unverified tests, lose work on interruption, and don't scale past one context window. The orchestrator turns coverage into a resumable, verifiable production line whose every merge is gated.

## Behavior

### Phases

#### REQ: phase-sequence

A run MUST execute these phases in order: Salvage (recover aborted-run leftovers; no-op on fresh runs), Collect (deterministic baseline via the profile's collect script; creates the integration branch `coverage/100-<stamp>` and its worktree), Classify (kind classification: heuristic draft + a single AI review returning only changes), Research (annotate uncovered regions: why-type taxonomy + concrete approaches), Helpers (build shared test helpers only when ≥3 distinct packages would use one), Plan units (manifest adoption + affinity bin-packing), Cover (worker pool of engineer↔verify cycles with integration), Finalize (whole-module build/lint/test, report-only race check, observation aggregation), optional Harvest (reusable patterns), Overview (machine-readable summary). Phase titles MUST be static and declared in workflow metadata so progress UIs order them correctly.

#### REQ: adaptive-budgets

Research-batch and engineer-unit statement budgets MUST scale with repo size (defaults: uncovered/32 batches and uncovered/64 units, with floors), and the run MUST log a projected agent count up front and warn when it approaches the platform's lifetime agent cap. Explicit budget arguments override the defaults.

### Cover loop

#### REQ: worker-pool

The Cover phase MUST use a bounded worker pool (not a FIFO pipeline): each worker owns one unit's engineer→verify cycle with at most one agent call outstanding, so verification starts the moment its engineer returns. Pool size MUST default to the platform's concurrent-agent cap minus one — reserving one slot for the serialized integrator — and MUST be capped by the unit count.

#### REQ: hard-gates

A unit MUST merge into the integration branch only when an independent SAFETY verifier (a separate agent from the engineer) confirms ALL hard gates: buildGreen (compiles, all tests pass), meaningful (tests assert real behavior — no coverage padding), production-change-policy compliance (per the language profile), and no new lint/fmt findings. The verifier's JUDGMENT effort MUST focus on exactly two checks — forbidden production changes and shortcut/padding tests; everything else it verifies mechanically (run the command, record the result). The completeness gate (targetMet: 100% or every remaining gap has a documented entry — an existence check, not a quality re-litigation) is soft — failing it downgrades to a partial merge, never blocks safe work. A documented gap that looks obviously coverable MUST be flagged in the verifier's issues (report-only), never re-litigated as a gate.

#### REQ: bounce-cycle

The engineer↔verify attempt count per unit MUST default to 1 (no bounce): per-unit retries hit diminishing returns, and the intended improvement loop is re-running the whole workflow (see REQ: next-run-instructions). A configured attempt count of 2–3 MAY re-enable bounces, in which case the engineer MUST be re-dispatched with the verifier's findings verbatim. On the final attempt (every attempt when the count is 1) the verifier MAY salvage documentation (write missing gap entries marked as needing investigation) rather than discard otherwise-safe work.

#### REQ: prompt-efficiency

All worker prompts (research, helpers, engineer, verify, finalize) MUST carry the shell-batching policy: one decision per roundtrip — commands feeding the same decision are batched into a single call (chained/piped, output trimmed to what the agent will act on), split only when the second command depends on reasoning about the first's output. The engineer's main loop MUST prescribe a concrete batched check command (test result AND remaining-uncovered detail in one call), supplied per language by the profile.

#### REQ: serialized-integration

Merges into the integration branch MUST be serialized (one at a time) and fire-and-forget from the worker's perspective: the worker moves to its next unit immediately while the merge queues. Each merge MUST re-sync the code-intelligence index so later units see earlier units' helpers and seams. Per-package salvage MUST apply when a unit partially fails: individually-safe packages merge while failing packages' changes are dropped and recorded.

### No work lost

#### REQ: no-work-lost

The run MUST be abortable at any point without losing committed work: engineers commit each package as soon as it is green (never batching to the end); units map to content-hash-named branches recorded in a per-stamp manifest so re-runs assign the same packages to the same existing branches (rebasing them onto the integration branch on adoption); the Salvage phase recovers committed-but-unintegrated branches build+test-gated; resuming with the same stamp re-collects on the existing integration branch and skips completed packages.

#### REQ: free-byproducts

The run MUST capture, without acting on them: code-review observations (suspected bugs, dead code, design smells, security flags, doc drift, flaky tests, fuzz candidates, reuse opportunities) appended by all agent roles to an append-only observations file and aggregated into CODE-REVIEW-NOTES.md as explicitly-unverified leads; an append-only lessons file written by engineers/verifiers and read by later units; and per-package TEST-COVERAGE.md gap registers.

### Cross-run improvement loop

#### REQ: next-run-instructions

The run report MUST contain a "Next-run instructions" section aggregating every unresolved problem across units — verifier issues (including coverable-gap flags), policy violations, dropped/abandoned packages with reasons, partial merges — deduplicated, grouped by package, and phrased as imperative instructions for the next run's engineer (not as history). Researchers and engineers MUST consult this section from the previous run's report when their packages appear in it.

#### REQ: cross-run-escalation

The collector MUST deterministically mark packages a previous run already worked (presence of the package's gap register, e.g. TEST-COVERAGE.md). By default, units containing such a package MUST use the configured large engineer model — the first run is a cheap sweep; the residue that survives it is the hard part. This escalation MUST be disableable and the plan log MUST state how many units it affects.

#### REQ: keyed-knowledge-base

The run MUST maintain a keyed knowledge base (one file per key, O(1) lookup), starting with the mock namespace: fully-qualified dependency type → its faking solution (existing mock to import, shared helper, or recipe). It has two layers: a HOT per-run copy outside any git tree — the live cross-agent channel, which MUST NOT live in the repo because unit worktrees are isolated and an in-repo write is invisible across units until merge — and a PERSISTENT registry at `.cover100/mocks/` committed on the integration branch. Collect MUST seed the hot layer from the persistent one; Finalize MUST copy new hot entries back without overwriting existing files (committed and human-edited entries stay authoritative). Researchers pre-seed, helper builders register what they build, and engineers MUST look up before writing any new mock and register what they decide. Entries are first-writer-wins (never overwritten) and advisory (an agent may deviate, noting why). When `.cover100/` first materializes, the run MUST install `.cover100/README.md` by copying a shipped asset file byte-exact (never overwriting an existing one, never model-authored) — content: registry explainer, cover100 repository link, GitHub-star invitation.

## Acceptance Criteria

### AC: verify-starts-immediately (verifies REQ:worker-pool)

**Given** a run with more units than workers and one unit's engineer just returned
**When** the scheduler assigns the next agent slot
**Then** that unit's verifier starts before any queued engineer for a later unit

### AC: unverified-unit-never-merges (verifies REQ:hard-gates)

**Given** a unit whose verifier reports meaningful=false
**When** the integration decision runs
**Then** the unit does not merge (individually-safe packages may still salvage), and the verifier's issues land in the run report's Next-run instructions

### AC: coverable-gap-flagged-not-bounced (verifies REQ:hard-gates)

**Given** a unit whose documented gap is obviously coverable by a standard pattern
**When** the verifier evaluates the completeness gate
**Then** targetMet still passes on entry existence, and a coverable-gap flag naming the pattern appears in the verifier's issues

### AC: partial-merge-on-incomplete-docs (verifies REQ:hard-gates)

**Given** a unit that is buildGreen, policy-clean, and lint/fmt-clean but has an undocumented gap (targetMet=false) on its final attempt
**When** integration runs
**Then** the unit merges as partial and the gap is recorded rather than the work being abandoned

### AC: bounce-carries-findings (verifies REQ:bounce-cycle)

**Given** a run configured with attempt count 2 and a unit rejected on attempt 1 with two named issues
**When** the engineer's attempt 2 prompt is generated
**Then** both issues appear verbatim in the prompt

### AC: default-is-single-cycle (verifies REQ:bounce-cycle)

**Given** a run launched without an attempt-count argument
**When** the Cover phase plans its units
**Then** each unit gets exactly one engineer→verify cycle and the verifier's documentation-salvage exception is active on it

### AC: abort-loses-nothing-committed (verifies REQ:no-work-lost)

**Given** a run aborted while 3 units have committed work (1 integrated, 2 on unit branches)
**When** the same stamp is re-run
**Then** the integrated unit's packages are skipped by re-collection, the 2 leftover branches are salvaged or re-adopted, and no committed test is rewritten from scratch

### AC: integration-syncs-index (verifies REQ:serialized-integration)

**Given** unit A merged a shared fake into the integration branch
**When** unit B's engineer later queries the code-intelligence index
**Then** the fake from unit A is findable

### AC: budgets-scale (verifies REQ:adaptive-budgets)

**Given** a repo with 38,000 uncovered statements and no explicit budget args
**When** budgets resolve
**Then** unit count projects to ≈64 (not hundreds) and the projected agent count is logged

### AC: static-phase-titles (verifies REQ:phase-sequence)

**Given** a running workflow displayed in the progress UI
**When** the Cover phase is active while Harvest/Overview are pending
**Then** Cover renders in its declared position (before Finalize), not appended after pending phases

### AC: observations-survive (verifies REQ:free-byproducts)

**Given** an engineer that appended a [bug] observation during a unit that was later abandoned
**When** Finalize aggregates observations
**Then** the observation appears in CODE-REVIEW-NOTES.md marked as unverified

### AC: next-run-instructions-consumed (verifies REQ:next-run-instructions)

**Given** a previous run whose report lists "pkg/foo — seam X collided with Y; name it Z" and a new run targeting pkg/foo
**When** the new run's engineer prompt for the unit containing pkg/foo is generated
**Then** the engineer is directed to the previous report's Next-run instructions for its packages

### AC: revisit-gets-large-model (verifies REQ:cross-run-escalation)

**Given** a package below 100% whose directory contains a TEST-COVERAGE.md from a previous run
**When** the Cover phase dispatches the unit containing it (escalation not disabled)
**Then** that unit's engineer uses the large engineer model and the plan log reports the revisit-unit count

### AC: second-agent-reuses-mock (verifies REQ:keyed-knowledge-base)

**Given** unit A registered a knowledge-base entry for interface I while unit B is still running
**When** unit B's engineer needs a fake for I and performs the prescribed lookup
**Then** the lookup returns unit A's entry and no second mock for I is invented

### AC: kb-first-writer-wins (verifies REQ:keyed-knowledge-base)

**Given** an existing knowledge-base entry for interface I
**When** another agent attempts to register a different solution for I
**Then** the original entry is unchanged

### AC: kb-persists-across-runs (verifies REQ:keyed-knowledge-base)

**Given** a completed run that registered a solution for interface I, merged to the repo's default branch
**When** a fresh run (new stamp, different machine) reaches its first engineer needing a fake for I
**Then** the prescribed lookup returns the entry seeded from `.cover100/mocks/` without re-research

### AC: human-edit-wins-over-rerun (verifies REQ:keyed-knowledge-base)

**Given** a maintainer who hand-corrected `.cover100/mocks/<I>.md` after a run
**When** a later run's Finalize persists its hot entries
**Then** the maintainer's version of that file is unchanged

### AC: batched-check-prescribed (verifies REQ:prompt-efficiency)

**Given** a generated engineer prompt for any unit
**When** its main-loop instruction is inspected
**Then** it contains a single batched command yielding both the test result and the remaining-uncovered detail

## Rehearse Integration

Stubs deferred to implementation: scheduler ACs (worker pool, integration ordering) need the orchestrator runtime; the no-work-lost AC maps to an integration test over a fixture repo planned for the implementation phase.

## Open Questions

- Platform coupling: which Workflow-runtime guarantees (agent cap formula, resume cache keyed on prompt+opts, journal) are assumed vs abstracted, and what is the documented minimum Claude Code version?
- Model routing defaults (haiku for git/manifest, sonnet for research/engineering/verification, large model for helper aggregation and revisit-escalated engineers per REQ: cross-run-escalation): contract or tunable defaults?

---
*This document follows the https://specscore.md/feature-specification*
