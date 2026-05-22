---
name: continue
description: Write a handoff document in the current working directory so a fresh session can resume the work.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
disable-model-invocation: true
argument-hint: "[optional-slug]"
---

# /continue: session handoff

Write or update a handoff document in the current working directory that lets a brand-new conversation resume the current work without ramp-up. Per-project by virtue of cwd: multiple parallel sessions across different projects each get their own handoff naturally.

## Process

### 1. Pick the target file

- No argument → `./continue.md`
- Slug `foo` → `./continue-foo.md`

The slug exists so multiple in-flight tasks in the same project can each have their own handoff (e.g., `/continue auth` in one session, `/continue ui` in another).

### 2. Check gitignore status (first write only)

A handoff is personal session state: it should not be committed, and it should not appear in teammates' working trees if they pull. Default to `.git/info/exclude` (local, per-clone) over `.gitignore` (shared, committed).

- Check whether the target filename is already ignored: `git check-ignore -q <filename> && echo ignored || echo not-ignored`.
- If not ignored:
  - If `.git/info/exclude` exists, append the pattern there and tell the user.
  - If only `.gitignore` is in use, ask the user which file they prefer before modifying.
  - Recommended pattern: `continue.md` and `continue-*.md` (covers both default and slugged forms).
- If already ignored, say nothing.

Skip this step on updates to an existing handoff file.

### 3. Write or update the file

If the file does not exist, create it.

If it already exists, **update in place, never wholesale overwrite**. Add new findings rather than replacing, and move stale "What's Next" items to "What's Done" as they complete. On every update, triage what the file already holds per **Keep the handoff current** below; a handoff that only ever grows dilutes the very fresh start it exists to provide. Never discard the verdict of a failed attempt, even once its problem is solved: re-trying a dead end is expensive.

Include a small header at the top so the resuming session can tell how stale the file is:

> **Last updated:** <ISO timestamp>
> **Branch:** `<branch>` (<last commit oneline>)
> **Working tree:** clean | dirty (<n> files modified, <n> untracked)

Gather via `git rev-parse --abbrev-ref HEAD`, `git log -1 --oneline`, `git status --short` (summarize if long). On updates, refresh the header.

### 4. Tell the user how to resume

After writing, output a one-line resume prompt:

> Saved to `<path>`. To resume in a new conversation: **"Read `<path>` and let's pick up where we left off."**

## Content guidance

### Survey the full arc, not just the recent slice

The model's attention tilts toward recency. Without push-back, the handoff captures only the last 5-10% of the session and silently loses the earlier 90%+. Two kinds of content go in the doc, and they're handled differently:

- **Arc-bearing** (what happened across the whole session): completed work, decisions and the reasoning behind them, attempts that failed and why, discoveries that took effort. These reflect the full conversation. Failures especially: re-trying a failed approach is expensive; preserving one extra bullet costs nothing.
- **Current-focus** (where we are now, going forward): the active goal, the exact next step, immediate open questions. Recency is *correct* here; that's the point.

Conflating the two destroys the document in both directions. Preserve both as they're intended.

Long conversations past several hundred thousand tokens are likely operating on **compacted context**: the harness auto-summarizes earlier exchanges without invocation. Treat the summaries as canonical for the segments they cover; do not invent detail beyond them. When content is drawn from a summary, note that briefly.

### Keep the handoff current

A handoff updated across many resume cycles accretes context that has gone cold: failed attempts on problems since solved, research behind decisions since made. Cold context is not free. It costs tokens on every future resume and pulls the fresh session's attention toward work that no longer matters, the very dilution `/continue` exists to escape. On every update, triage what the file already holds into three destinations:

- **Active.** Pertains to the current goal or a next step. Stays inline, verbatim and rich. This is what the resume runs on; it never moves.
- **Superseded.** A failed attempt on a problem now solved, or research behind a decision now made and committed. Collapse to a one-line verdict (`tried X (deadlocked), Y (too slow); chose Z, see PR #14`). The narrative is disposable once its outcome lives in a commit, PR, or ADR. The verdict is not: keep it so the next session does not re-walk a dead end.
- **Revisitable.** Real knowledge, not tied to current focus, not captured in any durable artifact. This graduates out of the handoff: a decision to an ADR, domain knowledge to a project context doc, research or reference material to `docs/`. The handoff keeps a one-line pointer with enough of a hook to judge relevance later.

Collapse or drop something only against a citation: a commit, merged PR, or ADR that demonstrably captures its outcome. With nothing to point at, it is not safe to call cold, so it stays Active or graduates to Revisitable. When in doubt, never drop.

Graduating context to an ADR or `docs/` file creates a committed artifact, unlike the gitignored handoff. Propose each promotion and let the user confirm; do not create the file silently.

### Don't duplicate other artifacts

Do not duplicate content already captured in other artifacts (PRDs, plans, ADRs, research docs, GitHub issues, commits, diffs). Reference them by path or URL. The handoff is glue, not encyclopedia.

Exception for **research findings**: if research wasn't saved to its own doc (often the case), capture it inline with sources, exact strings, and conclusions. Redoing research is the most expensive thing for a fresh session to repeat.

### Suggest next-session skills

Suggest skills the next session should use, if any.

### Redact secrets

A handoff lands in the working directory and can be opened, screen-shared, or accidentally committed even when gitignored. Strip API keys, tokens, passwords, connection strings, and PII before writing. Reference a secret-bearing file by path rather than inlining its contents.

## Cleanup

Don't delete the handoff file automatically. The user decides when work is done. A reasonable convention: delete after the related PR merges or the task is abandoned. If the user asks to clean up, remove the file (and the slugged variants if any).
