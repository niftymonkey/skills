---
name: continue
description: Compact the current conversation into a handoff document for a fresh session to pick up.
allowed-tools: Read, Write, Bash
disable-model-invocation: true
argument-hint: "what the next session will focus on (optional)"
---

# /continue — session handoff

Write a handoff document that lets a brand-new conversation resume the current work without ramp-up. Save it to a unique path prefixed with `continue-` in the OS temp directory. On Unix-like systems (macOS, Linux, WSL, Git Bash) use `mktemp -t continue-XXXXXX.md`; on native Windows, generate an equivalent unique filename in `$env:TEMP` (e.g., `continue-<timestamp>-<random-suffix>.md`) and create the file via Write. Read the empty file first (the harness requires this before overwriting), then write the contents.

If the user passed arguments, treat them as a description of what the next session will focus on and tailor the doc accordingly. Without arguments, capture the active state broadly.

Do not duplicate content already captured in other artifacts (PRDs, plans, ADRs, research docs, GitHub issues, commits, diffs, prior `continue-*.md` files in `/tmp`). Reference them by path or URL. The handoff is glue, not encyclopedia.

Exception — **research findings**: if research wasn't saved to its own doc (often the case), capture it inline in the handoff with sources, exact strings, and conclusions. Redoing research is the most expensive thing for a fresh session to repeat.

Suggest skills the next session should use, if any.

## Survey the full arc, not just the recent slice

The model's attention tilts toward recency. Without push-back, the handoff captures only the last 5–10% of the session and silently loses the earlier 90%+. Two kinds of content go in the doc, and they're handled differently:

- **Arc-bearing** (what happened across the whole session) — completed work, decisions and the reasoning behind them, attempts that failed and why, discoveries that took effort. These reflect the full conversation. Failures especially: re-trying a failed approach is expensive; preserving one extra bullet costs nothing.
- **Current-focus** (where we are now, going forward) — the active goal, the exact next step, immediate open questions. Recency is *correct* here; that's the point.

Conflating the two destroys the document in both directions. Preserve both as they're intended.

Long conversations past several hundred thousand tokens are likely operating on **compacted context** — Claude Code auto-summarizes earlier exchanges without invocation. Treat the summaries as canonical for the segments they cover; do not invent detail beyond them. When content is drawn from a summary, note that briefly so the resuming conversation knows where fidelity is reduced.

## After writing

Output the file path and a one-line resume prompt for the user:

> Saved to `<path>`. To resume in a new conversation: **"Read `<path>` and let's pick up where we left off."**
