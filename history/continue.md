---
skill: continue
created: 2026-05-06
current-version: v7
status: published
---

# continue — History

## Problem

Agents that support persistent memory persist *cross-session learnings* — preferences, architectural facts, recurring gotchas — but do *not* persist the current conversation's transcript, in-progress work state, open questions, or today's partial decisions. Auto-loaded context is also size-bounded (Claude Code, for example, auto-loads only the first ~200 lines of `MEMORY.md`). When a session ends mid-task — context filling up, hard stop, hand-off — a fresh conversation has no way to resume without re-deriving everything.

The skill exists to bridge that gap with an explicit, conventional `continue.md` artifact written to the working directory.

## How it works (brief)

`/continue` (optionally with a slug like `/continue refactor-x`) writes or updates a handoff document at `./continue.md` (or `./continue-<slug>.md` for multi-task projects) in the current working directory. The doc captures arc-bearing context (completed work, decisions, failures, discoveries) and current-focus state (active goal, current step, next steps, open questions) so a brand-new conversation can resume the work without ramp-up.

The assumption baked into the skill: the next conversation will only see the project's instruction files (e.g., `CLAUDE.md`), the auto-loaded portion of any agent memory, and the handoff file. Anything else needed to resume goes in the handoff.

The skill leans on the model's prior on what handoff documents look like rather than dictating a rigid template. Arc-bearing vs current-focus discipline prevents recency bias; references to existing artifacts (PRDs, plans, ADRs, research docs, etc.) prevent the handoff from becoming a content-duplicating encyclopedia. Research that isn't externalized to its own doc is explicitly carved out as an inline exception. Storage is cwd-based so N parallel sessions across N projects each get their own handoff naturally.

See `SKILL.md` for the full guidance.

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

### 2026-05-10 — v3 (companion-files structural migration)

`TEMPLATE.md` moved from `skills/continue/TEMPLATE.md` (real file) to `companion-files/TEMPLATE.md` (canonical location) + `skills/continue/TEMPLATE.md` (symlink). No functional change for installers — `npx skills add` resolves the symlink in both symlink mode (chained symlinks to the canonical content) and copy mode (file copied through). Authoring-side benefit: edits to the template happen in one place, and future skills that also need a markdown template can reference the same canonical file rather than maintaining their own.

This is the first migration to the repo-wide `companion-files/` shared-content pattern (see `companion-files/README.md` for the index). Prepares the ground for subsequent promotions where shared content across skills can sit in one place.

### 2026-05-10 — v4 (Pocock-influenced rewrite + recency-bias pushback)

Triggered by two things: a real-world failure on a long-running conversation (near 1M tokens) where `continue.md` captured only the most recent 5–10% of the session, and Matt Pocock's release of a `handoff` skill that's structurally similar to `continue` but radically shorter. Comparing the two surfaced patterns worth adopting and made our v3 SKILL.md look over-specified.

The rewrite is near-total but preserves what differentiates `continue` from Pocock's `handoff`:

**Adopted from Pocock's `handoff` (in our own wording):**

- `mktemp -t continue-XXXXXX.md` for path uniqueness, with our `continue-` prefix. Replaces v3's slug logic, gitignore management, and overwrite/update semantics with a single shell command. Files land in the OS temp directory — never in any working tree, never an artifact that needs ignoring.
- *"Do not duplicate content already captured in other artifacts."* PRDs, plans, ADRs, research docs, GitHub issues, commits, diffs all get referenced by path or URL rather than restated. The handoff is glue, not encyclopedia. Major content-reducer.
- Forward-pointing `argument-hint`: the user states what the next session will focus on, and the doc is tailored accordingly. Removes the implicit pressure to capture *everything*.
- *"Suggest skills the next session should use, if any."* Compositional thinking; the handoff explicitly connects to the rest of the skill ecosystem.
- No mandatory template. The model adapts the doc shape to the situation rather than forcing a 5-minute blocker handoff and a multi-day arc handoff into the same mold.
- Light prose flow instead of numbered steps.

**Preserved from v3 — the `continue`-specific differentiator:**

- **Arc-bearing vs current-focus distinction.** The recency-bias problem is real and is what triggered this whole iteration. Without explicit push-back, the model writes a doc that captures only the last 5–10% of the session. The body keeps the two-paragraph guidance: arc-bearing sections (completed work, decisions, failures, discoveries) must reflect the full conversation; current-focus sections (goal, current step, next, open questions) stay tight on the active work. Conflating the two destroys the doc in both directions.
- **Compaction acknowledgment.** Long conversations are likely operating on auto-summarized context (Claude Code does this without user invocation past several hundred thousand tokens). The skill instructs the writer to treat summaries as canonical for the segments they cover, not invent detail beyond them, and note when content is drawn from a summary so the resuming conversation knows where fidelity is reduced.
- **Resume prompt output** — small but useful for the user when starting the next session.

