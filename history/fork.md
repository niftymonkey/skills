---
skill: fork
created: 2026-05-21
current-version: v2
status: published
---

# History: fork

## Problem

Mid-conversation, the user often realizes a separate task needs doing: an out-of-scope refactor, a bug to file, a prototype to build. Pursuing it in the current session dilutes the context and risks finishing neither task well. Compacting clobbers the current session's progress. What is needed is a way to branch the new task off into its own session, carrying the realization and the reasoning that surfaced it, while the current session stays focused on its own work.

`continue` already covers the vertical case (resume the *same* task in a fresh session). `fork` is the lateral counterpart: spin a *different*, newly-surfaced task off the current one.

## How it works (brief)

`/fork <what the next session will focus on>` writes a single-use handoff document to `./fork-<slug>.md` in the working directory, gitignored. The document carries only what the forked task needs, filtered by relevance to the stated purpose rather than recency in the conversation, and carried verbatim rather than summarized. Named content categories (the task, why it surfaced, hard-won understanding, what's already settled, orientation) act as indicators so research and the connections the conversation built do not get lost. A separate agent, possibly in a different harness, picks up the document and runs the task independently. The document is single-use and deleted once consumed.

See `SKILL.md` for the full guidance.

## Iteration log

### 2026-05-21: v1 (initial)

Adapted from Matt Pocock's `handoff` skill (`mattpocock/skills`, `skills/productivity/handoff`). Started from his verbatim text and applied four changes:

- Renamed `handoff` to `fork`; reworded the description for the branch framing (his "compact the current conversation" became "branch a newly-surfaced task off it").
- Storage moved from the OS temp directory to `./fork-<slug>.md` in the working directory, gitignored via `.git/info/exclude`. Consistent with `continue` after its v6 temp-dir reversal. A single-use deletion line was added since cwd no longer provides disposability for free.
- Added frontmatter hardening: `disable-model-invocation: true` and `allowed-tools`, so `fork` fires only on explicit invocation.
- Grafted in a content discipline adapted from `continue`: a relevance filter (carry what the forked task needs, verbatim, not a recency-biased summary) plus five named content categories. "Why it surfaced" and "Hard-won understanding" (research findings and the connections the conversation built) are first-class indicators, because they are the costliest things for a cold agent to rederive and the easiest to lose.

Pocock's repo is MIT-licensed; an `> Adapted from` blockquote at the top of `SKILL.md` credits the source.

`continue`'s v7 three-tier triage was deliberately not ported. That triage exists because `continue.md` is updated in place and accretes across resume cycles. `fork` is single-use, so nothing accumulates and there is nothing to triage. What ported is the relevance judgment underneath it, applied once at write time.

### 2026-06-03, v2 (promoted to publishable repo)

Promoted as-is; no remediation. The `promote-skill` review found zero portability issues: no personal paths, no cross-skill dependencies in the runtime file, and zero em dashes anywhere. The `> Adapted from` Pocock attribution (MIT) was verified at review time (source URL resolves, GitHub reports the repo license as MIT) and travels verbatim at the top of `SKILL.md`.

No companion files. fork is the first promoted skill whose runtime `SKILL.md` is itself the adapted-from-Pocock artifact (architect-deep and explore-idea carried their Pocock content in separate companion files instead). The attribution is file-level, so the whole `SKILL.md` moves intact.

Source-of-truth moved from `~/dev/niftymonkey/claude/skills/fork/` to `~/dev/niftymonkey/skills/skills/fork/` + `history/fork.md`.

## Design uncertainties

- Non-git working directories: the gitignore step assumes a git repo. It should degrade gracefully if `.git` is absent. Not yet addressed.
- The round-trip pattern (a forked session handing findings back to the parent via another `fork`) is currently undocumented in the skill body. Left out to keep the skill lean; revisit if the pattern gets used often.
- Whether the shared house-style guidance (suggested-skills, don't-duplicate, redact-secrets, the relevance discipline) should be extracted into a companion file shared with `continue`. Deferred until `fork` stabilizes: two real adapters make the seam real, one does not.

## Files

- `SKILL.md`: frontmatter, attribution blockquote, the branch-and-store instruction, the relevance filter with five named content categories, suggested-skills, don't-duplicate, redact-secrets, argument tailoring, single-use deletion.
- `history/fork.md`: this file.
