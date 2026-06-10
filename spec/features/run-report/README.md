---
format: https://specscore.md/feature-specification
status: Approved
---
# Feature: Run report

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/cover100/spec/features/run-report?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/cover100/spec/features/run-report?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/cover100/spec/features/run-report?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/cover100/spec/features/run-report?op=request-change) |
**Status:** Approved
**Source Ideas:** cover100
**Grade:** A

## Summary

Every run ends with `REPORT.md` on the result branch: an executive summary a human who didn't run the tool can read in two minutes, followed by the receipts. It is the shareable artifact — the screenshot in the launch post, the consultant's client deliverable, the PR-body attachment in the OSS campaign.

## Problem

The run's raw artifacts (overview JSON, per-package gap registers, observation notes) are written for agents and operators. Adoption depends on a human-grade document that needs zero context to be convincing.

## Behavior

### Content

#### REQ: report-exec-summary

REPORT.md MUST open with an executive summary containing: coverage before → after (per the profile's metric, named explicitly — "statement" or "line"), number of tests added and packages touched, count of documented gaps, count of quarantined packages, count of suspected bugs and other code-review leads found, dependencies added, and the run's spend (tokens and agent count; dollar estimate when a price table is available).

#### REQ: report-sections

After the summary, REPORT.md MUST include: a per-package coverage table (pre/post/uncovered/status: covered | documented-gap | partial | quarantined); the gap register roll-up (why-type taxonomy counts with the refactor each gap needs); a "Next-run instructions" section (per the orchestrator's cross-run loop: unresolved verifier issues, coverable-gap flags, dropped/abandoned packages — imperative, per-package, deduplicated; "(none)" when everything integrated clean); code-review leads (from CODE-REVIEW-NOTES.md, security and suspected-bugs first, explicitly marked unverified); dependencies added; and quarantine list with reasons.

#### REQ: report-accuracy

Every number in REPORT.md MUST come from collect/verifier outputs recorded during the run — never estimated or recomputed by the report writer. If a value is unavailable, the report writes "not measured", never a guess.

#### REQ: report-portability

REPORT.md MUST be self-contained and portable: no absolute paths, no references to the operator's machine, environment, or session; relative repo paths only. It MUST render correctly as a standalone GitHub Markdown document.

## Acceptance Criteria

### AC: two-minute-summary (verifies REQ:report-exec-summary)

**Given** a completed run
**When** REPORT.md is opened
**Then** the first screen contains before/after coverage, tests added, gaps, suspected bugs, and spend without scrolling into per-package detail

### AC: numbers-traceable (verifies REQ:report-accuracy)

**Given** any coverage percentage in the report
**When** compared with the run's collect/verify outputs
**Then** it matches exactly

### AC: portable-document (verifies REQ:report-portability)

**Given** REPORT.md copied alone into a GitHub gist
**When** rendered
**Then** no link is broken by absolute paths and no machine-specific path appears

## Rehearse Integration

ACs become assertions over a fixture run's output once the orchestrator emits REPORT.md; report-portability is greppable (`/Users/`, `/home/`) and gets a deterministic stub first.

## Open Questions

- HTML export (for non-GitHub clients) in MVP or later?
- Should the report embed a compact methodology footer (what the gates verify) for first-time readers — likely yes, one paragraph?

---
*This document follows the https://specscore.md/feature-specification*
