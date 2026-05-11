---
skill: continue
created: 2026-05-06
current-version: v2
status: published
---

# continue — History

## Problem

Agents that support persistent memory persist *cross-session learnings* — preferences, architectural facts, recurring gotchas — but do *not* persist the current conversation's transcript, in-progress work state, open questions, or today's partial decisions. Auto-loaded context is also size-bounded (Claude Code, for example, auto-loads only the first ~200 lines of `MEMORY.md`). When a session ends mid-task — context filling up, hard stop, hand-off — a fresh conversation has no way to resume without re-deriving everything.

The skill exists to bridge that gap with an explicit, conventional `continue.md` artifact written to the working directory.

## How it works (brief)

`/continue` (optionally with a slug like `/continue refactor-x`) writes or updates a `continue.md` capturing the complete state of the current task: what was attempted, what worked, what failed and why, open questions, next step, and any context the next session needs that wouldn't survive in the agent's own memory alone.

The assumption baked into the skill: the next conversation will only see the project's instruction files (e.g., `CLAUDE.md`), the auto-loaded portion of any agent memory, and `continue.md`. Anything else needed to resume goes in `continue.md`.

The markdown template for the output lives in `TEMPLATE.md` (companion file loaded on-demand) rather than embedded in `SKILL.md`, so the runtime cost of invoking the skill stays low.

See `SKILL.md` for the step-by-step capture flow.

## Iteration log

### 2026-05-06 — v1 (initial)

> **Note:** The originating conversation for this skill wasn't archived, so the rationale and design choices below are reconstructed from the current `SKILL.md` rather than from contemporaneous notes. Filling in actual history here is a TODO.

Covered: target file selection (slug → `./continue-<slug>.md`, no slug → `./continue.md`), assumption that the next conversation has only `CLAUDE.md` + `MEMORY.md` index + `continue.md`, allowed-tools list (Read/Write/Edit/Glob/Grep/Bash), `disable-model-invocation: true` (user-triggered only).

### 2026-05-10 — v2 (promoted to publishable repo, with agent-agnostic reframe + template extraction)

Reviewed via `/promote-skill continue`. Automated portability checks were all clean — the changes below were driven entirely by conversational prompt 1 (public applicability): the skill's *implementation* referenced Claude Code's auto-memory layout and `${CLAUDE_SESSION_ID}` substitution in ways that would confuse users on Cursor / Codex / other agents the `skills` CLI also installs into.

**Agent-agnostic reframes (5 touchpoints):**

- *"Why this exists"* section — generalized from "Auto-memory (`~/.claude/projects/<project>/memory/`) persists…" to "Agents that support persistent memory (Claude Code's auto-memory at `~/.claude/projects/<project>/memory/`, similar systems in other agents) persist…". The 200-line-limit reference reframed similarly with Claude Code as a named example rather than the default.
- Step 3 metadata gathering — `${CLAUDE_SESSION_ID}` is now a conditional with the instruction *"omit the line entirely if no session-id mechanism exists"*, with Claude Code's substitution as the named example. Other agents supply their own analog or skip the line.
- Template Session-line placeholder — renamed from `<CLAUDE_SESSION_ID>` to `<session-id-from-harness>`. The omit-if-unavailable guidance lives upstream in step 3.
- Orientation section example (in `TEMPLATE.md`) — was three specific filenames (`CLAUDE.md`, `docs/reference/technical-reference.md`, `docs/prd-<feature>.md`); now three category-level slots: agent project instructions (e.g., `CLAUDE.md`), top-level reference docs (`README.md`, `docs/architecture.md`), task-specific working docs (PRDs, design notes). Preserves the diversity of the original (three different levels of context) without forcing specific filenames.
- Quality bar references — *"plus the project's CLAUDE.md"* → *"plus the project's instruction files (e.g., `CLAUDE.md`)"*; *"see MEMORY.md for context"* → *"see MEMORY.md"* with the lead-in changed to *"the agent's persistent memory"*. Same substantive guidance, no Claude-Code presumption.

**Structural refactor — template extracted to companion file:**

The `## Template` section (~56 lines of markdown skeleton) was inline in `SKILL.md`. Every `/continue` invocation loaded all 141 lines of SKILL.md, even though the template is only needed during step 4 (file write). Extracted to `TEMPLATE.md` following the same companion-file pattern already used by `architect-deep` (`LANGUAGE.md`, `INTERFACE-DESIGN.md`, `DEEPENING.md`), `explore-idea` (`CONTEXT-FORMAT.md`, `ADR-FORMAT.md`), and `kickoff` (`PATHS.md`, `CLASSIFICATION.md`, `SAFETY.md`, …). `SKILL.md` now references it via `[TEMPLATE.md](TEMPLATE.md)` and Claude reads it only when needed.

Result: `SKILL.md` dropped from 141 → 83 lines; `TEMPLATE.md` holds the 56-line skeleton; on-invocation token cost falls substantially for the common case of `/continue` firing once per session.

**Knowingly left in place:**

`disable-model-invocation: true` and `allowed-tools` frontmatter remain Claude-Code-flavored — they're unknown fields to other agents but should be ignored safely. The behavior risk on non-Claude agents is mild: worst case is a slightly more eager auto-trigger based on description matching rather than explicit `/continue` invocation, and the skill itself works correctly when fired.

Source-of-truth moved from `~/dev/niftymonkey/claude/skills/continue/` to `~/dev/niftymonkey/skills/skills/continue/` + `~/dev/niftymonkey/skills/history/continue.md`. Status flipped to `published`.

## Design uncertainties

- Whether the skill should also write a sibling `MEMORY.md` entry pointing at the `continue.md`, or whether keeping the two systems disjoint is intentional.
- Whether `continue-<slug>.md` files should accumulate (one per task) or be overwritten — current behavior favors overwrite.
- Whether to add automatic invocation triggers (e.g., on PreCompact) or keep it strictly user-triggered.
- For non-Claude-Code agents that don't respect `disable-model-invocation: true`, whether the skill should add a stronger in-body discipline (*"only act on explicit user invocation"*) to deter eager phrase-matching triggers. Not addressed in v2 — observe real behavior first.
- The 200-line limit referenced in "Why this exists" is Claude Code's specific number. Other agents have different limits; pinning to "~200 lines" risks being inaccurate for Cursor / Codex / etc. Considered making the number more vague (*"size-bounded"*) but the concrete example helps Claude Code users; left as-is with the "Claude Code, for example" framing carrying the qualifier.

## Files

In `niftymonkey/skills/skills/continue/` (runtime content, installed by `npx skills add`):

- `SKILL.md` — slash command definition, step-by-step capture flow, gitignore check, metadata gathering, write/update logic, resume-prompt output, quality bar, cleanup guidance
- `TEMPLATE.md` — the markdown skeleton for `continue.md` output (loaded on-demand at step 4)

In `niftymonkey/skills/history/` (archival, not installed):

- `continue.md` — this file (renamed from `HISTORY.md` during promotion)
