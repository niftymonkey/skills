---
name: externalize
description: Use as a standing disposition to write hard-won, expensive, non-reproducible context to the work-thread handoff file the instant it is earned during work, so it can never die in the context window or get lossily summarized by compaction. The continuous-write capture sibling of work-router. Fires when knowledge is earned, not at session end and not at a context-window threshold.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
disable-model-invocation: false
---

# externalize (continuous capture disposition)

The standing disposition that, the instant expensive, hard-won, non-reproducible context is earned during work, writes it to the work-thread handoff file. The value lands on disk while it is fresh and complete, so the context level and any compaction stop mattering: the thing worth keeping is already saved, not waiting on a boundary that may never get a clean shot at it.

## Core idea (one line)

When knowledge is earned, externalize it immediately. Never let hard-won context live only in the window, where the window will eventually lose it or flatten it to a generic gist.

## This is a disposition, not a workflow

externalize runs live in the main loop, the same way `work-router` does. It is the capture sibling of that skill: work-router routes WORK off-thread by default; externalize externalizes CONTEXT to disk by default. Both are always-on reflexes, not commands you run at a checkpoint. Do not rebuild this as something that fires at session end, at a context-window percentage, or as a detached pass. The trigger is the moment of earning, nothing else.

It is best-effort by nature: rich, faithful capture needs the model in the loop (a hook cannot judge what was hard-won), so this is a disposition the model holds, not a guarantee a tool enforces. A separate git-state backstop hook already covers the involuntary-compaction floor; externalize keeps the file current so that floor rarely matters.

## When it fires

The moment a unit of hard-won context is earned, not before and not deferred:

- a piece of research lands with its sources and conclusion,
- an architectural framing or boundary becomes clear,
- a dead end is hit and the reason it failed is understood,
- a decision is made and the rationale behind it is settled,
- a hard-won understanding of how some code actually behaves clicks into place.

Append it then. Do not batch these to the end of the session; the end is exactly when the window is most likely to have already lost the earliest and richest of them.

## What counts as expensive / hard-won (write it)

Context is worth externalizing when it was costly to arrive at and would be costly to recover:

- **Research:** findings with their sources, the exact strings that mattered, and the conclusion drawn. Redoing research is the single most expensive thing for a fresh session to repeat.
- **Architectural framing:** the boundary, the seam, the why-it-is-shaped-this-way that took effort to see.
- **Dead ends with reasons:** every approach tried and abandoned, each paired with the reason it failed. Re-walking a dead end is expensive; one extra line preventing it costs nothing.
- **Decisions with rationale:** what was chosen and the reasoning, not just the outcome.
- **Hard-won code understanding:** behavior, gotchas, and non-obvious mechanics that took real digging to establish.

## What to skip (it already has a durable home)

Do not copy context that already lives in a durable, citable artifact. Reference it by path or URL instead:

- an ADR, a `docs/` file, a PRD, a plan,
- source code, a commit, a merged PR, a diff,
- a GitHub issue.

If it is already saved somewhere durable, a pointer is enough; duplicating it just creates two copies that drift. The handoff file is glue, not an encyclopedia. (The one common exception is research that was never saved to its own doc: it has no durable home yet, so capture it inline per above.)

## The target file

Write to the live work-thread handoff for the current thread:

- code / engineering work -> `./continue.md` (or its slugged variant),
- personal / non-engineering work -> `./carryover.md` (or its slugged variant),
- otherwise, whatever handoff file the thread is already keeping.

This disposition is domain-general: it is the WRITE side for whichever handoff the thread uses. If a handoff file does not exist yet and hard-won context is accruing, start one. Append to the existing structure; do not wholesale rewrite it (rewriting is curation, which is not this skill's job, see the boundary below).

## Capture quality (the spine the curate-side siblings rely on)

This is the discipline both curate-side siblings rely on, owned here because faithful capture is externalize's whole thesis. The window will lose or lossily summarize hard-won context; the job is to get it to disk faithfully before that happens.

- **Capture faithfully.** Record what was actually found, decided, or concluded. Preserve exact strings, terms, and the reasoning, not a flattened paraphrase. Mark anything inferred as an inference.
- **Fight recency.** Attention tilts toward the most recent exchanges, so the earliest hard-won context is the most at risk and the most likely to be silently dropped. When externalizing, reach back for what was earned earlier in the thread, not only what just happened.
- **Keep dead ends with their reasons.** An abandoned approach without its reason is an invitation to re-try it. Always pair the dead end with why it failed.
- **Note compacted-summary provenance.** Past a few hundred thousand tokens the thread is likely running on compacted context: the harness auto-summarized earlier exchanges. Treat those summaries as canonical for the spans they cover, do not invent detail beyond them, and note briefly when a captured item is drawn from a summary rather than the live exchange.
- **Redact secrets.** A handoff lands in the working directory and can be opened, screen-shared, or accidentally committed. Strip API keys, tokens, passwords, connection strings, and PII; reference a secret-bearing file by path rather than inlining its contents.

## Boundary: this is the WRITE side only

externalize captures. It does not curate. It never prunes, never collapses cold context, never re-synthesizes, never re-compresses, never reorganizes the handoff into a resume-shaped document. Those are the on-demand curate-side jobs and they live elsewhere:

- `continue` owns PRUNE-curation for code handoffs (triage, collapse-against-citation, graduation to ADRs/docs, the git-state header, resume-framing).
- a personal-handoff curator (such as a `carryover` companion skill, if installed) owns PRESERVE-curation for personal handoffs: preserve rather than prune, since personal context has no external home to graduate to.

externalize keeps the file full and current; the curate-side skills decide, when deliberately starting a clean session, what to do with what externalize put there. Keep this skill to the capture half. If you find yourself triaging, pruning, or restructuring an existing handoff, you have crossed into curate-side territory (`continue` for code, a personal-handoff curator for personal threads), stop and use the right one.
