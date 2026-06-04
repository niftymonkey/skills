---
skill: seed-domain-language
created: 2026-05-15
current-version: v2
status: published
---

# History: seed-domain-language

## Problem

Starting an `explore-idea` grilling session in an established, code-heavy repository with an empty glossary wastes the first several sessions re-deriving domain vocabulary that is already sitting in the code. `explore-idea` builds the glossary incrementally, excellent for a project from day one but slow to catch up on a repo with existing architecture.

This skill bootstraps the glossary in one pass: it mines the domain vocabulary already present in a repository and proposes an initial `DOMAIN-LANGUAGE.md` for review, so the first real `explore-idea` session starts on a populated base.

## How it works (brief)

A linear pass (harvest → distill → write) with the user in the loop at three gates (context partitioning, per-cluster review, final whole-draft confirmation). The harvest gathers candidate vocabulary from ranked sources, adapting to repo type: code repos lean on identifiers, prose/documentation repos lean on headings and recurring terms. Distillation is the one deep step: it collapses synonyms, flags ambiguity, drops generic programming terms, and chooses canonical names.

`seed-domain-language` and `explore-idea` are decoupled. They share one contract (`companion-files/DOMAIN-LANGUAGE-FORMAT.md`) and one artifact (`DOMAIN-LANGUAGE.md` on disk). Neither skill invokes the other: this skill creates the glossary, `explore-idea` maintains it.

See `SKILL.md` for the workflow.

## Iteration log

### 2026-05-15, v1 (initial)

Created during a skill-library evaluation pass, as a companion to `explore-idea`. Designed via `architect-deep`: the pre-deletion test showed a single deep module (term distillation) bracketed by a thin harvest and a thin write step, so the skill is a linear flow rather than a set of modules. The architecturally significant seam is the `DOMAIN-LANGUAGE.md` format (shared with `explore-idea`), which is why the format spec lives in `companion-files/` rather than inside either skill, and why the two skills integrate only through the artifact and never invoke each other. Three human-in-the-loop gates (context, per-cluster, final). Scan strategy is ranked by signal and adapts to repo type.

### 2026-05-15, locked to user-only invocation

Added `disable-model-invocation: true`. Seeding a glossary is a deliberate one-time pass that writes files: a `/seed-domain-language` invocation, not model-triggered. Step 6 skill-locking review.

### 2026-06-03, v2 (promoted to publishable repo)

Promoted with no review remediation. The `promote-skill` portability review surfaced zero automated findings and no blockers: the only cross-skill reference is `explore-idea`, an already-published, decoupled companion that integrates solely through the on-disk `DOMAIN-LANGUAGE.md` artifact, so the references name it plainly rather than softening it.

Author-file cleanups applied during the move:

- SKILL.md: em dashes removed throughout (the author's own file), rewritten with commas, colons, parentheses, and sentence breaks. The Pocock-adapted `DOMAIN-LANGUAGE-FORMAT.md` was left verbatim, per the rule that adapted-from-external files stay as-is.
- SKILL.md: the three `DOMAIN-LANGUAGE-FORMAT.md` links were repointed from `../../companion-files/` to an in-directory reference, since a single-skill install carries only the skill directory.

Companion travel: `DOMAIN-LANGUAGE-FORMAT.md` is shared with `explore-idea`, so it travels under the `companion-files/` single-source pattern rather than being copied. This is the pattern's second consumer: the canonical file stays at `companion-files/DOMAIN-LANGUAGE-FORMAT.md`, and this skill carries a relative symlink (`skills/seed-domain-language/DOMAIN-LANGUAGE-FORMAT.md -> ../../companion-files/DOMAIN-LANGUAGE-FORMAT.md`). The `skills` CLI dereferences the symlink on install, so a stranger receives real file content rather than a dangling link. With both consumers now promoted, the incubation repo's own `companion-files/DOMAIN-LANGUAGE-FORMAT.md` copy was retired (no remaining local consumer); the published repo is now its sole source of truth.

Source-of-truth moved from `~/dev/niftymonkey/claude/skills/seed-domain-language/` to `~/dev/niftymonkey/skills/skills/seed-domain-language/` + `history/seed-domain-language.md`.

## Design uncertainties

- Re-seeding a repo that already has a `DOMAIN-LANGUAGE.md`: v1 surfaces the conflict and asks the user rather than auto-refreshing. Merging a fresh harvest with existing human edits is not designed yet.
- Cluster granularity for the review gates is left to judgment, not specified. May need a concrete heuristic after real use.
- Multi-context detection is deliberately not automated; it is surfaced as a question at the context gate. Large monorepos may make even posing the question hard; revisit if so.

## Files

- `SKILL.md`: the seeding workflow
- `DOMAIN-LANGUAGE-FORMAT.md`: glossary format spec, a symlink to the shared `companion-files/DOMAIN-LANGUAGE-FORMAT.md` (adapted from Pocock, MIT)
- `history/seed-domain-language.md`: this file
