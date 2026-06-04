---
skill: write-prd
created: 2026-05-03
current-version: v2
status: published
---

# History: write-prd

## Problem

Writing a good PRD is a multi-stage task that's easy to compress into "fill in the template and move on", and a compressed PRD is the leading cause of misaligned implementations. The information the PRD needs (problem framing, user stories, decisions made, modules to build) lives in different places: the user's head, the existing codebase, prior `explore-idea` artifacts, and architecture choices that haven't been made yet. None of those sources speak the PRD's language by default.

The skill exists to walk the full path: pre-fill from `explore-idea` output if it exists, verify assertions against the codebase, interview the user one decision-tree branch at a time, sketch major modules using `architect-deep` (which populates the Implementation Decisions section), and produce a structured PRD saved to the right location with optional GitHub-issue submission.

## How it works (brief)

Five-step flow that's skippable per-step when not needed: (1) check for `explore-idea` output to pre-fill understanding; (2) explore the codebase to verify assertions (skip for greenfield); (3) interview the user relentlessly down each branch of the design tree; (4) sketch the major modules (using `architect-deep` if installed, otherwise the deep-module method directly), whose output populates the PRD's Implementation Decisions section; (5) save the PRD per the embedded template, choosing the location (`./docs/` for project-specific, or an external directory for higher-level or private planning), then optionally submit as a GitHub issue.

The composition with `explore-idea` (input) and `architect-deep` (optional sub-skill in step 4) is what makes the skill work: it doesn't reinvent interview discipline or deep-module design, it routes to the right specialists.

See `SKILL.md` for the full PRD template and step-by-step behavior.

## Iteration log

### 2026-05-03, v1 (initial)

> **Note:** The originating conversation wasn't archived, so iteration detail below is reconstructed from the current `SKILL.md`. Filling in actual rationale is a TODO.

Established the five-step flow with explicit skippability per step, the composition contracts with `explore-idea` (consume its output if present, shorten the interview for resolved areas) and `architect-deep` (invoke as sub-skill in step 4, place its output in Implementation Decisions), the save-location decision tree (`./docs/` vs an external directory), and the embedded PRD template (Problem Statement, Solution, User Stories with numbered as-a/I-want/so-that format, plus the implementation sections populated by `architect-deep`).

The post-save GitHub-issue submission was set up as an optional follow-up rather than an automatic step, preserving user agency over what becomes public.

### 2026-05-15, locked to user-only invocation

Added `disable-model-invocation: true`. write-prd launches a multi-phase PRD interview: a deliberate `/write-prd`, not something Claude should start on its own. Step 6 skill-locking review.

### 2026-06-03, v2 (promoted to publishable repo)

Remediation applied during the `promote-skill` review before the move:

- SKILL.md: the hardcoded personal external save path was removed (it doesn't exist on a stranger's machine), along with the private-content examples that named it. The save-location step now offers `./docs/` in-repo or a generic external directory and asks the user where, rather than assuming a path. This mirrors the personal path `explore-idea` dropped at its own promotion.
- SKILL.md: step 4 was softened so the skill degrades gracefully standalone. It now uses `architect-deep` if that companion is installed and otherwise sketches the modules directly via the deep-module method, instead of hard-instructing a skill a stranger may not have. `architect-deep` is already published, so it is named as an optional companion.
- SKILL.md: em dashes removed throughout (the author's own file), rewritten with commas, colons, and sentence breaks.
- history: the personal save path and its private-content examples were redacted from this file so nothing tied to the author's setup reaches the public history.

No companion files: write-prd composes with `explore-idea` (optional input) and `architect-deep` (optional sub-skill), both already in this repo, purely by naming them, so there is nothing to symlink.

Source-of-truth moved from `~/dev/niftymonkey/claude/skills/write-prd/` to `~/dev/niftymonkey/skills/skills/write-prd/` + `history/write-prd.md`.

## Design uncertainties

- The "skip steps if you don't consider them necessary" allowance is permissive; over time it may need to be tightened with explicit gates (e.g., step 2 mandatory if any code exists, step 4 mandatory if new modules surface).
- The PRD template (User Stories format especially) is opinionated about the as-a/I-want/so-that structure; some projects may want freeform problem-shaping prose instead. Whether to make the template configurable is open.
- The composition with `architect-deep` is via skill-invocation but the handoff format isn't pinned: what should `architect-deep` return to `write-prd` for the Implementation Decisions section? Currently implicit.
- The save-location decision tree is two-option; some content fits neither (e.g., personal-but-not-secret docs that should live in a notes vault). May need a third option or an "ask the user" fallback.

## Files

- `SKILL.md`: five-step PRD authoring flow, PRD template, save-location decision tree, GitHub-issue submission option
- `history/write-prd.md`: this file
