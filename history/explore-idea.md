---
skill: explore-idea
created: 2026-05-04
current-version: v6
status: published
---

# History: explore-idea

## Problem

Plans that look good in prose collapse the moment you walk down each branch of the decision tree. The default mode of "explain the idea once" lets fuzzy language, contradictions between assertions and code, and unresolved sub-decisions slip through unchallenged, and they reappear during implementation, costing far more time than the interview would have.

This skill exists to make the interview discipline reliable: one question at a time, recommend an answer with each question, check the codebase before asking what the code already shows, surface contradictions when user assertions disagree with reality, sharpen overloaded terms, and stress-test boundaries with concrete edge-case scenarios.

A secondary purpose: when an interview crystallizes durable knowledge (domain terms, hard-to-reverse decisions), capture it inline as `DOMAIN-LANGUAGE.md` glossary entries and ADRs in `docs/adr/`, without forcing the user to manage either artifact manually.

## How it works (brief)

`SKILL.md` defines the interview discipline (one question at a time, lead with recommended answer, cross-reference assertions against code, sharpen fuzzy language, stress-test with edge cases) and the protocol for maintaining a `DOMAIN-LANGUAGE.md` glossary and ADRs. The glossary is offered up front, at session start, with its value stated: seed it from the codebase, build it as the interview goes, or skip it. ADRs are offered conservatively, mid-session, only when a decision clears the offer gate.

Consent is the key design choice: no project-shaping file (`DOMAIN-LANGUAGE.md` or `docs/adr/`) is created without the user's buy-in. Both are conventions the user should adopt deliberately, not have appear unannounced. After buy-in, maintenance becomes seamless, and the user never has to manage the files manually.

Format specifications for `DOMAIN-LANGUAGE.md` and ADR files are adapted (with MIT attribution) from Pocock's `grill-with-docs` skill.

See `SKILL.md` for the actual logic and end-of-session artifact format.

## Iteration log

### 2026-05-04, v1 (initial)

> **Note:** The originating conversation wasn't archived, so iteration detail below is reconstructed from the current `SKILL.md`. Filling in actual rationale is a TODO.

Established the interview discipline (six rules), the opt-in protocol for `CONTEXT.md` / ADRs, the ADR offer gate (hard-to-reverse + surprising-without-context + result-of-real-trade-off, all three required), and the end-of-session artifact structure (Problem/Opportunity, Target Users, Core Requirements with must-have vs nice-to-have, Key Decisions Made, Constraints/Boundaries, Open Questions).

Multi-context repo support landed in v1: `CONTEXT-MAP.md` at the root lists contexts with their locations and relationships; the skill infers which structure applies and reads the relevant `CONTEXT.md` for the current topic.

The format specs for both artifacts (`CONTEXT-FORMAT.md`, `ADR-FORMAT.md`) were lifted from Pocock's `grill-with-docs` skill: originally via symlinks, later inlined (see 2026-05-10).

### 2026-05-10, v1.1 (inlined format specs)

The two companion files were originally symlinks into `~/.agents/skills/grill-with-docs/`. Replaced with verbatim copies + visible MIT attribution at the top of each file. Motivation: portability for the publishable skills repo.

### 2026-05-15, v2 (DOMAIN-LANGUAGE rename + startup offer + grill-with-docs reconciliation)

