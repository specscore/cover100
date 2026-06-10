---
format: https://specscore.md/feature-specification
status: Approved
---
# Feature: cover100 skill UX

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/cover100/spec/features/cover100-skill-ux?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/cover100/spec/features/cover100-skill-ux?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/cover100/spec/features/cover100-skill-ux?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/cover100/spec/features/cover100-skill-ux?op=request-change) |
**Status:** Approved
**Source Ideas:** cover100
**Grade:** A

## Summary

The `/cover100` skill — the only surface most users touch. It owns preflight, scope and cost confirmation, the attempts question, launch, the completion report, and safe-abort guidance. Design goal: a stranger finishes their first run in one sitting.

## Problem

The orchestrator is powerful enough to spend real money and hours; without an opinionated front door users either under-scope (disappointing demo) or over-scope (surprise bill, abandoned run). The skill makes the cheap path the default path.

## Behavior

### Invocation

#### REQ: invocation-args

`/cover100 [pathFilter] [stamp=<prev>] [attempts=<1|2|3>] [lang=<id>] [rescan]` — pathFilter scopes to a subtree (default whole repo); `stamp=` resumes a prior run; `attempts=` pre-answers the cycles question; `lang=` overrides profile detection; `rescan` enables test-quality auditing of fully-covered packages.

#### REQ: preflight

Before launching, the skill MUST verify: inside a git repo, a profile detected (or `lang=` given), the profile's preflight commands pass, and the codegraph CLI is installed — stopping with specific install guidance on any failure. Preflight failures MUST occur before any agent is spawned (zero token cost).

#### REQ: attempts-question

Unless `attempts=` was given or the user's message clearly states it, the skill MUST ask explicitly: 1 (fastest; failing units merge partial or abandon), 2 (recommended; one bounce with verifier findings), or 3 (highest salvage, ~50% more spend on failing units).

#### REQ: scope-cost-confirm

For whole-repo runs the skill MUST present the collected/estimated scale (packages, uncovered statements, projected agents) and recommend a subtree alternative, proceeding only on confirmation. Subtree runs proceed without the warning. The cost projection MUST be shown before launch, not after.

### Completion

#### REQ: completion-report

On workflow completion the skill MUST report: per-package pre→post coverage and uncovered counts with the total delta; integration failures, policy violations, and quarantined packages; new dependencies added; patterns harvested; the result branch `coverage/100-<stamp>` and the locations of REPORT.md / TEST-COVERAGE-OVERVIEW.md.

#### REQ: no-auto-merge

The skill MUST NOT merge results into the default branch automatically. It offers a merge only after showing the production-change diffs (seams or none, per policy) and the build/test/lint status of touched packages.

#### REQ: safe-abort

When the user asks to abort a running workflow, the skill MUST stop the task and explain the no-work-lost recovery: resume with `stamp=<same>` and the Salvage phase recovers committed work automatically. The skill MUST NOT hand-merge leftovers itself.

## Acceptance Criteria

### AC: preflight-blocks-before-spend (verifies REQ:preflight)

**Given** a machine without codegraph installed
**When** `/cover100 ./pkg/x/...` is invoked
**Then** the skill stops with install guidance and no workflow or agent is started

### AC: attempts-asked-when-unspecified (verifies REQ:attempts-question)

**Given** an invocation with no attempts argument and no stated preference
**When** the skill resolves inputs
**Then** the user is asked the 1/2/3 question with 2 marked recommended before launch

### AC: whole-repo-warned (verifies REQ:scope-cost-confirm)

**Given** `/cover100` with no path filter on a large repo
**When** inputs resolve
**Then** the user sees the projected scale and a subtree recommendation, and the run starts only on confirmation

### AC: results-not-merged-silently (verifies REQ:no-auto-merge)

**Given** a completed run with a green result branch
**When** the skill reports completion
**Then** the default branch is untouched and the report offers (not performs) a merge

### AC: abort-then-resume (verifies REQ:safe-abort)

**Given** a running workflow and a user message asking to abort
**When** the skill handles it
**Then** the task is stopped and the reply names the exact resume command with the current stamp

## Rehearse Integration

Skill UX ACs are conversation-protocol checks; they become scripted scenario walkthroughs (`_tests/`) once the plugin skill file exists.

## Open Questions

- Cost projection units: agents and tokens are honest but abstract — do we attempt a dollar estimate (model-price table to maintain) or keep token/agent counts only?

---
*This document follows the https://specscore.md/feature-specification*
