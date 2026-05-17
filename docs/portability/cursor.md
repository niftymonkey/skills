# Cursor CLI - harness study

| | |
|---|---|
| **Harness** | [Cursor CLI](https://cursor.com/docs/cli/overview) (`cursor-agent`; closed-source; installed via `curl https://cursor.com/install`; the on-disk binary and shell command are both `agent`) |
| **Verified** | 2026-05-17 |
| **Harness version studied** | Cursor application release line 3.4 (2026-05-13); CLI ships on a rolling channel (`agent update`); skills require Cursor 2.4+; `cursor_version` in hook input observed as `1.7.x`-style strings |
| **Framework** | [`README.md`](./README.md) |
| **Refresh recipe** | [`research-prompt.md`](./research-prompt.md) (general; supply preamble with harness name, URL, companion file `cursor.md`) |

A snapshot of the Cursor CLI's affordances, structured against the framework's nine categories. Treat facts as authoritative only against the `Verified:` date; primary-source links inline are how to re-verify.

**Headline:** Cursor is closed-source, so every fact here traces to official `cursor.com/docs` pages rather than source. The CLI (`cursor-agent`, invoked as `agent`) shares its affordance surface with the Cursor IDE: same rules, skills, hooks, MCP config, and models. Cursor deliberately reads Claude Code and Codex on-disk layouts (`.claude/`, `.codex/`, `AGENTS.md`, `CLAUDE.md`) and accepts Claude Code's hook JSON formats, which makes it the most omnivorous port target studied so far. The divergence is the hook event vocabulary: Cursor has its own large, granular, lower-camelCase event set (`beforeShellExecution`, `afterFileEdit`, etc.) that does not name-match Claude's, even though Claude-named hooks are auto-mapped at load time.

---

## 1. Skill loading & discovery

- **Workspace skill path:** `.cursor/skills/<name>/SKILL.md` and `.agents/skills/<name>/SKILL.md`. The skills root is walked recursively, so category subfolders (`.cursor/skills/shipping/land-it/SKILL.md`) and nested-project roots (`apps/web/.cursor/skills/`) are both discovered. Source: [Agent Skills docs](https://cursor.com/docs/context/skills).
- **User-scope skill path:** `~/.cursor/skills/` and `~/.agents/skills/`.
- **Cross-vendor compatibility paths:** Cursor additionally loads skills from `.claude/skills/`, `.codex/skills/`, `~/.claude/skills/`, and `~/.codex/skills/` ([source](https://cursor.com/docs/context/skills)). Admin/enterprise skill scope: `(none)` documented as a distinct skill path (enterprise governance is applied through Team rules and managed hooks instead).
- **Auto-activation:** description-based. At startup Cursor discovers all skills and presents their `description` to the agent, which decides relevance from context. Explicit invocation is `/skill-name` (run) or `@skill-name` (attach as context) in chat.
- **Manifest standard:** the [Agent Skills](https://cursor.com/docs/context/skills) open standard, meaning `SKILL.md` plus optional `scripts/`, `references/`, `assets/`. This is the same `SKILL.md` layout the `vercel-labs/skills` CLI installs via [agentskills.io](https://agentskills.io); the `.agents/skills/` path ports cleanly.
- **Frontmatter honored:** `name` (required; lowercase letters, numbers, hyphens; must match the parent folder), `description` (required), `paths` (optional; comma-separated string or list of globs that scope the skill to matching files), `disable-model-invocation` (optional; `true` makes the skill explicit-invocation-only, like a slash command), `metadata` (optional; arbitrary key-value map). The legacy `globs` field is still accepted as a fallback for older skills. Source: [Skills frontmatter table](https://cursor.com/docs/context/skills).
- **Scope precedence:** project subdirectory skills are auto-scoped to their directory subtree (a skill under `apps/web/.cursor/skills/` is surfaced only when the agent touches files under `apps/web/`). Documentation does not specify merge/precedence behavior when the same skill `name` appears in multiple scopes. See Open questions.
- **Discovery commands:** type `/` in chat to fuzzy-search skills; `/create-skill` scaffolds a new skill via a built-in skill; `/migrate-to-skills` (Cursor 2.4+) converts eligible rules and slash commands into skills.

## 2. Memory / context files

- **Default filename:** `AGENTS.md` at the project root, plus `CLAUDE.md` at the project root. The CLI reads **both** and applies them as rules alongside `.cursor/rules` ([source](https://cursor.com/docs/cli/using)). Cursor's own native context system is the four-type Rules system (see below), not a single memory file.
- **Rule types:** Project Rules (`.cursor/rules/*.md` / `*.mdc`, version-controlled), User Rules (global to the Cursor account), Team Rules (dashboard-managed, Team/Enterprise plans), and `AGENTS.md`. Source: [Rules docs](https://cursor.com/docs/context/rules).
- **Configurable filename:** `(none)`. `AGENTS.md` and `CLAUDE.md` are fixed names; there is no `context.fileName`-style override. Customization happens through `.cursor/rules` files, not by renaming the memory file.
- **Scope and load order:** `.cursor/rules` files apply per their frontmatter (`alwaysApply: true` always; `globs` when a matching file is in context; `description` when the agent judges relevance; none of those set means `@`-mention-only). Rule contents are injected at the start of the model context. `AGENTS.md` / `CLAUDE.md` apply as rules. Documentation does not specify a strict precedence ordering between `AGENTS.md`, `CLAUDE.md`, and `.cursor/rules` when they conflict.
- **Size cap:** `(none)` documented as a hard byte cap. Best-practice guidance is to keep individual rules under 500 lines and split large rules.
- **Import syntax:** `@filename` references inside a rule file pull in another file (e.g., `@migration-template.sql`, `@service-template.ts`). This references files for the agent rather than textually concatenating them.
- **Inspection / scaffold commands:** `/rules` (create or edit rules in chat); `agent generate-rule` / `agent rule` (CLI subcommand to generate a rule interactively); `+ Add Rule` in `Cursor Settings > Rules, Commands`.

## 3. Tool ecosystem

The CLI does not publish an exhaustive built-in-tool catalog the way an open-source harness does. Tool names are observable through the hook matcher vocabulary and the `--output-format stream-json` tool-call payloads.

| Category | Tool / matcher name | Notes |
|---|---|---|
| File system | `Read`, `Write`, `Delete` | Matcher values for `preToolUse` / `postToolUse`. Stream-JSON exposes `readToolCall` and `writeToolCall` payloads with `args.path` / `args.fileText`. |
| Search | `Grep` | Matcher value for tool-use hooks. |
| Shell | `Shell` | Matcher value; the `beforeShellExecution` hook receives the full `command` string. |
| Subagents | `Task` | The `Task` tool spawns subagents; `subagentStart` / `subagentStop` hooks gate it. |
| Web | web fetch tool | Gated by `WebFetch(domain)` permission tokens; exact tool name not surfaced in CLI docs. |
| Tab (inline completions) | `TabRead`, `TabWrite` | Matcher values for the separate Tab hook surface. |
| MCP | `MCP: <server>:<tool>` | Matcher format for MCP tool calls in `preToolUse`; per-call hooks `beforeMCPExecution` / `afterMCPExecution`. |

Source: [hook matcher table](https://cursor.com/docs/agent/hooks) ("Available matchers by hook").

**Naming convention:** CamelCase nouns for tool matchers (`Shell`, `Read`, `Write`, `Grep`, `Delete`, `Task`). This partially matches Claude Code's `Read`/`Write` and diverges on `Shell` (vs Claude's `Bash`) and `Delete`. There is no separate model-facing edit/replace tool name documented: file edits surface as `Write`, with `afterFileEdit` carrying an `edits[]` array of `{old_string, new_string}`.

**Default enablement:** all tools available; mutating tools (shell, write) prompt for approval in interactive mode. In print mode (`-p` / `--print`), Cursor has "full write access," but file modifications still require `--force` / `--yolo` to be applied rather than merely proposed ([source](https://cursor.com/docs/cli/headless)).

**Modes (the CLI's three operating modes):** `agent` (default; full tool access), `plan` (design first, asks clarifying questions; via `--plan`, `--mode=plan`, `/plan`, or Shift+Tab), `ask` (read-only exploration; via `--mode=ask` or `/ask`). Source: [CLI overview](https://cursor.com/docs/cli/overview).

**Approval controls:** `/auto-run [on|off|status]` toggles auto-run of commands; `-f` / `--force` (alias `--yolo`) force-allows commands unless explicitly denied.

**Sandbox:** `--sandbox <enabled|disabled>` or the `/sandbox` interactive menu. Sandbox mode toggles command execution isolation and network access; the setting persists across sessions. Sudo-requiring commands trigger a masked password prompt routed directly to `sudo` over IPC (the model never sees the password). The docs do not name the underlying sandbox primitive (Seatbelt / bwrap / etc.) for the CLI. The `beforeShellExecution` hook input carries a `sandbox: boolean` field. CLI worktrees (`--worktree`) isolate edits under `~/.cursor/worktrees`.

**Protected paths:** `(none)` documented as a fixed protected-path set inside writable roots; protection is expressed through `permissions.deny` tokens (e.g., `Read(.env*)`, `Write(**/*.key)`).

## 4. Sub-agents / delegation

- **Available:** yes, usable in the CLI, the editor, and Cloud Agents. Source: [Subagents docs](https://cursor.com/docs/agent/subagents).
- **Definition format:** Markdown file with YAML frontmatter, one agent per file. Project: `.cursor/agents/` (also reads `.claude/agents/` and `.codex/agents/` for compatibility). User: `~/.cursor/agents/` (also `~/.claude/agents/`, `~/.codex/agents/`).
- **Frontmatter fields:** `name`, `description` (used for routing); `model` (accepts `inherit`); `readonly` (boolean). The body is the subagent's system prompt.
- **Built-in subagents:** `Explore` (codebase search; faster model; runs many parallel searches), `Bash` (runs shell command series; isolates verbose output), `Browser` (drives a browser via MCP; filters DOM noise). These run automatically without configuration. Hook-visible subagent-type values include `generalPurpose`, `explore`, `shell`.
- **Parallel execution:** yes, multiple subagents launch simultaneously. The `subagentStart` hook input carries `is_parallel_worker: boolean`. No documented numeric concurrency cap.
- **Context isolation:** yes, each subagent starts with a clean context window and no access to prior conversation history; the parent must pass needed context in the spawn prompt. The subagent returns a final summary message.
- **Invocation:** the parent agent launches subagents automatically when a task warrants it. Custom subagents are encoded as files; documentation describes natural-language delegation rather than an explicit `@agent` invocation token in the CLI.
- **Background / async:** yes, subagents run Foreground (block until complete) or Background (return immediately, work independently). Distinct from the Cloud Agent handoff (`&` prefix), which pushes the whole conversation to a remote agent at `cursor.com/agents`.
- **Precedence:** project subagents override user subagents on name conflict; `.cursor/` overrides `.claude/` or `.codex/`.

## 5. Hooks / deterministic enforcement

**Headline:** Cursor's hook surface is the largest and most granular of the harnesses studied. It is not event-name-compatible with Claude Code, but Cursor explicitly **maps Claude Code hook names to its own** at load time and accepts both Claude's nested `hookSpecificOutput` JSON and Cursor's flat JSON. The vocabulary is lower-camelCase and split across three surfaces (Agent, Tab, App-lifecycle).

- **Configuration location:** `hooks.json` (a dedicated file, not the CLI config). Layers, highest precedence first: Enterprise (`/Library/Application Support/Cursor/hooks.json` on macOS, `/etc/cursor/hooks.json` on Linux/WSL, `C:\ProgramData\Cursor\hooks.json` on Windows), then Team (dashboard-distributed, Enterprise only), then Project (`.cursor/hooks.json`), then User (`~/.cursor/hooks.json`), then the Claude Code compatibility layers (`.claude/settings.local.json`, then `.claude/settings.json`, then `~/.claude/settings.json`). All matching hooks from every layer run; higher-priority sources win on conflict. Source: [Hooks docs](https://cursor.com/docs/agent/hooks), [Third Party Hooks](https://cursor.com/docs/reference/third-party-hooks).
- **Top-level shape:** `{ "version": 1, "hooks": { "<eventName>": [ { "command": "...", "matcher": "...", "timeout": <s>, "loop_limit": <n|null>, "failClosed": <bool>, "type": "command"|"prompt" } ] } }`.
- **Hook types:** `command` (default; shell script over stdio JSON) and `prompt` (LLM-evaluated natural-language condition returning `{ ok, reason? }`; `$ARGUMENTS` is replaced with hook input JSON; optional `model` override).

### Event types

Source: [Hooks reference](https://cursor.com/docs/agent/hooks).

**Agent hooks** (fire during a Cmd+K / Agent Chat / CLI session):

| Event | Trigger | Can block? | Key output fields |
|---|---|---|---|
| `sessionStart` | New conversation created | No (fire-and-forget) | `env`, `additional_context` |
| `sessionEnd` | Conversation ends | No (fire-and-forget) | (none) |
| `beforeSubmitPrompt` | User hits send, before backend request | Yes | `continue`, `user_message` |
| `preToolUse` | Before any tool runs (generic, all tools) | Yes | `permission` (`allow`/`deny`; `ask` accepted but not enforced), `user_message`, `agent_message`, `updated_input` |
| `postToolUse` | After a tool succeeds | No | `updated_mcp_tool_output`, `additional_context` |
| `postToolUseFailure` | Tool fails, times out, or is denied | No | (none) |
| `beforeShellExecution` | Before a shell command runs | Yes | `permission` (`allow`/`deny`/`ask`), `user_message`, `agent_message` |
| `afterShellExecution` | After a shell command | No | (none) |
| `beforeMCPExecution` | Before an MCP tool runs | Yes | `permission` (`allow`/`deny`/`ask`), `user_message`, `agent_message` |
| `afterMCPExecution` | After an MCP tool | No | (none) |
| `beforeReadFile` | Before the agent reads a file | Yes | `permission` (`allow`/`deny`), `user_message` |
| `afterFileEdit` | After the agent edits a file | No | (none) |
| `subagentStart` | Before a `Task`-tool subagent spawns | Yes | `permission` (`allow`/`deny`; `ask` treated as `deny`), `user_message` |
| `subagentStop` | A subagent completes/errors/aborts | No (can auto-continue) | `followup_message` |
| `preCompact` | Before context compaction | No (observational) | `user_message` |
| `stop` | Agent loop ends | No (can auto-continue) | `followup_message` |
| `afterAgentResponse` | Agent finished an assistant message | No | (none) |
| `afterAgentThought` | Agent finished a thinking block | No | (none) |

**Tab hooks** (fire only for autonomous inline completions, not the agent): `beforeTabFileRead` (can `deny`), `afterTabFileEdit` (observational; carries `range`, `old_line`, `new_line`).

**App-lifecycle hook:** `workspaceOpen` (fires when Cursor opens a workspace and on every workspace-folder change; can return additional plugin paths; runs in both the desktop app and the CLI).

### Matcher syntax

- The `matcher` is a **regex string**, but **what it matches against depends on the event** ([source](https://cursor.com/docs/agent/hooks)):
  - `preToolUse` / `postToolUse` / `postToolUseFailure`: tool type (`Shell`, `Read`, `Write`, `Grep`, `Delete`, `Task`, or `MCP: <server>:<tool>`).
  - `subagentStart` / `subagentStop`: subagent type (`generalPurpose`, `explore`, `shell`).
  - `beforeShellExecution` / `afterShellExecution`: the **full shell command string**, a genuine content predicate rather than just a tool name.
  - `beforeReadFile`: tool type (`TabRead`, `Read`).
  - `afterFileEdit`: tool type (`TabWrite`, `Write`).
  - `beforeSubmitPrompt`, `stop`, `afterAgentResponse`, `afterAgentThought`: matched against a fixed event-label string (`UserPromptSubmit`, `Stop`, `AgentResponse`, `AgentThought`).

### Hook input JSON (stdin), common fields

All hooks receive a base set, plus event-specific fields:

```json
{
  "conversation_id": "string",
  "generation_id": "string",
  "model": "string",
  "hook_event_name": "string",
  "cursor_version": "1.7.2",
  "workspace_roots": ["/path"],
  "user_email": "string | null",
  "transcript_path": "string | null"
}
```

Event-specific examples: `preToolUse` adds `tool_name`, `tool_input`, `tool_use_id`, `cwd`, `agent_message`; `postToolUse` adds `tool_output` (JSON-stringified), `duration`; `beforeShellExecution` adds `command`, `cwd`, `sandbox`; `afterFileEdit` adds `file_path`, `edits[]`; `preCompact` adds `trigger`, `context_usage_percent`, `context_tokens`, `context_window_size`, `messages_to_compact`, `is_first_compaction`.

### Hook output JSON

There is no single universal output block; the shape is per-event (see the table above). The principal decision field is `permission` (`"allow"` / `"deny"`, plus `"ask"` on the shell/MCP `before*` events). Side-effect fields are `updated_input` (rewrite tool input on `preToolUse`), `updated_mcp_tool_output` (rewrite MCP output on `postToolUse`), `additional_context` (inject context), `followup_message` (auto-submit a next user message on `stop` / `subagentStop`), and `env` (set session env vars on `sessionStart`).

### Exit-code semantics

- `0`: hook succeeded; the JSON on stdout is used.
- `2`: block the action (equivalent to `permission: "deny"`).
- Any other exit code, a crash, a timeout, or invalid JSON: **fail-open by default** (the action proceeds). Set `failClosed: true` per hook definition to fail-closed instead; this is recommended for security-critical `beforeMCPExecution` / `beforeReadFile` hooks.

### Trust model & loop limits

Project hooks run only in a **trusted workspace**. The `stop` and `subagentStop` auto-follow-up loops are capped by `loop_limit` (default `5` for native Cursor hooks, `null` / uncapped for hooks loaded via the Claude Code compatibility path; `null` removes the cap entirely).

**Known partial coverage:** `preToolUse` accepts `permission: "ask"` in its schema but **does not enforce it** today; only `beforeShellExecution` / `beforeMCPExecution` enforce `ask`. `subagentStart` accepts `ask` but treats it as `deny`. `sessionStart` accepts `continue` / `user_message` but does not enforce them (session creation is never blocked). `preCompact`, `afterAgentResponse`, `afterAgentThought`, `afterShellExecution`, `afterMCPExecution`, `afterFileEdit`, `postToolUseFailure`, and `afterTabFileEdit` have no enforceable output.

## 6. MCP support

- **Client present:** yes. The CLI auto-detects and respects the same `mcp.json` the editor uses. Source: [CLI using](https://cursor.com/docs/cli/using), [MCP docs](https://cursor.com/docs/context/mcp).
- **Transports:** `stdio`, `SSE`, and `Streamable HTTP`.
- **Configuration location:** `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global). Top-level `mcpServers` object.
- **Per-server fields:** stdio servers take `type`, `command`, `args`, `env`, `envFile`; remote servers take `url`, `headers`, and an optional `auth` object (`CLIENT_ID`, `CLIENT_SECRET`, `scopes`) for static OAuth. Config interpolation (`${env:NAME}`, `${workspaceFolder}`, `${userHome}`, `${pathSeparator}`) is supported in `command`, `args`, `env`, `url`, `headers`.
- **OAuth:** supported, including static OAuth client credentials and dynamic client registration. Fixed redirect URL `cursor://anysphere.cursor-mcp/oauth/callback`. CLI auth subcommand: `agent mcp login <server>`.
- **Helper CLI:** `agent mcp list`, `agent mcp list-tools <server>`, `agent mcp enable <server>`, `agent mcp disable <server>`, `agent mcp login <server>`. The `--approve-mcps` global flag auto-approves all MCP servers.
- **Discovery commands:** `/mcp list`, `/mcp enable <server>`, `/mcp disable <server>` in chat.
- **Protocol extensions:** Tools, Prompts, Resources, Roots, Elicitation, and the MCP Apps extension (interactive UI views) are all supported.
- **Cursor as MCP server:** `(none)` documented for the CLI. The CLI can instead run as an **ACP** (Agent Client Protocol) server: `agent acp` exposes Cursor over stdio with JSON-RPC for custom ACP clients (a hidden, advanced command).

## 7. Slash commands / explicit invocation

- **Built-in catalog:** large, including `/plan`, `/ask`, `/model`, `/auto-run`, `/sandbox`, `/max-mode`, `/new-chat`, `/vim`, `/help`, `/feedback`, `/resume`, `/usage`, `/about`, `/copy-request-id`, `/copy-conversation-id`, `/logout`, `/quit`, `/setup-terminal`, `/mcp`, `/rules`, `/commands`, `/compress`. See [slash commands reference](https://cursor.com/docs/cli/reference/slash-commands).
- **User custom commands:** Markdown files in `.cursor/commands/` (project) or `~/.cursor/commands/` (user). The kebab-case filename becomes the command name. Plain Markdown body of instructions; no required frontmatter. `/commands` in chat creates or edits them.
- **Argument injection:** `(none)` documented as a templated placeholder syntax for command files (unlike Gemini's `{{args}}` or Codex's `$1`/`$ARGUMENTS`). Command files are instruction text the agent reads.
- **Shell injection:** `(none)` documented inside command files.
- **File injection:** `@`-mention in chat attaches a file or skill to context; `@filename` inside a *rule* file references another file. No `@{file}`-style interpolation inside command files is documented.
- **Reload:** hooks config is watched and hot-reloaded automatically; for command/skill/rule files, `(none)` documented as an explicit in-session reload command.
- **Direction of travel:** Cursor is steering reusable prompts toward **skills**. `/migrate-to-skills` (Cursor 2.4+) converts both user-level and workspace-level slash commands into skills, preserving explicit-invocation behavior. Custom commands are not deprecated but skills are the recommended surface for multi-step workflows.

## 8. Permission model

- **Operating modes:** `agent` (default), `plan`, `ask`. See section 3.
- **Permission tokens:** the CLI config (`cli-config.json` global, `cli.json` project) has a `permissions` object with `allow` and `deny` string arrays. Token forms: `Shell(commandBase)` (supports glob and `command:args` syntax, e.g. `Shell(curl:*)`), `Read(pathOrGlob)`, `Write(pathOrGlob)`, `WebFetch(domainOrPattern)`, `Mcp(server:tool)`. Globs use `**`, `*`, `?`. **`deny` always overrides `allow`.** Only the `permissions` block is configurable at the project level; all other CLI settings are global. Source: [Permissions docs](https://cursor.com/docs/cli/reference/permissions).
- **Auto-run / force:** `/auto-run` toggles command auto-execution; `-f` / `--force` (alias `--yolo`) force-allows everything not explicitly denied. In print mode, `--force` is required to actually write files. `--trust` trusts a workspace without prompting (headless mode only).
- **Sandbox modes:** `--sandbox <enabled|disabled>` or `/sandbox`, controlling command isolation and network access; the setting persists.
- **Network gating:** the web-fetch tool prompts per fetch unless the domain is in `WebFetch(...)` allow tokens; sandbox network access is toggled through `/sandbox`.
- **Elevation:** the model cannot self-elevate its permission mode. The `beforeShellExecution` / `beforeMCPExecution` hooks can return `permission: "ask"` to route a single action to a user prompt. Sudo-requiring shell commands trigger a masked password prompt.
- **Auto-review / guardian:** `(none)` in the CLI itself. Cursor's reviewer-agent surface (Bugbot) is a PR-review product, not a CLI runtime guardian.
- **Project trust model:** project-scoped hooks and project config load only in a **trusted workspace**; `--trust` opts in for headless runs.
- **Managed/enterprise:** Enterprise-managed hooks (MDM-deployed system-wide config) and Team hooks (dashboard cloud distribution, synced every ~30 minutes) are not user-disablable. Team Rules are dashboard-managed on Team/Enterprise plans.

## 9. Default model & coding behavior

- **Default model:** `Auto`. Cursor selects a model balancing intelligence, cost, and reliability. Set or change with the `/model` slash command (`/model auto`, `/model gpt-5.2`, `/model sonnet-4.5-thinking`) or the `--model` flag; `agent models` / `--list-models` enumerate options. Source: [Configuration docs](https://cursor.com/docs/cli/reference/configuration), [Models & Pricing](https://cursor.com/docs/models).
- **Routing:** `Auto` is a router, not a fixed model. Cursor's own `Composer 2` model also draws from the same low-cost usage pool as `Auto`. The model that actually answered a turn is visible in `stream-json` `system`/`init` events (`"model"`) and in hook input.
- **Reasoning / Max Mode:** `/max-mode [on|off]` toggles extended reasoning on models that support it. Provider-specific reasoning-effort variants exist (e.g., `gpt-5.2-high`, `sonnet-4.5-thinking`).
- **Context window:** model-dependent; not surfaced as a single CLI-level number. The `preCompact` hook input reports `context_window_size` per conversation (the docs' example shows `128000`). Anthropic models support up to 1M tokens in Max Mode.
- **Max output tokens:** *(reported, not verified)*, not surfaced in CLI docs.
- **Compaction:** automatic near the context limit, plus manual `/compress`. The `preCompact` hook observes (but cannot block) compaction and reports `trigger: "auto"|"manual"`.
- **Available alternates:** broad. Cursor's API pool lists Anthropic (`Claude 4.5/4.6/4.7 Opus`, `Claude 4.5/4.6 Sonnet`, `Claude 4.5 Haiku`), OpenAI (`GPT-5`, `GPT-5.1/5.2/5.3 Codex` family, `GPT-5 Mini`), Google (`Gemini 3 Pro`, `Gemini 3.1 Pro`, `Gemini 3 Flash`), and Cursor's own `Composer 1/1.5/2`. Many are hidden by default and several require Max Mode.
- **Known weaknesses:** Cursor is closed-source, so there is no public upstream issue tracker to cite for instruction-drift / long-session-degradation issues the way the Codex and Gemini studies cite GitHub issues. A community report ([forum thread](https://forum.cursor.com/t/cannot-use-default-model-in-latest-cursor-cli-version-on-grandfathered-plan/155372)) describes the `Auto` default model being temporarily unavailable on some plans after a usage-limit switch. Treat long-session behavior as model-dependent (it varies with whichever model `Auto` routed to) and undocumented at primary-source level. See Open questions.
- **Pricing:** verified against Cursor's [Models & Pricing page](https://cursor.com/docs/models). The `Auto + Composer` pool: `Auto` is billed at $1.25 / 1M input (and cache write), $6.00 / 1M output, $0.25 / 1M cache read; `Composer 2` at $0.50 / $2.50 / $0.20 (no cache-write line). API-pool models are billed at each provider's API rate (e.g., `Claude 4.7 Opus`: $5 input / $25 output / $0.50 cache read / $6.25 cache write). Individual plans bundle two pools; pricing is plan-and-model dependent.
- **Enterprise lag:** Enterprise plans pin model availability and governance through Team Rules, managed hooks, and dashboard controls. No specific version-lag interval is documented; large orgs typically restrict the model menu rather than lag a CLI version, since the CLI auto-updates on a rolling channel.

---

## Practical takeaways

Pointers from the README's portable-skill rules, with Cursor-specific manifestations:

- **Rule 1 / 8 (hooks are vendor-specific; ship per-harness snippets).** Cursor's hook surface is large and genuinely divergent in *event names*, but Cursor *reads Claude Code's `hooks` block directly* and maps the names. A Claude `settings.json` hooks block dropped into `.claude/settings.json` runs on Cursor unchanged. This is the strongest hook interop of any harness studied, but it is one-directional (Cursor reads Claude's format, not the reverse).
- **Rule 2 (don't extract instructions into hooks without an equivalent event).** Cursor's event list is a *superset* in several places (`beforeReadFile`, `afterAgentThought`, `workspaceOpen`, `beforeTabFileRead`) and a near-match in others. The genuine portability trap is the reverse direction: hooks authored against Cursor-only events (`afterAgentThought`, the Tab hooks) have no Claude or Codex equivalent.
- **Rule 3 (don't hard-code tool names).** Cursor tool/matcher names are CamelCase (`Read`, `Write`, `Grep`, `Delete`, `Task`) and `Shell`, close to Claude's but `Shell` is not `Bash` and `Delete` is Cursor-specific. Prefer the action ("run the command", "edit the file") over the tool name.
- **Rule 5 (don't assume the memory filename).** Cursor reads `AGENTS.md` *and* `CLAUDE.md` *and* `.cursor/rules` simultaneously, with no documented precedence between them. Reference "the always-loaded memory file" generically; do not assume a single canonical file.
- **Rule 6 (don't assume `ask`-style permission UX).** Cursor's `ask` decision works only on `beforeShellExecution` / `beforeMCPExecution`. On `preToolUse` it is parsed but not enforced; on `subagentStart` it degrades to `deny`. Design skill discipline for hard-deny with a clear `user_message`/`agent_message`.
- **Rule 7 (don't assume matcher granularity).** Mixed picture: most Cursor hook matchers see only a tool/subagent type, but `beforeShellExecution` / `afterShellExecution` matchers *do* see the full command string, a real content predicate Claude lacks. Do not rely on it for non-shell events; filter inside the script body there.
- **Rule 9 (long-session behaviors).** Cursor's `Auto` router means the underlying model can change mid-project, so long-session behavior is not a fixed property. Skills needing multi-turn adherence should ship `/compress` checkpoints or delegate to subagents.

---

## Cross-harness portability notes (vs Claude Code)

*Comparison anchor: 2026-05-17. Claude Code reference state as of 2026-05-14. See [`README.md#claude-code-reference-state`](./README.md#claude-code-reference-state) for the Claude state-of-truth this section anchors against. When Claude rev's, the anchor in the README updates; this section becomes re-anchored automatically.*

**Headline observation:** Cursor is the most omnivorous port target studied. It reads Claude Code's and Codex's on-disk layouts (`.claude/`, `.codex/`, `AGENTS.md`, `CLAUDE.md`, `.claude/settings.json` hooks) and accepts Claude's hook JSON formats. But its native hook event vocabulary is its own: large, granular, lower-camelCase, and split across three surfaces. Skills and rules port well; hooks port only because Cursor does the translation, not because the names match.

### Hook event equivalents

| Cursor event | Claude equivalent | Portability |
|---|---|---|
| `preToolUse` | `PreToolUse` | FULL (Claude name auto-mapped; `ask` parsed but not enforced on Cursor) |
| `postToolUse` | `PostToolUse` | FULL (auto-mapped) |
| `beforeSubmitPrompt` | `UserPromptSubmit` | FULL (auto-mapped) |
| `stop` | `Stop` | FULL (auto-mapped; `decision: block` maps to `followup_message`) |
| `subagentStop` | `SubagentStop` | FULL (auto-mapped) |
| `subagentStart` | (none) | CURSOR-ONLY (Claude has no pre-subagent gate) |
| `sessionStart` | `SessionStart` | PARTIAL (Cursor is fire-and-forget; cannot block) |
| `sessionEnd` | `SessionEnd` | PARTIAL (advisory in both) |
| `preCompact` | `PreCompact` | PARTIAL (Cursor cannot block compaction; Claude can) |
| `beforeShellExecution` / `afterShellExecution` | (none directly) | CURSOR-ONLY (shell-specific; Claude routes shell through `PreToolUse`/`PostToolUse`) |
| `beforeMCPExecution` / `afterMCPExecution` | (none directly) | CURSOR-ONLY (MCP-specific gate) |
| `beforeReadFile` | (none) | CURSOR-ONLY (pre-read access control) |
| `afterFileEdit` | (partly) `PostToolUse` | CURSOR-ONLY granularity (dedicated post-edit event with `edits[]`) |
| `postToolUseFailure` | (none) | CURSOR-ONLY (dedicated tool-failure event) |
| `afterAgentResponse` / `afterAgentThought` | (none) | CURSOR-ONLY (reasoning/response observation) |
| `beforeTabFileRead` / `afterTabFileEdit` | (none) | CURSOR-ONLY (separate inline-completion hook surface) |
| `workspaceOpen` | (partly) `CwdChanged` / `WorktreeCreate` | CURSOR-ONLY (workspace-open / folder-change event) |
| (none) | `PostCompact` | CLAUDE-ONLY (Cursor observes only the pre-compaction event) |
| (none) | `Notification`, `PostToolBatch`, `ConfigChange`, `Setup`, `TaskCreated`/`Completed`, `TeammateIdle`, `InstructionsLoaded`, `Elicitation`/`Result`, `StopFailure`, `PermissionRequest`/`Denied` | CLAUDE-ONLY (lifecycle / state-watch surface) |

### What doesn't exist here (that Claude Code has)

- **Working `ask` on generic tool use.** Cursor enforces `ask` only on `beforeShellExecution` / `beforeMCPExecution`. On `preToolUse` it is parsed but ignored; on `subagentStart` it degrades to `deny`. Claude's `ask` works across `PreToolUse`.
- **`defer` decision value.** Claude has four (`allow`/`deny`/`ask`/`defer`); Cursor effectively has `allow`/`deny` plus a shell/MCP-only `ask`.
- **`if:` content predicate on arbitrary tool hooks.** Claude can filter on `tool_input` globs at config-load time for any tool. Cursor's matcher is a content predicate *only* for the shell `before*`/`after*` events; everywhere else it sees just a type.
- **`PostCompact` hook.** Cursor observes `preCompact` only.
- **Blocking compaction.** Claude's `PreCompact` can block; Cursor's `preCompact` is observational.
- **A hard-coded single memory filename.** Claude's `CLAUDE.md` is the one fixed memory file. Cursor layers `AGENTS.md` plus `CLAUDE.md` plus the four-type Rules system with no documented precedence.
- **The broad lifecycle / state-watch event surface** (`ConfigChange`, `Setup`, task and teammate events, etc.). See table.
- **Custom slash commands with argument/shell/file injection syntax.** Cursor command files are plain instruction Markdown; no `$ARGUMENTS`/`!`/`@{file}` templating.

### What exists here that Claude Code doesn't

- **Cross-vendor on-disk omnivory.** Cursor natively reads `.claude/skills/`, `.codex/skills/`, `.claude/agents/`, `.codex/agents/`, `AGENTS.md`, `CLAUDE.md`, and Claude Code `settings.json` hooks, and maps Claude's hook event names and both its JSON formats to its own.
- **A dedicated Tab hook surface** (`beforeTabFileRead`, `afterTabFileEdit`) for autonomous inline completions, governed separately from agent operations.
- **A `workspaceOpen` app-lifecycle hook** that fires outside any agent session and can return additional plugin paths.
- **Shell- and MCP-specific hook events** (`beforeShellExecution`, `beforeMCPExecution`, and their `after*` pairs) whose matcher is a real content predicate over the command string.
- **`beforeReadFile`**, pre-read access control that can block a file from ever reaching the model.
- **`afterAgentThought`**, observe the agent's completed reasoning blocks.
- **`failClosed` per-hook fail-closed flag** and the `prompt`-type LLM-evaluated hook.
- **`disable-model-invocation` and `paths` skill frontmatter.** `paths` scopes a skill to matching files; nested project skill directories auto-scope without it.
- **`Auto` model routing** as the default; the underlying model is chosen per turn rather than fixed.
- **Cloud Agent handoff** (`&` prefix) and **CLI worktrees** (`--worktree`) as first-class CLI affordances.
- **ACP server mode** (`agent acp`), Cursor CLI as an Agent Client Protocol server.

---

## Known volatile (re-verify first on revisit)

- **CLI version and default model.** The CLI auto-updates on a rolling channel with no stable version pin; `Auto` is a router whose underlying model set changes as providers ship.
- **Hook event surface.** Cursor's hook system is large and actively growing (the three-surface split, `workspaceOpen`, `afterAgentThought`, and the Tab hooks all look recent). Expect new events and tightened enforcement of currently-parsed-but-ignored fields (`preToolUse` `ask`, `sessionStart` `continue`).
- **Skills/commands convergence.** `/migrate-to-skills` and the "use skills not commands" steer suggest custom commands may be deprecated; re-check command-file support on revisit.
- **Pricing.** `Auto` / `Composer` pool rates and per-model API rates change as Cursor revises plans; re-verify against the Models & Pricing page.
- **Cross-vendor compatibility paths.** The `.claude/` and `.codex/` skill/agent/hook loading is a compatibility feature that could change as the agentskills.io standard evolves.

## Known stable (low rot risk)

- **`SKILL.md` filename and `.agents/skills/` layout**, the agentskills.io open standard; externally referenced.
- **`AGENTS.md` as a memory file**, the cross-vendor `agents.md` convention.
- **`hooks.json` as the hook config file** and the four-layer Enterprise > Team > Project > User precedence.
- **The exit-code contract** (`0` use output, `2` block, otherwise fail-open) and the `failClosed` opt-in.
- **Permission token grammar** (`Shell(...)`, `Read(...)`, `Write(...)`, `WebFetch(...)`, `Mcp(...)`; `deny` overrides `allow`).
- **MCP `mcp.json` location and `mcpServers` shape**, shared with the editor and aligned with the wider MCP ecosystem.

## Open questions

- **Memory-file precedence.** Documentation does not state which wins when `AGENTS.md`, `CLAUDE.md`, and a conflicting `.cursor/rules` entry disagree.
- **Skill name-collision precedence.** Behavior when the same skill `name` appears in `.cursor/skills/`, `.agents/skills/`, and a `.claude/skills/` compatibility path is not documented.
- **CLI sandbox primitive.** The docs name `--sandbox enabled|disabled` but not the underlying OS mechanism (Seatbelt / bwrap / etc.) the CLI uses.
- **Context window / max output tokens** per model are not surfaced at the CLI level; only `preCompact` exposes a per-conversation `context_window_size`.
- **Subagent concurrency cap.** Parallel subagents are supported but no numeric `max_threads`/`max_depth`-style cap is documented.
- **Long-session degradation.** With no public issue tracker, instruction-drift and long-session behavior cannot be cited to a primary source the way the Codex/Gemini studies do.
