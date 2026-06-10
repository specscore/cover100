---
format: https://specscore.md/feature-specification
status: Approved
---
# Feature: Plugin packaging and distribution

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/cover100/spec/features/plugin-packaging-and-distribution?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/cover100/spec/features/plugin-packaging-and-distribution?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/cover100/spec/features/plugin-packaging-and-distribution?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/cover100/spec/features/plugin-packaging-and-distribution?op=request-change) |
**Status:** Approved
**Source Ideas:** cover100
**Grade:** A

## Summary

How cover100 ships: a Claude Code plugin repo (skill + bundled workflow + language profiles) installable in one command from the specscore marketplace, with pinned versions and a documented codegraph install path per OS.

## Problem

The Idea's second must-be-true assumption is that users will tolerate an external-CLI prerequisite; the only mitigation is install friction so low it never gets the chance to annoy. Distribution is product, not afterthought.

## Behavior

### Layout

#### REQ: plugin-layout

The repo MUST follow the Claude Code plugin layout: `.claude-plugin/plugin.json` (name `cover100`, semver version, description), `skills/cover100/SKILL.md` (the `/cover100` skill), `workflows/cover100.js` (the orchestrator) plus per-profile scripts, `profiles/<lang>/` directories per the language-profile contract, and `assets/` for fixed files installed verbatim into user repos (e.g. the `.cover100/README.md` template). The skill MUST reference the workflow by its in-plugin path so the installed plugin is self-contained.

#### REQ: version-pinning

Each plugin release MUST bundle the exact orchestrator script and profiles it was tested with — no fetch-at-runtime of unpinned code. Upgrades happen through plugin version updates only; the changelog MUST note orchestrator behavior changes that affect resume compatibility.

### Install

#### REQ: one-command-install

Installation MUST be one command from a clean Claude Code setup (marketplace add + plugin install), verified on macOS and Linux. The README MUST show the full path from zero to first run in ≤5 steps including codegraph.

#### REQ: codegraph-install-path

The plugin MUST document one-command codegraph installation per OS (brew tap for macOS, script or binary release for Linux), and the skill preflight MUST link to exactly that documentation when codegraph is missing. The codegraph requirement is hard: no grep-fallback mode ships.

### Project hygiene

#### REQ: oss-hygiene

The repo MUST carry: MIT LICENSE, README with the claim + demo + case-study link, CONTRIBUTING.md (including the profile-contribution path and the conformance tool), and CI running spec lint, profile conformance for all bundled profiles, and shellcheck on profile scripts.

## Acceptance Criteria

### AC: clean-machine-install (verifies REQ:one-command-install)

**Given** a machine with Claude Code but no cover100 or codegraph
**When** the README's install steps are followed verbatim
**Then** `/cover100` preflight passes within 5 steps total

### AC: self-contained-plugin (verifies REQ:plugin-layout)

**Given** the installed plugin cache directory
**When** `/cover100` launches a run with the network unavailable for plugin assets
**Then** the workflow and profiles load from the plugin directory alone

### AC: ci-gates (verifies REQ:oss-hygiene)

**Given** a PR adding a malformed profile
**When** CI runs
**Then** the conformance job fails naming the violated field

## Rehearse Integration

clean-machine-install is a release-checklist manual scenario; ci-gates becomes a real CI fixture test at repo bootstrap.

## Open Questions

- Marketplace listing(s): specscore's own marketplace repo only, or also community catalogs at launch?
- Windows: document as WSL-only at MVP?
- codegraph dependency policy: it is `@colbymchenry/codegraph` (github.com/colbymchenry/codegraph — MIT, ~46k stars, actively developed). Pin an exact tested version per plugin release. Influence path: contribute upstream (precision/language improvements benefit cover100 and earn visibility). Break-glass contingency only: MIT permits a Go port with golden-parity testing (~1-2 weeks of agent-driven work) if upstream ever stalls or turns hostile; do not build it speculatively. Distribution note: upstream bundles a ~115MB Node runtime — a contributed or upstream single-binary story would materially improve our install step.

---
*This document follows the https://specscore.md/feature-specification*