**New in v4 (not in Pocock's, not in our v3):**

- **Research findings exception.** Research is the most expensive thing for a fresh session to redo. When it isn't saved to its own doc — often the case in practice — the handoff should capture findings inline with sources, exact strings, and conclusions. Carve-out from the "don't duplicate other artifacts" rule.
- **Cross-platform path guidance.** `mktemp` is Unix only. Native Windows (without WSL or Git Bash) doesn't have it. The skill explicitly documents the Windows fallback: generate an equivalent unique filename in `$env:TEMP` and create the file via Write. Avoids the silent `command not found` failure on Windows.

**Dropped from v3:**

- "Why this exists" backstory section (~10 lines) — modern models know what a handoff doc is; heavy framing was over-specification.
- Numbered step structure (1. Pick file, 2. Gitignore, 3. Metadata, etc.) — replaced with light prose flow.
- Slug-based multi-doc support — `mktemp` (or the Windows equivalent) handles uniqueness without per-slug logic.
- Gitignore management — files in the OS temp directory are out of any working tree.
- Explicit metadata-gathering step — the model knows to grab git context if relevant.
- Quality bar section — replaced by the arc-bearing-vs-current-focus body section, which carries the same criteria inline.
- Cleanup section — OS temp directories are ephemeral; the OS handles it.
- `TEMPLATE.md` companion file (and its `companion-files/` canonical version + symlink) — no template means the doc adapts to the situation. Updated `companion-files/README.md` to reflect the empty index.

**Result:** SKILL.md is roughly half its v3 length while addressing two failure modes Pocock's `handoff` doesn't (recency bias on long sessions; OS-portability on Windows) and one practice gap his version also has (research findings that weren't externalized).

**Follow-ups tracked for separate iterations:**

- `promote-skill` should grow an OS-portability check category — its current `CHECKS.md` audits user-specific concerns (paths, CLIs, credentials) but doesn't catch Unix-only tooling (mktemp, awk, realpath, BSD-vs-GNU flag differences, etc.) that breaks on Windows installers. Would have caught the `mktemp` issue at promote time.
- Trigger-phrase rewording: the description still implies "use when stopping" / "use when context is filling up," which is too late if compaction has already happened. Future iteration could suggest invoking `/continue` periodically (e.g., every 30–50% context growth) on long sessions.

### 2026-05-13 — v5 (autocomplete-friendly description)

Shortened the frontmatter description from ~350 characters across three concerns (what + when + trigger phrases) to ~90 characters of just "what." The long version crowded the slash-command autocomplete dropdown. Trigger phrases dropped as dead weight under `disable-model-invocation: true`; the "Use when..." guidance lives in the SKILL.md body anyway.

### 2026-05-13 — v6 (revert storage to cwd; multi-project parallel sessions)

v4's `mktemp` approach optimized for "one user, one session, no project context" and broke the user's actual workflow: N parallel Claude sessions across N projects, each `/continue` at end of day, each resumed next day. With `mktemp` + `/tmp`, all handoffs collapsed into one global namespace and "Read the most recent" couldn't disambiguate. Per-project context was lost.

Reverted the storage approach to v3's cwd-based pattern (`./continue.md`, or `./continue-<slug>.md` for multi-task projects). Re-added the gitignore management (`.git/info/exclude`) and update-in-place semantics that v3 had. Kept all v4 content improvements: arc-bearing vs current-focus discipline, don't-duplicate rule, research-findings carve-out, suggest-skills line, compaction acknowledgment, lean prose flow. `argument-hint` reverted from forward-pointing ("what the next session will focus on") to slug semantics (`"[optional-slug]"`). Description shifted to cwd-centric framing.

### 2026-05-21: v7 (cold-context triage + redaction)

Adds a three-tier triage for what an in-place update keeps, collapses, or graduates out of the handoff, so a `continue.md` carried across many resume cycles stops accreting cold context that dilutes the fresh start it exists to provide. Triggered by comparing `continue` against Pocock's `handoff` and the user observing their own handoffs still carrying context from 5-10 conversations earlier. Also adds redaction guidance and sweeps the file's pre-existing em dashes.

- [SKILL.md:39] Step 3 update rule rewritten: "preserve What We've Tried entirely" is gone. "Entirely" forced unbounded accretion. Now it triages per "Keep the handoff current", and never discards the *verdict* of a failed attempt (the narrative is disposable once its outcome is committed).
- [SKILL.md:68-78] New "Keep the handoff current" Content guidance subsection. Three tiers: Active stays inline verbatim; Superseded collapses to a one-line verdict; Revisitable graduates to an ADR, project context doc, or `docs/`. Dropping is citation-gated (a commit, PR, or ADR must capture the outcome). Promotions are proposed, not silently created.
- [SKILL.md:90-92] New "Redact secrets" subsection: strip keys, tokens, passwords, connection strings, and PII before writing. A gitignored handoff can still be screen-shared or accidentally committed.
- [SKILL.md] Swept 7 em dashes and 1 en dash to commas, colons, and parentheses. The file predated the no-em-dash rule.

## Design uncertainties

- Whether the skill should also write a sibling `MEMORY.md` entry pointing at the handoff file, or whether keeping the two systems disjoint is intentional.
- Whether to add automatic invocation triggers (e.g., on PreCompact) or keep it strictly user-triggered.
- For non-Claude-Code agents that don't respect `disable-model-invocation: true`, whether the skill should add a stronger in-body discipline (*"only act on explicit user invocation"*) to deter eager phrase-matching triggers. Observe real behavior first.
- No-template-shape: trusting the model to adapt the doc structure to the situation is the v4 bet (still in effect in v6). If outputs end up inconsistent or low-quality across runs, a lightweight schema (not a full template) may need to come back.
- Non-git projects: the gitignore step in v6 uses `git check-ignore` which fails outside a git repo. The skill should detect non-git working directories and skip the gitignore step gracefully rather than surfacing an error. Not addressed in v6; observe real behavior first.

## Files

In `niftymonkey/skills/skills/continue/` (runtime content, installed by `npx skills add`):

- `SKILL.md`: frontmatter, four-step process (file selection, gitignore management, write-or-update with header metadata, resume prompt), content guidance (arc-bearing vs current-focus, keep-the-handoff-current cold-context triage, don't-duplicate rule with research-findings exception, suggest-skills, redact-secrets), cleanup

In `niftymonkey/skills/history/` (archival, not installed):

- `continue.md` — this file (renamed from `HISTORY.md` during promotion)
