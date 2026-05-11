# continue.md Template

The structure for the `continue.md` file produced by the `/continue` skill. Use it verbatim as the skeleton when creating a new continue.md, omitting any section that has no content — empty headings add noise.

---

# Continue

> **Session:** `<session-id-from-harness>`
> **Last updated:** <ISO timestamp>
> **Branch:** `<branch>` — <last commit oneline>
> **Working tree:** clean | dirty (<n> files modified, <n> untracked)

## Orientation
Where to look first if the resuming conversation needs project context that
isn't in this file. Examples: your agent's project instructions file
(e.g., `CLAUDE.md`), top-level reference docs (`README.md`,
`docs/architecture.md`), and any task-specific working docs (PRDs, design
notes). Do not duplicate their contents — just point.

## Goal
What we're working on and why. Frame it so someone with zero session context
understands the objective and the motivation in one short paragraph.

## Current Step
Exactly where we are right now. The single most specific thing the resuming
conversation needs to know to take the next action. Include file paths and
function names.

## What's Done
- Concrete, completed work. Each item should be observably true (e.g. "wrote
  tests for X in `src/foo.test.ts`, all passing", not "started on X").

## What's Next
- Remaining steps in execution order. Be specific about the first item — it
  should read like an instruction the resuming conversation can act on
  immediately.

## What We've Tried
- Approaches attempted and their outcomes. **Especially failures.** Include
  *why* each failed (exact error message, observed behavior, or the reason we
  abandoned it). This section is append-only across updates — never delete
  failed attempts. They prevent the resuming conversation from repeating them.

## Key Decisions
- Decisions made this session and the reasoning behind them. Include the
  alternatives that were considered and rejected. Future-you will want to
  know *why* a path was chosen, not just that it was.

## Research / Findings
- Discoveries that took effort: API behaviors, error message meanings,
  command outputs, doc snippets, runtime debugging results. Quote exact
  strings where relevant. This is the "things I'd hate to rediscover" bucket.

## Open Questions
- Unresolved issues, ambiguities, things waiting on the user, decisions
  deferred. Each should be phrased so the resuming conversation knows what
  would unblock it.
