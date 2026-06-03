---
skill: architect-deep
created: 2026-05-03
current-version: v2
status: published
---

# History: architect-deep

## Problem

Pocock's `improve-codebase-architecture` finds shallow modules in code that already exists, a reactive lens applied after commitments are made. The gap: at design time, before any code is written, there's no skill applying the same deep-module discipline. Architecture choices made before code commits are the cheapest to revise and the most expensive to live with if they're wrong, so this is where the lens belongs.

This skill exists to apply the deep-module method proactively: pre-deletion test, dependency classification (in-process / local-substitutable / remote-but-owned / true-external), interface design via parallel sub-agents ("Design It Twice"). It composes with `explore-idea`, `write-prd` (step 4), and `prd-to-plan` (step 3); its output populates their architecture sections rather than producing standalone artifacts.

## How it works (brief)

Lean `SKILL.md` plus three companion docs the skill loads on demand: `LANGUAGE.md` (shared vocabulary: module, interface, depth, seam, adapter, leverage, locality), `DEEPENING.md` (four-category dependency classification + seam discipline + replace-don't-layer testing strategy), `INTERFACE-DESIGN.md` (parallel sub-agent pattern for exploring radically different interfaces of a chosen module).

The companion docs are inlined (with MIT attribution) from Pocock's `improve-codebase-architecture` skill: same vocabulary, same principles, applied in the opposite direction.

See `SKILL.md` for the actual logic.

## Iteration log

### 2026-05-03, v1 (initial)

> **Note:** The originating conversation wasn't archived, so iteration detail below is reconstructed from the current `SKILL.md` rather than contemporaneous notes. Filling in actual rationale is a TODO.

Established the skill as the proactive companion to Pocock's reactive `improve-codebase-architecture`. Vocabulary, deepening process, and interface-design process were lifted from upstream (originally via symlinks; later inlined with attribution, see 2026-05-10 below). The proactive-specific content is in `SKILL.md`: the "When to fire" gate (new boundaries being drawn: module/package/service naming, PRD module sketches, plan step 3, late explore-idea, ad-hoc design talk), the "do not fire" gate (function-level decisions, refactoring existing code, pure phasing), and the input-gathering priority (PRD draft → explore-idea output → conversation).

### 2026-05-10, v1.1 (inlined companion docs)

The three companion files (`LANGUAGE.md`, `INTERFACE-DESIGN.md`, `DEEPENING.md`) were originally symlinks into `~/.agents/skills/improve-codebase-architecture/`. Replaced with verbatim copies + visible MIT attribution at the top of each file. Motivation: portability for the upcoming publishable skills repo, since symlinks resolve to nothing on someone else's machine.

### 2026-06-03, v2 (promoted to publishable repo)

Remediation applied during the `promote-skill` review before the move:

- Description (`SKILL.md`) reworded to lead with the standalone path and drop the named unpublished skills (`explore-idea`, `write-prd`, `prd-to-plan`); the em dash was removed.
- "Relationship to other skills" softened so the references read as optional companions, not requirements; noted that `improve-codebase-architecture` lives at `mattpocock/skills`.
- Em dashes removed throughout `SKILL.md` (the author's own file). The three Pocock-adapted companion files were left verbatim, per the rule that adapted-from-external files stay as-is.

Promoted with the companion files as inline copies (self-contained). A later effort will migrate Pocock-derived companions to a dependency model (install the upstream skill, symlink to it) rather than vendoring copies.

Source-of-truth moved from `~/dev/niftymonkey/claude/skills/architect-deep/` to `~/dev/niftymonkey/skills/skills/architect-deep/` + `history/architect-deep.md`.

## Design uncertainties

- The "Pre-deletion test" framing works well for new-module decisions but is awkward for boundary-redrawing decisions (where the question is "should these two modules merge?" not "should this module exist?"). May want a sibling framing.
- The companion-doc inlining freezes a snapshot of Pocock's upstream; drift will accumulate. Need a periodic re-sync or an automated diff against upstream. (The planned dependency-model migration would resolve this by referencing the installed upstream directly.)
- The composition contract with `write-prd` step 4 and `prd-to-plan` step 3 is implicit; an explicit handoff format (what does this skill *return* to the caller?) hasn't been pinned down.

## Files

- `SKILL.md`: main entry, when-to-fire gates, process (gather input → list candidates → classify dependencies → design interface)
- `LANGUAGE.md`: shared vocabulary (adapted from Pocock, MIT)
- `DEEPENING.md`: dependency categories + seam discipline (adapted from Pocock, MIT)
- `INTERFACE-DESIGN.md`: parallel sub-agent interface-design pattern (adapted from Pocock, MIT)
- `history/architect-deep.md`: this file