Reconciliation pass against current upstream `grill-with-docs` (explore-idea's origin skill), plus a rename to remove a naming collision. The glossary artifact `CONTEXT.md` was renamed `DOMAIN-LANGUAGE.md` (and `CONTEXT-MAP.md` → `DOMAIN-LANGUAGE-MAP.md`, `CONTEXT-FORMAT.md` → `DOMAIN-LANGUAGE-FORMAT.md`): "context" collided with the LLM-context sense and obscured the file's purpose. The domain-language step was promoted from a buried mid-session opt-in to a prominent, value-explaining startup offer. Two items were folded in from current upstream; nothing else had drifted (the format specs were byte-identical to Pocock's).

- SKILL.md: restructured into `<what-to-do>` / `<supporting-info>` tags (adopted from upstream)
- SKILL.md: new "Before the interview" startup offer: seed / build-as-we-go / skip, value stated
- SKILL.md: folded in upstream guardrail: `DOMAIN-LANGUAGE.md` is a glossary and nothing else
- SKILL.md: folded in upstream ASCII file-structure diagrams (single- and multi-context)
- SKILL.md: description shortened ~480→~90 chars for `/` autocomplete readability
- SKILL.md: end-of-session save location: `./docs/` default; dropped the personal `~/dev/niftymonkey/plans/` path
- CONTEXT-FORMAT.md → DOMAIN-LANGUAGE-FORMAT.md: renamed; internal references updated

### 2026-05-15, v3 (decouple from seed-domain-language)

Companion change to the new `seed-domain-language` skill. The `DOMAIN-LANGUAGE.md` glossary format spec is now a shared contract between two skills, so `DOMAIN-LANGUAGE-FORMAT.md` moved out of this skill to `companion-files/DOMAIN-LANGUAGE-FORMAT.md` (same pattern as `CHECKS.md`). The startup offer's option (1) "Seed it now" changed from explore-idea doing its own inline codebase pass to a soft pointer at the dedicated `seed-domain-language` skill. The two skills are decoupled: they integrate only through the `DOMAIN-LANGUAGE.md` artifact and never invoke each other.

- SKILL.md: option (1) now points at the `seed-domain-language` skill instead of describing an inline pass
- SKILL.md: `DOMAIN-LANGUAGE-FORMAT.md` links repointed to `../../companion-files/`
- DOMAIN-LANGUAGE-FORMAT.md: moved to `companion-files/` (shared contract)

### 2026-05-15, locked to user-only invocation

Added `disable-model-invocation: true`. explore-idea launches a relentless interview that takes over the session, so it should be a deliberate `/explore-idea`, not model-triggered. Step 6 skill-locking review.

### 2026-06-03, v4 (promoted to publishable repo)

Remediation applied during the `promote-skill` review before the move:

- SKILL.md: em dashes removed throughout (the author's own file), rewritten with commas, colons, and sentence breaks. The Pocock-adapted `ADR-FORMAT.md` and `DOMAIN-LANGUAGE-FORMAT.md` were left verbatim, per the rule that adapted-from-external files stay as-is.
- SKILL.md: the `DOMAIN-LANGUAGE-FORMAT.md` link was repointed from `../../companion-files/` to an in-directory reference, since a single-skill install carries only the skill directory.
- SKILL.md: the `seed-domain-language` startup-offer reference was softened to read as a named companion rather than a hard dependency.

Companion travel: `DOMAIN-LANGUAGE-FORMAT.md` is shared with `seed-domain-language`, so it was promoted under the single-source pattern rather than copied. The canonical file lives at `companion-files/DOMAIN-LANGUAGE-FORMAT.md`; each consuming skill carries a relative symlink (`skills/<name>/DOMAIN-LANGUAGE-FORMAT.md -> ../../companion-files/DOMAIN-LANGUAGE-FORMAT.md`). The `skills` CLI dereferences the symlink on install (`cp` with `dereference: true`), so a stranger receives real file content rather than a dangling link. Verified before commit by replicating the install-shape copy. This is the first use of the published repo's `companion-files/` single-source pattern, and it sets the template for `seed-domain-language` and `kickoff`.

Source-of-truth moved from `~/dev/niftymonkey/claude/skills/explore-idea/` to `~/dev/niftymonkey/skills/skills/explore-idea/` + `history/explore-idea.md`. The shared `companion-files/DOMAIN-LANGUAGE-FORMAT.md` stays in the source repo too, still used there by the local `seed-domain-language` skill until that skill is promoted.

### 2026-06-15, v5 (folded in the no-complexity-warnings interview rule)

Added an interview rule: during endgame / dream-mapping, capture the user's ambitious choices without editorializing implementation complexity (defer feasibility to a separate phasing pass), while still surfacing genuine logical contradictions. Promoted from a dormant project-scoped memory (`feedback_no_complexity_warnings_during_endgame_mapping`) that only loaded from `~/dev` and so rarely fired, into the skill where it actually belongs. Origin of the rule: a 2026-05-23 explore session where per-choice complexity caveats pushed the user toward conservative answers and corrupted the dream-mapping data.

### 2026-08-02, v6 (description carries the whole boundary against `grilling`)

Came out of a kit-wide evaluation that found three grilling-lineage skills doing one job, with
nothing in any description saying which to reach for. `explore-idea` descends from Pocock's
`grill-with-docs` and reproduces the bare `grilling` interview protocol almost word for word, so the
overlap is real rather than superficial.

A first attempt split them by location: `explore-idea` inside a code repository, `grilling` outside
one. Wrong, and the user caught it. That reading came from this skill's body being dense with
repository material (`DOMAIN-LANGUAGE.md`, exploring the codebase, ADRs), but that section is gated
on "**when** the session is rooted in a code repository". An enrichment, not a scope.

**The axis is output.** `grilling` is a conversation that ends at shared understanding and leaves
nothing behind. This skill is the same interview that lands as a document, carrying a raw
just-thought-of-it idea through to the PRD-shaped end-of-session artifact. That is what it was built
for.

The new description states both sides, and deliberately claims the "grill me" phrasing when an idea
should survive as a document. **`grilling` itself was left byte-identical to upstream on purpose:**
it is a third-party skill, so a boundary written into its description would be silently erased by
the next `npx skills update` while still reading as though both sides were stated. When two skills
need telling apart and only one is yours, the whole distinction goes in the one you own.

- SKILL.md: description rewritten from a one-line summary to a routing statement naming the artifact
  it produces and when to use `grilling` instead

## Design uncertainties

- The opt-in gate ("ask once per project, drop forever if declined") is the right default for human-driven projects but may be too cautious for greenfield where the user just hasn't thought about it yet. Should there be a re-offer trigger after N decisions accumulate without `DOMAIN-LANGUAGE.md`?
- The ADR offer gate (3-of-3 conjunction) is strict; some decisions worth recording fail one criterion. Loosening to 2-of-3 may produce too much noise. Needs real-world calibration.
- The skill doesn't currently challenge a user assertion that contradicts an existing ADR, only the existing `DOMAIN-LANGUAGE.md` glossary. ADRs probably deserve the same treatment.
- Cross-reference against code is described as a discipline but not as a step; in practice it has to be invoked as a sub-task. Open whether to formalize it as a checkpoint.

## Files

- `SKILL.md`: interview discipline, startup offer, ADR offer gate, end-of-session artifact format
- `ADR-FORMAT.md`: ADR format spec + when-to-offer criteria (adapted from Pocock, MIT)
- `DOMAIN-LANGUAGE-FORMAT.md`: glossary format spec, a symlink to the shared `companion-files/DOMAIN-LANGUAGE-FORMAT.md` (adapted from Pocock, MIT)
- `history/explore-idea.md`: this file
