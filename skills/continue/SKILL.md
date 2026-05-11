---
name: continue
description: Capture full session state to a continue.md file so a new conversation can pick up exactly where this one ended. Use when the context window is filling up, before stopping work mid-task, or any time the user wants to hand off state to a fresh conversation. Triggers on /continue or phrases like "save state", "write a continue doc", "preserve context for next session".
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
disable-model-invocation: true
argument-hint: "[optional-slug]"
---

# /continue — session handoff

Write or update a `continue.md` in the current working directory that captures the *complete* state of the current task. The file's job is to make a brand-new conversation feel like a continuation of this one — no fresh-start ramp-up, no rediscovery.

## Why this exists (and what it isn't)

Agents that support persistent memory (Claude Code's auto-memory at `~/.claude/projects/<project>/memory/`, similar systems in other agents) persist *cross-session learnings* — preferences, architectural facts, recurring gotchas. These mechanisms typically do **not** persist:

- The current conversation's transcript
- In-progress work state ("we just tried X and it failed because Y")
- Open questions from this session
- Today's research findings, error messages, or partial decisions

Auto-loaded context is also size-bounded — Claude Code, for example, auto-loads only the first ~200 lines of `MEMORY.md`; other agents have analogous limits. **Assume the next conversation will see your project's instruction files (e.g., `CLAUDE.md` for Claude Code), the auto-loaded portion of any agent memory, and `continue.md` — nothing else from this session.** If a fact is needed to resume, it goes in this file.

## When invoked

The user typed `/continue` (optionally with a slug like `/continue refactor-x`). Run these steps:

### 1. Pick the target file

- No slug → `./continue.md`
- Slug `foo` → `./continue-foo.md`

The slug exists so the user can run multiple conversations in the same repo without one clobbering the other (e.g. `/continue auth` in one session, `/continue ui` in another).

### 2. Check gitignore status

A `continue.md` is personal session state — it should not be committed, and it should not appear in teammates' working trees if they pull. Default to `.git/info/exclude` (local, per-clone) over `.gitignore` (shared, committed).

- Check whether the target filename is already ignored. Quick check: `git check-ignore -q <filename> && echo ignored || echo not-ignored`.
- If not ignored:
  - If `.git/info/exclude` exists, append the pattern there and tell the user.
  - If only `.gitignore` is in use, ask the user which file they prefer before modifying.
  - Recommended pattern to add: `continue.md` and `continue-*.md` (covers both default and slugged forms).
- If already ignored, say nothing — no need to mention it.

Only do this check on the first write. If the file already exists and is being updated, skip it.

### 3. Gather orienting metadata

Run these in parallel and capture the output for the header:

- Session identifier from the harness if available (e.g., `${CLAUDE_SESSION_ID}` for Claude Code; check your agent's docs for an analogous substitution). Omit the line entirely if no session-id mechanism exists.
- Current timestamp
- `git rev-parse --abbrev-ref HEAD` (branch)
- `git log -1 --oneline` (last commit)
- `git status --short` (dirty/clean snapshot — keep concise, summarize if long)

### 4. Write or update the file

If the file does not exist, create it from the structure in [TEMPLATE.md](TEMPLATE.md). Read TEMPLATE.md once at this step and use it verbatim as the skeleton.

If it already exists, **update in place — never wholesale overwrite**. Preserve the entire "What We've Tried" section (failed attempts are the most valuable part — the next conversation must not re-try them). Move stale "What's Next" items to "What's Done" as they're completed. Add new findings to "Research / Findings" rather than replacing.

When updating, refresh the header metadata block (timestamp, last commit, dirty/clean) so the resuming session can tell how stale the file is.

### 5. Tell the user what to do next

After writing, output a one-line resume prompt the user can paste into a new conversation, e.g.:

> Saved to `./continue.md`. To resume in a new conversation: **"Read ./continue.md and let's pick up where we left off."**

## Quality bar

The file is good if a brand-new conversation, given only this file plus the project's instruction files (e.g., `CLAUDE.md`), could resume the work without asking the user "where were we?" or "what have we already tried?".

- **Specific over summarized.** Exact error messages > "the build was failing." Exact file paths and line numbers > "somewhere in the auth module." Verbatim command output > "it didn't work."
- **Self-contained.** Don't lean on the agent's persistent memory ("see MEMORY.md") for anything that's load-bearing for the resume. Memory is for cross-session facts; this file is for the *current task's* state.
- **Failures preserved.** "What We've Tried" is the highest-value section. The cost of re-trying a failed approach is high; the cost of writing one extra bullet is nothing.
- **No filler.** Skip sections with nothing in them. Don't write "TBD" placeholders.

## Cleanup

Don't delete `continue.md` automatically. The user decides when work is done. A reasonable convention: delete after the related PR merges or the task is abandoned. If the user asks to clean up, remove the file (and the slugged variants if any).
