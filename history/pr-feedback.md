---
skill: pr-feedback
created: 2026-05-06
current-version: v3
status: published
---

# pr-feedback — History

## Problem

PR review comments accumulate quickly, and the default "fix what I agree with, ignore the rest" pattern leaves threads unresolved. Review bots (CodeRabbit especially) track resolution per-comment — a top-level reply summarizing multiple fixes doesn't close any of the individual threads, so the PR looks like it has unaddressed feedback even when everything was handled. The discipline gap is small but the noise it creates is large.

The skill exists to enforce a per-comment resolution discipline: every review comment gets either a code fix or an explicit inline reasoning for dismissal — never silence, never top-level summaries, never batched responses.

## How it works (brief)

Fetches review comments and review-body comments via `gh api`, filters out self-comments and already-replied threads, presents grouped-by-file for triage with a recommended fix/dismiss disposition, applies the chosen actions (commits + push for fixes; inline reply via `in_reply_to` for dismissals), and reports unaddressed remainders.

For nested findings inside a review body (CodeRabbit's "minor comments" block being the canonical case), there is no per-comment reply ID; the skill posts a fresh inline review comment at the affected file/line instead, so the reviewer can track resolution at file granularity.

See `SKILL.md` for full per-step behavior.

## Iteration log

### 2026-05-06 — v1 (initial)

> **Note:** The originating conversation wasn't archived, so v1 detail below is reconstructed from the current `SKILL.md`. Filling in actual rationale is a TODO.

Covered: PR-number argument (with current-branch detection fallback), fetch + filter logic, grouped-by-file triage, the fix vs dismiss decision, the `in_reply_to` reply pattern (with the `-F` vs `-f` gotcha about integer-vs-string `in_reply_to_id` documented inline). Established the hard rules: never leave a comment without a response, never batch into a top-level PR comment, dismissal responses must be specific.

### 2026-05-10 — v2 (review-body nested findings)

CodeRabbit's review-body "minor comments" blocks surfaced a gap: those findings have no individual reply ID, so `in_reply_to` doesn't work. The temptation was to summarize them in a top-level PR comment — but that leaves every finding unresolved as far as CodeRabbit's per-thread tracking is concerned.

Added a new section to `SKILL.md`: for findings nested inside a review body, post a fresh inline review comment at the file/line each finding references (using `commit_id` + `path` + `line` + `side`). This creates a per-file chat thread reviewers can resolve individually. Reinforced in the hard-rules section: never use a top-level PR comment for these.

### 2026-05-10 — v3 (promoted to publishable repo, with portability polish)

First skill graduated from `niftymonkey/claude` to `niftymonkey/skills` via the `/promote-skill` workflow. Three minor concerns + one false positive surfaced during review were remediated inline before promotion:

- **description frontmatter** — added "GitHub" early so strangers immediately know the platform expectation, added "specific" before "reasoning for dismissing" to match the skill's own internal discipline, added "Uses `gh` CLI" at the end so the install dependency is upfront rather than discovered on first run.
- **commit-message instruction** (SKILL.md step 5) — softened from a hardcoded `fix: address review feedback on PR #{number}` to *"using your project's commit-message convention. For conventional commits: `fix: address …` For other styles, match what the rest of the PR's branch uses."* Removes the assumption that every project uses conventional commits.
- **typecheck/tests instruction** (SKILL.md step 5) — softened from "Run typecheck and tests after fixes, before committing" to "…if the project has them." Removes the assumption that every project has both.
- **false positive** (SKILL.md step 4) — changed *"brief description of what changed"* to *"short description of what changed"* in the example reply body. The grep pattern in `promote-skill/CHECKS.md` matches `\bbrief\b` because `brief` is a custom CLI; sidestepping the English word avoids re-flagging on future runs without needing a `publishability-notes:` suppression.

Source-of-truth moved from `~/dev/niftymonkey/claude/skills/pr-feedback/` to `~/dev/niftymonkey/skills/skills/pr-feedback/` (runtime content) + `~/dev/niftymonkey/skills/history/pr-feedback.md` (this file). Status flipped to `published`.

## Design uncertainties

- The skill currently fetches all review comments unconditionally; large PRs with hundreds of comments may benefit from pagination or an "only show unresolved" filter.
- The fix/dismiss recommendation logic is qualitative (Claude judges what to suggest); some teams may want a stricter mode that flags any dismissal for explicit user confirmation.
- For review bots that nest findings in slightly different ways (other than CodeRabbit's pattern), the file/line fallback may not always be reliable — needs more real-world exposure to other bots.
- No handling yet for "request changes" reviews that block merge — the skill addresses comments but doesn't dismiss the review state.
- Now that the skill is in the publishable repo, drift between the local edits and the published copy is no longer possible (single source of truth), but iteration discipline shifts — edits to `skills/pr-feedback/SKILL.md` must be paired with a new dated entry in `history/pr-feedback.md` and a `current-version` bump.

## Files

In `niftymonkey/skills/skills/pr-feedback/` (runtime content, installed by `npx skills add`):

- `SKILL.md` — argument handling, fetch + filter, grouped triage, fix vs dismiss flow, inline reply pattern, review-body-nested-findings handling, commit + push, hard rules

In `niftymonkey/skills/history/` (archival, not installed):

- `pr-feedback.md` — this file (renamed from `HISTORY.md` during promotion)
