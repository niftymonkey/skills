# OpenCode - harness study

| | |
|---|---|
| **Harness** | [`anomalyco/opencode`](https://github.com/anomalyco/opencode) (MIT; TypeScript monorepo; distributed as `opencode-ai` on npm, binary `opencode`) |
| **Verified** | 2026-05-17 |
| **Harness version studied** | [`v1.15.4`](https://github.com/anomalyco/opencode/releases/tag/v1.15.4) (2026-05-17); plugin-based hook system; subagents Stable; no single hard-coded default model |
| **Framework** | [`README.md`](./README.md) |
| **Refresh recipe** | [`research-prompt.md`](./research-prompt.md) (general; supply preamble with harness name, URL, companion file `opencode.md`) |

A snapshot of OpenCode's affordances, structured against the framework's nine categories. Treat facts as authoritative only against the `Verified:` date; primary-source links inline are how to re-verify.

## Which OpenCode this study covers

**The name "OpenCode" is genuinely tangled, so this study pins one project.** This document covers the terminal coding agent currently distributed under the package name `opencode-ai` and the binary `opencode`, served from [opencode.ai](https://opencode.ai), with its canonical source repository at **[`github.com/anomalyco/opencode`](https://github.com/anomalyco/opencode)**. The repository's GitHub org was renamed from `sst` to `anomalyco`; `github.com/sst/opencode` still redirects there and the Go module path remains `github.com/sst/opencode`. This is the project built by the team behind SST and terminal.shop.

The disambiguation matters because of provenance. The original "OpenCode" was created by Kujtim Hoxha at `github.com/opencode-ai/opencode`. That project moved to Charm and is now maintained as **Crush** ([`github.com/charmbracelet/crush`](https://github.com/charmbracelet/crush)); the `opencode-ai/opencode` repo was archived. `anomalyco/opencode` is a separate, from-scratch rewrite that forked off that earlier work and became the more actively-developed project. Charm's maintainers describe the relationship directly: "Crush is the continuation of the original OpenCode project ... and `sst/opencode` is a fork of that repo" ([charmbracelet/crush #1097](https://github.com/charmbracelet/crush/issues/1097)). **This study is about `anomalyco/opencode`, not Crush.** When a skill author says "OpenCode" today, the package on npm and the docs at opencode.ai are this project; a skill targeting Crush would need a separate study.

**Headline:** OpenCode is the *most* structurally divergent harness studied so far. It deliberately rejects the shell-script hook model that Claude Code and Codex share: there is no `settings.json` `hooks` block and no JSON-on-stdin contract. Instead, "hooks" are TypeScript/JavaScript plugin functions loaded in-process, subscribing to event keys like `tool.execute.before`. Plugins block by *throwing an exception*, not by emitting a decision JSON. OpenCode also has no fixed default model and no built-in approval-mode vocabulary in the Claude/Codex sense: model choice is bring-your-own-provider, and "modes" are just primary agents (`Build`, `Plan`). Skill content ports cleanly; every deterministic-enforcement assumption does not.

---

## 1. Skill loading & discovery

- **Workspace skill path:** `.opencode/skills/<name>/SKILL.md`, walked from CWD up to the git worktree root. Also reads `.claude/skills/<name>/SKILL.md` and `.agents/skills/<name>/SKILL.md`. Source: [Skills docs](https://opencode.ai/docs/skills/).
- **User-scope skill path:** `~/.config/opencode/skills/<name>/SKILL.md`; also `~/.claude/skills/<name>/SKILL.md` and `~/.agents/skills/<name>/SKILL.md`. Admin scope: `(none)` documented.
- **Auto-activation:** **explicit, not description-matched.** Skills are surfaced to the model as a single built-in `skill` tool; the model loads a skill on demand by calling `skill({ name: "git-release" })`. Unlike Claude and Gemini, OpenCode does not auto-select a skill from `description:` similarity at prompt time. The `description` informs the model's *choice* of which `skill` call to make, but the activation step is an explicit tool call.
- **Manifest standard:** [agentskills.io](https://agentskills.io) `SKILL.md` layout: `SKILL.md` plus optional bundled `scripts/`/`references/`/`assets/`. The `.agents/skills/` and `.claude/skills/` read paths make the `vercel-labs/skills` CLI install cleanly.
- **Frontmatter honored:** `name` (required, 1-64 chars, pattern `^[a-z0-9]+(-[a-z0-9]+)*$`), `description` (required, 1-1024 chars), `license` (optional), `compatibility` (optional), `metadata` (optional string-to-string map). `allowed-tools` is **not** a documented `SKILL.md` field; tool restriction is done at the agent level (section 4), not in skill frontmatter.
- **Scope precedence:** project-level beats user-level when the same skill `name` appears in both. `.opencode/` is checked before the `.claude/`/`.agents/` compatibility paths.
- **Discovery commands:** the `skill` tool itself is the catalog surface. No dedicated `/skills` slash command is documented.

## 2. Memory / context files

- **Default filename:** `AGENTS.md` (project root). Source: [Rules docs](https://opencode.ai/docs/rules/). `CLAUDE.md` is read as a fallback for Claude Code compatibility.
- **Override filename:** `(none)`. There is no per-directory `.override.md` mechanism.
- **Configurable filename:** no fixed-name override, but the `instructions` field in `opencode.json` accepts additional file paths and glob patterns (e.g., `"packages/*/AGENTS.md"`), so arbitrary extra instruction files can be pulled in.
- **Scope and load order:** on startup OpenCode loads, in order: (1) local files by traversing up from CWD, (2) global `~/.config/opencode/AGENTS.md`, (3) Claude-compat `~/.claude/CLAUDE.md`. Project `CLAUDE.md` and global `~/.claude/CLAUDE.md` are used only when their OpenCode counterparts are absent.
- **Size cap:** `(none)` documented.
- **Import syntax:** `(none)` as a first-class memory-file directive. Composition is via the `opencode.json` `instructions` glob list. The `@file` syntax exists for *slash commands* (section 7), not for `AGENTS.md` includes; an `AGENTS.md` can mention a path in prose and instruct the agent to read it, but that is model-driven, not a literal include.
- **Inspection / update commands:** `/init` scans the repo and creates or updates `AGENTS.md` with project guidance. No `/memory`-style live inspection command documented.

## 3. Tool ecosystem

Built-in tools (all enabled by default; gated by the permission system in section 8). Source: [Tools docs](https://opencode.ai/docs/tools/), [permissions docs](https://opencode.ai/docs/permissions/).

| Category | Tool names |
|---|---|
| File system | `read`, `write`, `edit`, `apply_patch`, `glob`, `grep`, `list` |
| Shell | `bash` |
| Code intelligence | `lsp` |
| Web | `webfetch`, `websearch` |
| Interaction | `question` |
| Tasking | `todowrite`, `todoread`, `task` |
| Skills | `skill` |
| MCP | `<server>_<tool>` (MCP tools exposed alongside built-ins; permission wildcard `mcp_*`) |

**Naming convention:** lowercase, underscore-joined for multi-word names (`apply_patch`, `todowrite`). This diverges from Claude Code's CamelCase verbs (`Edit`, `Bash`, `Read`) and from Codex's hybrid hook-name scheme; see README rule 3. OpenCode has *distinct* `read`, `write`, `edit`, and `apply_patch` tools (closer to Claude's separation than to Codex's single `apply_patch`).

**Default enablement:** "By default, all tools are enabled and don't need permission to run" ([tools docs](https://opencode.ai/docs/tools/)); `todowrite` is disabled for subagents by default. The permission system (section 8) is how tools are gated, not a per-tool feature flag.

**Approval modes:** `(none)` as a Claude/Gemini-style mode vocabulary. OpenCode has no `default`/`auto`/`yolo`/`plan` mode enum. The closest equivalents are the **permission system** (`allow`/`ask`/`deny` per tool, section 8) and **primary agents** (`Build` vs `Plan`, section 4). "Plan mode" is the `Plan` agent, which sets `edit` and `bash` to `ask`.

**Sandbox:** `(none)`. OpenCode ships no OS-level sandbox (no Seatbelt, no `bwrap`/`seccomp`, no Windows sandbox). Containment is by permission rules only; the docs position OpenCode itself as the trust boundary. Protected paths: `read` permission denies `.env` files by default.

## 4. Sub-agents / delegation

- **Available:** yes, Stable. Two agent kinds: **primary agents** (the assistant you talk to) and **subagents** (delegated specialists). Source: [Agents docs](https://opencode.ai/docs/agents/).
- **Definition format:** Markdown files in `~/.config/opencode/agents/` (global) or `.opencode/agents/` (project), where the filename becomes the agent name; or a JSON `agent` block in `opencode.json`.
- **Required fields:** `description`. **Optional:** `mode` (`primary` / `subagent` / `all`, default `all`), `model`, `prompt` (system-prompt file reference), `permission` (per-agent tool gating, section 8), `temperature`, `top_p`, `steps` (max agentic iterations), `color`, `hidden` (omit from `@` autocomplete).
- **Parallel execution:** subagents run as child sessions; the parent navigates parent/child sessions via keybinds (`session_child_first`, `session_child_cycle`, `session_parent`). Concurrency caps such as `max_threads`/`max_depth` are `(none)` documented; OpenCode does not surface explicit fan-out limits the way Codex does.
- **Context isolation:** yes. Each subagent runs as its own child session with its own context window and curated tool/MCP set.
- **Invocation:** both. Explicit `@agent-name` mention in the composer, and automatic (a primary agent invokes a subagent based on its `description`). Programmatic invocation is via the `task` tool.
- **Built-in agents:** primary: `Build` (all tools enabled; default) and `Plan` (`edit` and `bash` set to `ask`). Subagents: `General` (full access except `todo`), `Explore` (read-only codebase), `Scout` (read-only external docs/dependencies). Hidden system agents: `Compaction`, `Title`, `Summary`.
- **Background / async:** subagents are foreground child sessions. Note [#24391](https://github.com/anomalyco/opencode/issues/24391): OpenCode can hang if exited while waiting on subagents.

## 5. Hooks / deterministic enforcement

**Headline:** OpenCode's enforcement model is *categorically different* from Claude Code and Codex. There is **no shell-script hook system**: no `settings.json` `hooks` block, no event-name matchers, no JSON-on-stdin / JSON-on-stdout contract, no `0`/`2` exit-code semantics. "Hooks" are **TypeScript/JavaScript plugin functions** running in-process. This is the single hardest portability fact in this study.

- **Available since:** plugins are a Stable, documented part of OpenCode. Source: [Plugins docs](https://opencode.ai/docs/plugins/), plugin type definitions [`packages/plugin/src/index.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/plugin/src/index.ts).
- **Configuration location:** plugins are JS/TS files in `.opencode/plugins/` (project) or `~/.config/opencode/plugins/` (global), auto-loaded at startup; or npm packages listed in the `plugin` array of `opencode.json`. Load order: global config, project config, global plugin dir, project plugin dir.
- **Plugin shape:** a module exports a function `(input, options) => Promise<Hooks>`. The `input` context object carries `client` (an OpenCode SDK client), `project`, `directory`, `worktree`, `serverUrl`, and `$` (Bun shell API). The returned `Hooks` object maps hook keys to handler functions.

### Hook keys (the `Hooks` interface)

These are the canonical hook keys from [`packages/plugin/src/index.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/plugin/src/index.ts). Each is a function `(input, output) => Promise<void>`; the handler mutates `output` in place to influence behavior.

| Hook key | Trigger | Influence path |
|---|---|---|
| `event` | Catch-all for every emitted event (see event list below) | Observe only |
| `config` | Once on init, with the merged config | Observe / mutate config |
| `auth` | Provider authentication flow registration | Register OAuth / API auth methods |
| `provider` | Provider/model registration | Add models for a provider |
| `tool` | Register a custom tool | Adds a tool callable by the model |
| `tool.definition` | Before a tool definition is sent to the LLM | Mutate `description` / `parameters` |
| `tool.execute.before` | Before a tool runs | Mutate `output.args`; **throw to block** |
| `tool.execute.after` | After a tool produces output | Mutate `output.title` / `output.output` / `output.metadata` |
| `chat.message` | A new user message is received | Observe `message` / `parts` |
| `chat.params` | Before LLM request params are finalized | Mutate `temperature`, `topP`, `topK`, `maxOutputTokens`, `options` |
| `chat.headers` | Before the LLM HTTP request | Mutate request `headers` |
| `command.execute.before` | Before a slash command runs | Mutate `output.parts` |
| `permission.ask` | A permission prompt is about to surface | Set `output.status` to `ask` / `deny` / `allow` |
| `shell.env` | Building env for any shell execution | Mutate `output.env` |
| `experimental.chat.messages.transform` | Before messages are sent to the model | Rewrite the `messages` array |
| `experimental.chat.system.transform` | Before the system prompt is assembled | Rewrite the `system` string array |
| `experimental.session.compacting` | Before compaction summary generation | Append `output.context[]` or replace `output.prompt` |
| `experimental.compaction.autocontinue` | After compaction, before synthetic continue turn | Set `output.enabled` false to skip auto-continue |
| `experimental.text.complete` | A streamed text part completes | Mutate `output.text` |

### Event types (delivered to the `event` hook)

The `event` hook receives `{ event: Event }` for every event below. These are *observation* channels, not decision channels; they cannot block. Source: [Plugins docs](https://opencode.ai/docs/plugins/).

`command.executed`; `file.edited`, `file.watcher.updated`; `installation.updated`; `lsp.client.diagnostics`, `lsp.updated`; `message.part.removed`, `message.part.updated`, `message.removed`, `message.updated`; `permission.asked`, `permission.replied`; `server.connected`; `session.created`, `session.compacted`, `session.deleted`, `session.diff`, `session.error`, `session.idle`, `session.status`, `session.updated`; `todo.updated`; `tool.execute.before`, `tool.execute.after`; `tui.prompt.append`, `tui.command.execute`, `tui.toast.show`.

### Matcher syntax

`(none)`. There is no config-time matcher. A plugin's hook fires for *every* invocation of that hook key; filtering on `tool` name, file path, or command pattern happens inside the JavaScript handler body (e.g., `if (input.tool === "bash")`). There is no string/regex/content-predicate matcher comparable to Claude's `if:` or Codex's regex matcher.

### Hook input / output shape

There is no JSON-on-stdin. Handlers receive typed JS objects. Representative shapes from the type definitions:

```ts
// tool.execute.before
input:  { tool: string; sessionID: string; callID: string }
output: { args: any }                       // mutate args; throw Error to block

// tool.execute.after
input:  { tool: string; sessionID: string; callID: string; args: any }
output: { title: string; output: string; metadata: any }

// permission.ask
input:  Permission                          // the SDK Permission object
output: { status: "ask" | "deny" | "allow" }
```

### Decision semantics

- **Block a tool:** `throw` an `Error` inside `tool.execute.before` (the docs' `.env`-protection example throws `new Error("Do not read .env files")`).
- **Allow / deny / ask a permission:** set `output.status` in `permission.ask`. This is the only hook with a Claude-style three-value decision, and `ask` *is* a working value here (it surfaces the interactive prompt).
- **Exit codes:** `(none)`. Plugins run in-process; there is no exit-code contract.

### Trust model

Plugins are arbitrary in-process JS/TS with full `$` shell access and SDK access. There is **no per-plugin trust/approval gate** comparable to Codex's `trusted_hash` workflow: a plugin file dropped into `.opencode/plugins/` runs at next startup. Treat any repo's `.opencode/plugins/` as executable code.

**Known partial coverage:** the `experimental.*` hooks are explicitly experimental and may change shape. `auth` and `provider` hooks are integration surfaces, not enforcement surfaces.

## 6. MCP support

- **Client present:** yes, full MCP client. Source: [MCP docs](https://opencode.ai/docs/mcp-servers/).
- **Transports:** local (stdio, `"type": "local"`) and remote (HTTP/SSE, `"type": "remote"`).
- **Configuration location:** the `mcp` field in `opencode.json` / `opencode.jsonc`; each server keyed by a unique name.
- **Per-server fields:** local: `type`, `command` (array), `environment` (object), `enabled` (bool), `timeout` (ms, default 5000). Remote: `type`, `url`, `enabled`, `headers` (object), `oauth`, `timeout`.
- **OAuth:** supported via the `oauth` field plus CLI helpers.
- **Helper CLI:** `opencode mcp auth <server>`, `opencode mcp list`, `opencode mcp logout <server>`, `opencode mcp debug <server>`.
- **Discovery commands:** CLI helpers above; MCP tools are auto-exposed to the model once enabled. MCP tool permissions are reachable via the `mcp_*` wildcard in the permission block.
- **Codex-style aliasing:** MCP tools are named `<server>_<tool>`, not Claude's `mcp__<server>__<tool>`. This is a portability divergence for any matcher or permission rule that hard-codes the Claude double-underscore form.

## 7. Slash commands / explicit invocation

- **File format:** Markdown with YAML frontmatter; or a JSON `command` object in `opencode.jsonc` with a `template` field.
- **Locations:** `.opencode/commands/` (project) and `~/.config/opencode/commands/` (global). `test.md` defines `/test`. Subdirectory namespacing is supported.
- **Frontmatter fields:** `description`, `agent` (which agent runs the command), `model` (model override), `subtask` (force subagent invocation).
- **Argument injection:** `$ARGUMENTS` (all args as a string), `$1`/`$2`/`$3` (positional).
- **Shell injection:** ``!`command` `` runs a shell command in the project root and inlines its output into the prompt.
- **File injection:** `@filename` inlines file content into the prompt.
- **Reloadability:** `(none)` documented as an in-session reload command; restart the session to pick up new commands.
- **Deprecation status:** slash commands and skills coexist; slash commands are *not* deprecated in favor of skills (unlike Codex, where custom prompts are deprecated).

## 8. Permission model

- **Modes:** the permission system has three values, not modes: `allow` (run without approval), `ask` (prompt the user), `deny` (block). Set via the `permission` field in `opencode.json`. Source: [Permissions docs](https://opencode.ai/docs/permissions/).
- **Default:** most permissions default to `allow`; `doom_loop` and `external_directory` default to `ask`; `read` denies `.env` files by default. There is no separate top-level approval-mode enum.
- **Tool granularity:** per-tool string (`"bash": "allow"`) or wildcard (`"*": "ask"`, `"mcp_*": "ask"`). Per-pattern object form for input-bearing tools: `{"bash": {"*": "ask", "git *": "allow", "rm *": "deny"}}`. Patterns support `*`/`?` wildcards and `~`/`$HOME` expansion, applied to file paths, commands, URLs, and queries. **Rule resolution: last matching rule wins.**
- **Per-pattern allow/deny:** yes. This is a real divergence from Gemini (which has no per-pattern permission rules) and aligns with Codex's granular permissions.
- **Covered tools:** `read`, `edit`, `glob`, `grep`, `bash`, `task`, `skill`, `lsp`, `question`, `webfetch`, `websearch`, `external_directory`, `doom_loop`. The `edit` permission also governs `write` and `apply_patch`.
- **Agent overrides:** an agent's `permission` block merges with the global config; agent rules take precedence. This is how `Plan` enforces read-only behavior.
- **Elevation:** `(none)` documented as a model-initiated mode change. The model surfaces `ask` prompts; it does not request a global permission-mode change mid-run.
- **Sandbox integration:** `(none)`. See section 3; OpenCode has no OS sandbox, so the permission model is the *entire* containment story.
- **Project trust:** `(none)` documented as an untrusted-project gate. Project `.opencode/` config, commands, agents, and plugins load without a trust prompt, which is notable given plugins are arbitrary code (section 5).
- **Managed/enterprise:** `(none)` documented as an MDM-style requirements file.

## 9. Default model & coding behavior

- **Default model:** `(none)` hard-coded. OpenCode has **no fixed default model identifier**. On startup it resolves a model in priority order: (1) `--model`/`-m` flag, (2) `model` key in `opencode.json` (format `provider_id/model_id`), (3) last-used model, (4) the first model by an internal priority. Source: [Models docs](https://opencode.ai/docs/models/). This is a categorical difference from Claude Code, Gemini CLI, and Codex, all of which ship a default.
- **Provider model:** bring-your-own-key. OpenCode uses the AI SDK and [Models.dev](https://models.dev) to support 75+ providers and local models. Providers are added via `/connect`. Optionally, **OpenCode Zen** is a first-party AI gateway: sign in, get a Zen API key, and reference models as `opencode/<model>` (e.g., `opencode/gpt-5.5`). Zen is explicitly optional and not required to use OpenCode.
- **Routing:** `(none)`. Single model per session; no `Auto`-style router. `/models` switches the active model.
- **Reasoning effort:** model variants expose thinking-budget settings (e.g., `high` default, `max`) where the provider supports them; this is per-model, not a harness-wide knob.
- **Context window / max output tokens:** `(reported, not verified)`. OpenCode does not surface a harness-level context window or max-output figure; these are properties of whichever provider model is selected (Models.dev is the source of per-model limits).
- **Compaction:** automatic, with the `Compaction` hidden agent generating a continuation summary near the context limit; plugins can customize it via `experimental.session.compacting`. Manual `/compact` is supported.
- **Pricing:** OpenCode the harness is free and MIT-licensed. When using your own provider key, the provider bills directly. **OpenCode Zen** is pay-as-you-go, billed per request; representative Zen rates per 1M tokens *(reported, from [Zen docs](https://opencode.ai/docs/zen), 2026-05-17)*: Claude Opus 4.7 $5.00 in / $25.00 out / $0.50 cached read; Claude Sonnet 4.6 $3.00 / $15.00 / $0.30; Gemini 3.1 Pro (at or below 200K tokens) $2.00 / $12.00 / $0.20; GPT 5.5 (at or below 272K tokens) $5.00 / $30.00 / $0.50. Some Zen models (Big Pickle, DeepSeek V4 Flash Free, MiniMax M2.5 Free, Nemotron 3 Super Free) are free. Zen passes card-processing fees through at cost.

**Known weaknesses with primary citations** (these are *harness* behaviors; underlying-model behavior depends on which provider model the user picked):

- **Long-session / context-limit failure:** [#27884](https://github.com/anomalyco/opencode/issues/27884) "Model stuck on context full", [#26707](https://github.com/anomalyco/opencode/issues/26707) "Forked session inherits full uncompressed context after parent compaction, causing 'exceeds model context limit'", [#24143](https://github.com/anomalyco/opencode/issues/24143) "Context token count greatly underestimated", [#28057](https://github.com/anomalyco/opencode/issues/28057) "Context percentage indicator doesn't reflect usable context proportion".
- **Compaction quality:** [#25746](https://github.com/anomalyco/opencode/issues/25746) "Compaction updates made the models dumber", [#23998](https://github.com/anomalyco/opencode/issues/23998) "If sending a prompt and compaction happens immediately, compaction excludes the prompt", [#27758](https://github.com/anomalyco/opencode/issues/27758) "Model refused to make a Compaction".
- **Memory leaks in long sessions:** [#21430](https://github.com/anomalyco/opencode/issues/21430) "1.4.0 memory leak. Bun occupies more than 3g memory at most.", [#22198](https://github.com/anomalyco/opencode/issues/22198) "Memory leak: SSE connections stuck in CLOSE_WAIT cause unbounded AsyncQueue growth".
- **Subagent shutdown hang:** [#24391](https://github.com/anomalyco/opencode/issues/24391) "Opencode hangs indefinitely if exited during waiting for subagents to finish".
- **Long-session resume / fork:** [#6653](https://github.com/anomalyco/opencode/issues/6653) "Interrupting long session then resuming results in Internal Server Error", [#16311](https://github.com/anomalyco/opencode/issues/16311) "/fork is incredibly slow for long sessions".

**Enterprise lag caveat:** OpenCode has no documented MDM/managed-config layer, so there is no harness-level enterprise version pin. In practice an org's lag is governed by which provider models its credentials grant, not by the OpenCode binary version.

---

## Practical takeaways

Pointers from the README's portable-skill rules with OpenCode-specific manifestations:

- **Rule 1 (hooks are vendor-specific) is at its most extreme here.** A Claude/Codex shell-script hook does not port to OpenCode *at all*: OpenCode hooks are in-process TypeScript plugin functions, not scripts invoked with JSON on stdin. A skill that ships hooks must ship a separate OpenCode *plugin* alongside the Claude/Codex settings snippet. There is no shared wire format to lean on.
- **Rule 2 (don't extract instructions into hooks unless every harness has the event).** OpenCode's event surface is rich but observation-only for most keys; only `tool.execute.before` (via `throw`) and `permission.ask` (via `output.status`) actually *enforce*. Discipline extracted into a Claude `FileChanged` or `PostToolBatch` hook has no OpenCode equivalent: keep it in the skill or in `AGENTS.md`.
- **Rule 3 (no hard-coded tool names).** OpenCode tool names are lowercase (`edit`, `write`, `bash`, `read`, `grep`, `glob`), divergent from Claude's CamelCase and from Codex's hook-name aliases. MCP tools are `<server>_<tool>`, not `mcp__<server>__<tool>`. Reference the *action*, not the tool name.
- **Rule 5 (no hard-coded memory filename).** OpenCode defaults to `AGENTS.md` and falls back to `CLAUDE.md`. Reference "the always-loaded memory file" generically.
- **Rule 6 (no `ask` UX): OpenCode is an exception worth noting.** Unlike Gemini and Codex, OpenCode's `permission.ask` hook *does* support a working `ask` value, and the permission system has a first-class `ask`. But a skill still should not *depend* on `ask`, because the other harnesses lack it; design for hard-deny and treat working `ask` as a bonus on OpenCode.
- **Rule 9 (long-session behaviors).** OpenCode has documented context-limit, compaction-quality, and memory-leak issues in long sessions (section 9). The supported mitigations are `/compact`, the `experimental.session.compacting` plugin hook, and delegating to `Explore`/`Scout`/`General` subagents for context isolation.

---

## Cross-harness portability notes (vs Claude Code)

*Comparison anchor: 2026-05-17. See [`README.md#claude-code-reference-state`](./README.md#claude-code-reference-state) for the Claude state-of-truth this section anchors against (itself last verified 2026-05-14). When Claude rev's, the anchor in the README updates; this section becomes re-anchored automatically.*

**Headline observation:** OpenCode is the *least* Claude-Code-compatible harness studied. Codex deliberately mirrors Claude's hook spec; Gemini diverges on names but keeps the shell-script plus JSON-stdin model. OpenCode rejects that model entirely: hooks are in-process TypeScript plugins, "events" are observation channels, blocking is `throw`, and there is no `settings.json` `hooks` block, no JSON wire format, and no exit-code contract. There is also no fixed default model. Skill *content* still ports (agentskills.io `SKILL.md`); skill *enforcement machinery* does not.

### Hook event equivalents

OpenCode does not have shell-script hook *events* in the Claude sense. The closest mapping is from Claude hook events to OpenCode plugin hook keys / events. "Equivalent" here means "achievable via a plugin," not "wire-compatible."

| Claude event | OpenCode equivalent | Portability |
|---|---|---|
| `PreToolUse` | `tool.execute.before` plugin hook | PARTIAL (block via `throw`, not a decision JSON; no shell-script form) |
| `PostToolUse` | `tool.execute.after` plugin hook | PARTIAL (observe/mutate output; no shell-script form) |
| `UserPromptSubmit` | `chat.message` plugin hook | PARTIAL (observation only; cannot block the prompt) |
| `Stop` | `session.idle` event | PARTIAL (observation only; cannot force continuation) |
| `SessionStart` | `session.created` event | PARTIAL (observation only) |
| `SessionEnd` | `session.deleted` event | PARTIAL (observation only) |
| `PreCompact` | `experimental.session.compacting` plugin hook | PARTIAL (can customize/replace the compaction prompt; experimental) |
| `PostCompact` | `session.compacted` event / `experimental.compaction.autocontinue` | PARTIAL (observe, or skip the auto-continue turn) |
| `Notification` | `tui.toast.show` event | PARTIAL (observation only) |
| `PermissionRequest` | `permission.ask` plugin hook | FULL-ish (real `allow`/`deny`/`ask` decision via `output.status`) |
| `FileChanged` | `file.edited` / `file.watcher.updated` events | PARTIAL (observation only; cannot block on file change) |
| `SubagentStop` | (none) | CLAUDE-ONLY (no subagent-completion hook despite OpenCode having subagents) |
| `PostToolBatch` | (none) | CLAUDE-ONLY |
| `ConfigChange`, `CwdChanged`, `WorktreeCreate`/`Remove`, `Setup`, `TaskCreated`/`Completed`, `TeammateIdle`, `InstructionsLoaded`, `Elicitation`/`Result`, `StopFailure`, `PermissionDenied` | mostly (none); `permission.replied` loosely echoes `PermissionDenied` | CLAUDE-ONLY (lifecycle / state-watch surface) |

### What doesn't exist here (that Claude Code has)

- **Shell-script hooks.** No `settings.json` `hooks` block, no JSON-on-stdin, no `0`/`2` exit codes. OpenCode hooks are in-process JS/TS plugin functions. A Claude hook script must be rewritten as an OpenCode plugin; there is no mechanical port.
- **Config-time matchers.** No `tool_name` matcher and no `if:` content predicate. A plugin hook fires for every invocation; filtering happens inside the JS handler.
- **`defer` decision value.** OpenCode's `permission.ask` has `allow`/`deny`/`ask` only.
- **Per-plugin trust gate.** Claude/Codex gate new hooks; OpenCode runs any plugin in `.opencode/plugins/` at startup with full shell access. This is a *more* permissive posture, not a missing feature in the user's favor.
- **OS-level sandbox.** No Seatbelt / `bwrap` / Windows sandbox. The permission system is the entire containment story.
- **A fixed default model.** Claude ships Opus 4.7; OpenCode ships none, so the user must configure a provider.
- **`SubagentStop`, `PostToolBatch`, and most lifecycle/state-watch events.** See table.
- **Hard-coded memory filename.** Claude's `CLAUDE.md` is fixed; OpenCode defaults to `AGENTS.md` with `CLAUDE.md` fallback.
- **Project trust model.** Claude has untrusted-project handling; OpenCode loads project `.opencode/` config, agents, commands, and plugins without a trust prompt.

### What exists here that Claude Code doesn't

- **In-process TypeScript plugins** with full SDK plus Bun-shell access, far more powerful than shell-script hooks (custom tools, provider/auth registration, message/system-prompt rewriting, LLM-param mutation), at the cost of zero portability and no trust gate.
- **`chat.params` / `chat.headers` hooks** that mutate the LLM request parameters and HTTP headers per turn.
- **`experimental.chat.system.transform`** that rewrites the system prompt array before assembly.
- **`tool.definition` hook** that rewrites a tool's description/parameters before the model sees it.
- **`tool` registration hook** so plugins add first-class custom tools that sit alongside built-ins.
- **`auth` / `provider` hooks** so plugins register new model providers and OAuth/API auth flows.
- **75+ provider support via Models.dev** plus local models, with no harness-mandated default.
- **OpenCode Zen**, an optional first-party AI gateway with per-request pay-as-you-go billing and some free models.
- **Per-pattern bash permission rules** (`"git *": "allow"`, `"rm *": "deny"`) with last-match-wins resolution. Claude has per-pattern `allow`/`deny`, but OpenCode's command-substring form is ergonomically distinct.
- **Distinct `Build` / `Plan` primary agents and `Explore` / `Scout` / `General` subagents** shipped built-in, each with its own permission profile.

---

## Known volatile (re-verify first on revisit)

- **Plugin hook key list.** The `Hooks` interface in `packages/plugin/src/index.ts` is the source of truth and changes with releases; several keys are `experimental.*` and may rename or stabilize.
- **OpenCode Zen model catalog and pricing.** Zen is a curated, frequently-updated model list; the pricing in section 9 will rot fast. Re-verify against [opencode.ai/docs/zen](https://opencode.ai/docs/zen).
- **Subagent system.** Built-in agent roster (`Build`/`Plan`/`General`/`Explore`/`Scout`) and child-session navigation are evolving; no documented concurrency caps yet.
- **Default-model resolution.** No hard-coded default; the four-step priority order could change.
- **Event list.** The `event`-hook event catalog is broad and still expanding.
- **Org/repo naming.** Repo moved `sst` to `anomalyco`; `sst/opencode` redirects today but the canonical name may shift again.

## Known stable (low rot risk)

- **`SKILL.md` filename and agentskills.io layout** (`.agents/skills/`, `.claude/skills/` read paths), an externally referenced standard.
- **`AGENTS.md` as the primary memory filename** with `CLAUDE.md` fallback, a cross-vendor convention.
- **The plugin model itself.** In-process JS/TS plugins are a deliberate architectural choice unlikely to be replaced by shell hooks.
- **Permission values `allow`/`ask`/`deny`** and the `opencode.json` `permission` block.
- **MIT license and bring-your-own-provider posture.**
- **MCP client support**, since MCP is upstream of the harness.

## Open questions

- **Subagent concurrency limits.** Whether OpenCode caps parallel child sessions, and at what number.
- **Context window / max output tokens.** Not surfaced at the harness level; entirely provider-model dependent (Models.dev).
- **Plugin trust roadmap.** Whether OpenCode will add a per-plugin approval gate given plugins run arbitrary code without one.
- **`experimental.*` hook stability.** Which experimental hook keys are on track to stabilize, and on what timeline.
- **Whether a `.claude/skills/` skill with Claude-specific hooks degrades silently** on OpenCode. The skill content loads, but bundled shell-script hooks have no loader.
- **Crush divergence.** `charmbracelet/crush` shares ancestry but is a separate harness; it would need its own study to compare.
