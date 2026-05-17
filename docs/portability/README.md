# Cross-harness portability for skills

This directory documents how different agent harnesses (Claude Code, Gemini CLI, Codex, etc.) implement the affordances that skills depend on, and what that means for authoring skills that survive across them.

Skills authored here in `niftymonkey/skills` install via the [`vercel-labs/skills` CLI](https://github.com/vercel-labs/skills), which targets multiple harnesses via the [agentskills.io](https://agentskills.io) standard. The SKILL.md content plus bundled `scripts/`/`references/`/`assets/` ports cleanly. The *runtime affordances* a skill relies on — sub-agents, hooks, memory files, permission flows — are vendor-specific. This directory tracks those deltas.

## Durable vs volatile content — the contract

This directory has two kinds of file. Treat them differently:

- **This README — durable.** Affordance categories, facets, and portable-skill rules survive harness rev's. Update only when a genuinely new category appears (a harness ships an affordance no existing study had to describe), or when an existing rule is invalidated by cross-harness evidence.
- **`<harness>.md` files — volatile snapshots.** Each is a point-in-time description of one harness. Trust them only against their `Verified:` date. Re-verify before relying on a specific fact.
- **`research-prompt.md` — durable recipe.** A single general prompt that researches *any* harness when prepended with a short preamble (harness name, canonical URL, companion-file name, optional Known-volatile bullets from the existing study). Re-running it is how a study gets refreshed or how a new harness study gets authored.

The split keeps the framework rarely-touched and the studies refreshable on a known cadence. No changelogs — each study is a clean snapshot; git history is the change record.

> **Note on Claude.** Claude Code is the implicit baseline this framework anchors against, not a peer harness. Each `<harness>.md` includes a "Cross-harness portability notes" section tagged with a *Comparison anchor* date. The Claude reference state is captured once below in [Claude Code reference state](#claude-code-reference-state); when Claude rev's, that anchor updates and dated comparisons in studies re-anchor against it. This keeps the operational value of "what's different on this harness vs your daily driver" without parallel maintenance of a full Claude study.

## How to read this directory

- **`README.md`** (this file) — the framework: affordance categories, facets each category must describe, and rules for writing portable skills.
- **`<harness>.md`** — one file per harness studied. Each is structured against the same facets, so studies diff and compare side-by-side.
- **`research-prompt.md`** — single general research recipe used to author or refresh any harness study. Operator supplies a short preamble at spawn time with harness name + URL + companion-file name + optional Known-volatile cues from the existing study.

Current studies:

- [`gemini.md`](./gemini.md) — Gemini CLI (`google-gemini/gemini-cli`).
- [`codex.md`](./codex.md) — OpenAI Codex CLI (`openai/codex`).
- [`copilot-cli.md`](./copilot-cli.md): GitHub Copilot CLI (`github/copilot-cli`, npm `@github/copilot`).
- *Planned: `cursor.md`, others as adopted.*

(Claude Code is not a study — it's the implicit baseline. See [Claude Code reference state](#claude-code-reference-state) below.)

---

## The framework — affordance categories and facets

Each harness study fills in the facets below, one section per category. If a facet doesn't apply (the affordance doesn't exist), record `(none)` rather than omitting — explicit absence is more informative than missing data.

### 1. Skill loading & discovery

How the harness finds skills and decides when to activate them.

- **Workspace skill path** — where in a project skills live.
- **User-scope skill path** — where global skills live.
- **Auto-activation** — does the harness pick skills by `description:` match, or only on explicit reference?
- **Manifest standard** — agentskills.io? Proprietary? Hybrid?
- **Frontmatter honored** — which required and optional fields are recognized (`name`, `description`, `allowed-tools`, `disable-model-invocation`, etc.).
- **Discovery commands** — slash commands or other introspection (`/skills`, etc.).

### 2. Memory / context files

Always-loaded context that survives across all conversations.

- **Default filename** — `CLAUDE.md`, `GEMINI.md`, `AGENTS.md`, etc.
- **Configurable filename** — yes/no, and via what mechanism.
- **Scope** — global, project, hierarchical.
- **Load order** — which file wins when multiple exist; how they're concatenated/merged.
- **Import syntax** — e.g., `@./other.md`, `<include>`, etc.
- **Inspection / update commands** — slash commands or other affordances.

### 3. Tool ecosystem

Built-in tools the model can call.

- **Built-in tools** — table, categorized (file system, shell, web, interaction, tasking, skills, etc.).
- **Tool naming convention** — capture exact names; these diverge across harnesses and are a major portability gotcha.
- **Default enablement** — which tools are on by default vs require opt-in.
- **Approval modes** — `default`, `auto`, `yolo`, `plan`, etc.
- **Sandbox** — platform-specific sandboxing affordances.

### 4. Sub-agents / delegation

Whether the harness can spawn isolated sub-agents.

- **Available** — yes / no / partial; since which version.
- **Definition format** — file location and format.
- **Parallel execution** — concurrency support and caveats.
- **Context isolation** — fresh context per agent?
- **Invocation** — explicit `@name`, auto-routing, or both.
- **Background execution** — async support.

### 5. Hooks / deterministic enforcement

Runtime hooks that fire on events and can deny or modify behavior. This is the most divergent category across harnesses — treat hook portability as the hardest portability question.

- **Available since** — version.
- **Configuration location** — settings file path and shape.
- **Event types** — full list, grouped by lifecycle (tool, agent, session, etc.).
- **Matcher syntax** — string / regex / content predicate.
- **Hook input JSON shape** — fields available on stdin.
- **Hook output JSON shape** — decision values, side-effect fields.
- **Exit-code semantics** — `0`/`2` and any others.

### 6. MCP support

Model Context Protocol client.

- **Client available** — yes / no / partial.
- **Transports** — stdio / SSE / streamable HTTP.
- **Configuration** — file location and shape.
- **Per-server settings** — trust, tool allow/deny, env, timeout.
- **Discovery commands** — slash commands for inspection or auth.

### 7. Slash commands / explicit invocation

Manually-triggered procedures.

- **File format** — TOML / JSON / Markdown.
- **Locations** — global, project, scoping rules.
- **Argument injection** — e.g., `{{args}}`.
- **Shell injection** — inline shell expansion inside a command file.
- **File injection** — file-content interpolation inside a command file.
- **Reloadability** — in-session reload command.

### 8. Permission model

How the harness decides whether to run a tool / continue.

- **Modes** — list available, with default highlighted.
- **Tool granularity** — per-tool overrides; per-pattern allow/deny.
- **Elevation** — can the model request a mode change mid-run?
- **Sandbox integration** — how the permission model relates to platform sandboxing.

### 9. Default model & coding behavior

What model runs under the hood and how it behaves over time.

- **Default model identifier** — and routing logic if applicable.
- **Context window** — token limit.
- **Max output tokens** — per turn.
- **Known weaknesses** — primary-source citations preferred (upstream GitHub issues, official docs).
- **Pricing** — input / output / cache, with verification status.
- **Enterprise lag** — what version users actually get inside large orgs.

---

## Rules for authoring a portable skill

Distilled from the harness studies. These survive most harness rev's; update when a new study surfaces evidence that invalidates one.

1. **Author the SKILL.md and bundled scripts as portable; treat hooks as vendor-specific.** The agentskills.io spec covers skill file structure and frontmatter — that's the portable surface. Hook wiring is **not** standardized and must be authored per-harness. If a skill ships with hooks, ship a per-harness settings snippet alongside the scripts.

2. **Don't extract instructions into hooks unless every target harness has the equivalent event.** The hook-event lists differ across harnesses. If a Claude `FileChanged` hook has no Gemini equivalent (or vice versa), extracting it from a skill into a single-harness hook silently strips the discipline elsewhere. Keep the instruction in the skill, or pair the hook with a fallback memory-file fragment.

3. **Don't hard-code tool names in skill instructions.** Tool names diverge across harnesses (e.g., `Edit` vs `replace`, `Bash` vs `run_shell_command`). When writing a skill instruction that references a tool, prefer the *action* ("edit the file", "run the command") over the harness-specific tool name. If the harness-specific name is unavoidable, `/promote-skill` should flag it.

4. **Don't assume sub-agent fan-out is available.** Some harnesses don't have sub-agents at all; others have them with different parallelism limits. If a skill's procedure depends on sub-agents for quality, ship a single-agent fallback path.

5. **Don't assume memory-file filename or load order.** Filenames (`CLAUDE.md`, `GEMINI.md`, `AGENTS.md`) and most load-order behaviors are harness-specific and often user-configurable. Reference *"the always-loaded memory file"* generically in skill content; let the install procedure wire it up.

6. **Don't assume `ask`-style permission UX.** Some harnesses can prompt the user mid-run for confirmation; some can only `allow` or `deny`. If a skill's behavior relies on *"ask the user to confirm before X"*, redesign for hard-deny with a clear reason. That's the more conservative behavior and ports universally.

7. **Don't assume matcher granularity in hooks.** Some harnesses support content predicates in matchers (e.g., filter on `tool_input.file_path` matching `*.ts` at config-load time); others don't. Filter inside the shell script body — scripts are universal, matcher syntax isn't.

8. **For hooks that don't fully port, ship a memory-file fallback fragment.** If a hook only works on one harness, bundle a paragraph for the *other* harness's memory file that re-states the discipline as an instruction. The bundle installer concatenates fragments or imports them via the harness's `@./`-style mechanism.

9. **Beware long-session behaviors specific to the underlying model.** Model behaviors (context rot, instruction drift over many turns) are surface-area for the harness study to document, not for the skill to silently depend on. Skills that need multi-turn adherence should ship explicit checkpoint/compress instructions or delegate subtasks to sub-agents when available.

---

## Freshness model — how studies stay trustworthy

Every harness study carries three layers of freshness signal:

1. **Document-level metadata** — `Verified:` and `Harness version studied:` in the header. If the document hasn't been re-verified within ~6 months, treat as stale.
2. **Per-section verification** — sections can carry their own `verified: <date>` when only part of the harness changes. Update sections individually rather than retouching the whole doc.
3. **Per-fact citations** — high-stakes facts (event names, tool names, model identifiers, JSON shapes) link to primary sources inline. Re-verification is mechanical: click the link, check the source, refresh the date if it still matches.

**Triggers for re-verification:**

- **Calendar** — every ~6 months at minimum.
- **Event** — when the harness ships a major version, or removes/renames an affordance.
- **Use** — before building or shipping anything that relies on a specific affordance.

**Re-verification procedure:**

1. Read the existing `<harness>.md`'s "Known volatile" section. Those are your re-verification anchors.
2. Construct a preamble per [`research-prompt.md`](./research-prompt.md)'s "How to run" instructions — harness name, canonical URL, companion-file name, plus the Known-volatile bullets from step 1.
3. Spawn a general-purpose agent with `<preamble> + <research-prompt.md body>`.
4. Diff agent output against current `<harness>.md`; update what changed.
5. Refresh the `Verified:` header and the "Known volatile" / "Known stable" callouts based on what actually changed.

---

## Claude Code reference state

*Comparison anchor used by `<harness>.md` studies' "Cross-harness portability notes" sections.*

**Verified:** 2026-05-14
**Reference version:** Claude Code (current release line; default model: Opus 4.7 (1M context))

Brief enumeration of Claude Code's state-of-truth that other harness studies anchor their comparisons against. When Claude rev's, update only this section; dated `Comparison anchor: <date>` markers in other studies become re-anchored automatically.

### Tool names

CamelCase verb-style. Core set: `Edit`, `Write`, `Read`, `Bash`, `Glob`, `Grep`, `WebFetch`, `WebSearch`, `NotebookEdit`. Sub-agent invocation via `Agent` (subagent dispatcher).

### Memory file

`CLAUDE.md` — hard-coded filename (not configurable at the harness level). Hierarchical: global `~/.claude/CLAUDE.md` + project-tree files. Supports `@./` imports.

### Hook events

`PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `SubagentStop`, `SessionStart`, `SessionEnd`, `Notification`, `PreCompact`, `PostCompact`, `PostToolBatch`, `FileChanged`, `ConfigChange`, `CwdChanged`, `WorktreeCreate`, `WorktreeRemove`, `Setup`, `TaskCreated`, `TaskCompleted`, `TeammateIdle`, `InstructionsLoaded`, `Elicitation`, `ElicitationResult`, `StopFailure`, `PermissionRequest`, `PermissionDenied`.

### Hook matcher syntax

String matcher on `tool_name` — supports `*` (all), `|`-list, exact, or regex (auto-detected by special chars). Plus per-handler `if:` **content predicate** at config-load time (e.g., `if: "Edit(*.ts)"` filters by `tool_input` without invoking the script). Multiple matcher groups per event are allowed; matching hooks run in parallel.

### Hook decision values

Four: `allow`, `deny`, `ask`, `defer` — set via `hookSpecificOutput.permissionDecision` for `PreToolUse`. Universal fields `continue`, `systemMessage`, `suppressOutput` also supported. Exit codes: `0` = parse stdout, `2` = block.

### Sub-agents

`Agent` tool with `subagent_type` parameter. Parallel execution via multiple `Agent` tool calls in one assistant message. Isolated context per sub-agent. Foreground or background execution.

### Permission model

Modes: `default`, `acceptEdits`, `bypassPermissions`, `plan`. Per-tool / per-pattern `allow` / `deny` / `ask` rules in `settings.json` `permissions` block. Model can request elevation mid-run when blocked.

### MCP

Full MCP client (stdio, SSE, streamable HTTP). Configured in `.mcp.json` (project) or `settings.json` (global). Per-server `command`/`url`, `env`, `disabled`.

### Slash commands

Custom commands as Markdown files in `.claude/commands/` (project) or `~/.claude/commands/` (user). Subdirectories form `:` namespaces. Supports `$ARGUMENTS`, `!` shell injection, `@` file injection.

### Context window & pricing

Opus 4.7 (1M context): 1,048,576 tokens. Pricing (Opus 4.x tier, reported): ~$15 / 1M input, ~$75 / 1M output, ~$1.50 / 1M cached read.

---

## Future: feeding `/promote-skill`

`/promote-skill` already strips Claude-specific references during incubation → publication. Once these portability notes are mature, the promotion review can check candidate skills against this directory and *block promotion* on:

- References to a specific memory filename (`claude.md`, `gemini.md`) not abstracted to "the memory file."
- Hook instructions that depend on single-harness events without a documented fallback.
- Tool names hard-coded into procedure text.
- Sub-agent fan-out without a single-agent fallback path.
- Permission UX that depends on `ask` semantics.

Implementation sketch: a checklist in the `/promote-skill` pipeline that reads from this directory's harness studies and verifies the candidate skill won't silently degrade on any documented harness. Until then, this directory is a manual review reference.

---

## Adding or refreshing a harness study

**Adding a new harness:**

1. Construct a preamble for [`research-prompt.md`](./research-prompt.md) per its "How to run" instructions — harness name, canonical implementation URL, companion-file name (the `<harness>.md` you'll create), distribution/package identifier.
2. Spawn a general-purpose agent with `<preamble> + <research-prompt.md body>`. ~2–3 minutes.
3. Copy `gemini.md` or `codex.md` as the structural template for `<harness>.md`.
4. Rewrite each section of the new harness study against the agent's output, filling in every facet (use `(none)` for absent affordances). Cite primary sources inline.
5. Populate the "Known volatile" / "Known stable" sections based on observed change-rate cues during research.
6. Add a "Cross-harness portability notes" section anchored to today's date, comparing against the [Claude Code reference state](#claude-code-reference-state) above.
7. Add an entry to the "Current studies" list above.
8. If the harness surfaces a *new* affordance category the framework doesn't capture, add the category here in the README and backfill the existing studies (usually a short paragraph each).

**Refreshing an existing study:** see the "Re-verification procedure" in the Freshness model section above.
