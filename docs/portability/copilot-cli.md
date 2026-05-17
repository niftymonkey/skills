# GitHub Copilot CLI - harness study

| | |
|---|---|
| **Harness** | [`github/copilot-cli`](https://github.com/github/copilot-cli) (issue tracker / feedback repo; CLI itself is closed-source, distributed as `@github/copilot` on npm and `copilot` binary) |
| **Verified** | 2026-05-17 |
| **Harness version studied** | [`v1.0.48`](https://github.com/github/copilot-cli/releases/tag/v1.0.48) (2026-05-14; latest stable at verification; `v1.0.49` pre-releases also published); default model Claude Sonnet 4.5; GA since 2026-02-25 |
| **Framework** | [`README.md`](./README.md) |
| **Refresh recipe** | [`research-prompt.md`](./research-prompt.md) (general; supply preamble with harness name, URL, companion file `copilot-cli.md`) |

A snapshot of the GitHub Copilot CLI's affordances, structured against the framework's nine categories. Treat facts as authoritative only against the `Verified:` date; primary-source links inline are how to re-verify.

**Product-surface clarification (read first).** "GitHub Copilot CLI" has named more than one product. This study covers the **current agentic GitHub Copilot CLI**: the standalone terminal coding agent distributed as the npm package [`@github/copilot`](https://github.com/github/copilot-cli) with binary `copilot`, which went generally available on [2026-02-25](https://github.blog/changelog/2026-02-25-github-copilot-cli-is-now-generally-available/). It is **not** the older `gh copilot` gh-extension (`github/gh-copilot`), which only explained or suggested shell commands and is a separate, non-agentic product. Everything below describes the `copilot` binary.

**Headline:** Copilot CLI is close to Claude Code structurally and deliberately so. Hook events ship dual-named (a Copilot camelCase name plus a Claude-Code-style PascalCase alias), the hook output JSON uses Claude's `permissionDecision: allow|deny|ask` vocabulary, and skills are loaded from `.claude/skills/` and `.agents/skills/` directly alongside Copilot's own paths. The largest divergences are that the CLI is closed-source (no source-level verification of internals), the default model is itself Anthropic's Claude Sonnet 4.5, and the permission model is built around `--allow-tool` / `--deny-tool` pattern rules rather than named modes.

---

## 1. Skill loading & discovery

- **Workspace skill path:** `.github/skills/`, `.claude/skills/`, or `.agents/skills/` within the repository (each skill is a directory containing a `SKILL.md`). Source: [Create skills for Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills).
- **User-scope skill path:** `~/.copilot/skills/` or `~/.agents/skills/`. `~/.copilot/skills/` is one of the directories listed in the [config directory reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference).
- **Admin-scope path:** `(none)` documented as a distinct skills scope; org/enterprise customization is delivered through repository instructions and the `.github-private` repo for custom agents, not a skills directory.
- **Auto-activation:** description-based. "When performing tasks, Copilot will decide when to use your skills based on your prompt and the skill's description." Explicit invocation is `/<skill-name>` in the composer (e.g., `/frontend-design`).
- **Manifest standard:** `SKILL.md` (the filename is required) with YAML frontmatter plus optional bundled assets. The accepted directory names (`.claude/skills/`, `.agents/skills/`, `~/.agents/skills/`) match the [agentskills.io](https://agentskills.io) layout in practice, so `vercel-labs/skills` installs port; GitHub's own docs describe the format without naming the agentskills.io spec.
- **Frontmatter honored:** `name` (required; lowercase, hyphenated), `description` (required; what the skill does and when to use it), `license` (optional). No documented `allowed-tools` or `disable-model-invocation` field at the skill level.
- **Scope precedence:** `(none)` explicitly documented for duplicate skill names; repository instructions are documented to take precedence over global instructions, but the skills docs do not state a duplicate-`name` merge or override rule.
- **Discovery commands:** `/skills list`, `/skills info`, `/skills reload`.

## 2. Memory / context files

- **Default filename:** `.github/copilot-instructions.md` (repository-wide). Source: [Best practices for Copilot CLI](https://docs.github.com/copilot/how-tos/copilot-cli/cli-best-practices).
- **Additional recognized files:** `AGENTS.md` (Git root or cwd), `Copilot.md`, `GEMINI.md`, `CODEX.md` (repository), and `.github/instructions/**/*.instructions.md` (modular, repository-scoped).
- **User-scope file:** `~/.copilot/copilot-instructions.md`, personal custom instructions applied to all sessions; plus `~/.copilot/instructions/` for additional personal instruction files.
- **Override filename:** `(none)`, there is no documented per-directory `.override.md` mechanism.
- **Configurable filename:** effectively yes by recognizing a fixed *set* of filenames (`AGENTS.md`, `Copilot.md`, `GEMINI.md`, `CODEX.md`), but `(none)` as a free-form user-set filename via a settings key.
- **Scope and load order:** global (`~/.copilot/copilot-instructions.md`) then repository files. **Repository instructions always take precedence over global instructions.** Multiple repository files are read and combined; the docs do not publish a strict line-precedence algorithm beyond the repo-over-global rule.
- **Size cap:** `(none)` documented; no published byte cap or settings key for instruction file size.
- **Import syntax:** `(none)`, no documented `@./`-style include directive for composing instruction files. Modularity is via the `.github/instructions/**/*.instructions.md` fan-out, not inline imports.
- **Inspection / update commands:** `/instructions` (view loaded custom instruction files); `/env` (display loaded configuration).

## 3. Tool ecosystem

The CLI is closed-source, so tool names below come from the public [Allowing tools](https://docs.github.com/en/copilot/how-tos/copilot-cli/allowing-tools) doc, not source inspection. Tools are referenced as "tool kinds" in `--allow-tool` / `--deny-tool` patterns.

| Category | Tool kinds |
|---|---|
| Shell | `shell`, `bash` |
| File read | `read`, `view`, `grep`, `glob` |
| File write | `write`, `edit` |
| Web | `web_fetch`, `web_search` |
| Skills | skill invocation (via `/<skill-name>`) |
| MCP servers | `<server-name>(<tool>)` (e.g., `MyMCP(create_issue)`) |

**Naming convention:** lowercase, snake_case for multi-word names (`web_fetch`, `web_search`). This diverges from Claude Code's CamelCase verbs (`Bash`, `Edit`, `Read`) and is a primary portability gotcha (README rule 3). The GitHub MCP server is built in.

**Default enablement:** all built-in tools are available; mutating tools (`shell`, `write`, `edit`) require approval at runtime in the default interactive flow unless pre-allowed.

**Approval / autonomy modes:** `(none)` as a set of named permission *modes* like Claude's `acceptEdits`/`plan`. Instead the CLI exposes operational modes: **Interactive** (default), **Autopilot** (autonomous continuation; `Shift+Tab` to cycle, or `--autopilot`), and **Plan mode** (`--plan`, read-only planning before autopilot). Tool gating itself is pattern-based (see section 8).

**Sandbox:** `(none)` documented as a built-in OS sandbox (no Seatbelt / bwrap+seccomp / Windows-native sandbox in the public docs). Containment is via tool allow/deny patterns and directory-scoped `write(...)` rules, not a syscall sandbox. Runs on Linux, macOS, and Windows (PowerShell and WSL).

**Protected paths:** `(none)` documented as automatically protected paths inside writable roots; restriction is opt-in via `deny-tool` patterns such as `write(.github/**)`.

## 4. Sub-agents / delegation

- **Available:** yes, "custom agents." Work performed by a custom agent runs on a **subagent**, a temporary agent with its own context window. Source: [Create custom agents for Copilot CLI](https://docs.github.com/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli).
- **Definition format:** Markdown file with the `.agent.md` extension.
- **File location:** `.github/agents/` (project) or `~/.copilot/agents/` (user). User-level agents take precedence on duplicate names.
- **Required vs optional fields:** a `name`, a description of the agent's expertise and usage triggers, and behavioral instructions in the body; an optional `tools` specification restricts the agent's tool access. The docs do not publish the exact frontmatter key list beyond this.
- **Parallel execution:** `(reported, not verified)`, the public docs describe subagent delegation without confirming concurrent fan-out. Upstream issue [#2132](https://github.com/github/copilot-cli/issues/2132) ("CLI crashes during parallel background agent execution") and [#3350](https://github.com/github/copilot-cli/issues/3350) ("Background sub-agents get stuck indefinitely") indicate parallel/background subagents exist but are unstable.
- **Context isolation:** yes, each subagent has its own context window; results are returned to the parent without cluttering main context.
- **Invocation:** four ways. `/agent` interactive picker; explicit natural-language instruction ("use the security-auditor agent on ..."); inference (Copilot auto-selects by prompt match); programmatic `copilot --agent <name> --prompt "..."`.
- **Background / async:** the `subagentStart` / `subagentStop` hook events and issues [#3350](https://github.com/github/copilot-cli/issues/3350)/[#2132](https://github.com/github/copilot-cli/issues/2132) confirm background subagent execution exists; treat as unstable.
- **Built-in agents:** `(none)` enumerated in the public docs; a "general-purpose" subagent is referenced in [#3350](https://github.com/github/copilot-cli/issues/3350).
- **Related slash command:** `/fleet` speeds up a multi-step plan via delegated work; `/delegate` creates a PR with AI-generated changes.

## 5. Hooks / deterministic enforcement

**Headline:** Copilot CLI hooks are command-hooks configured in JSON, with events dual-named, a Copilot camelCase key and a Claude-Code-style PascalCase alias. Decision vocabulary borrows Claude's `permissionDecision: allow|deny|ask`.

- **Available:** yes, configured as JSON. Source: [Use hooks](https://docs.github.com/copilot/how-tos/copilot-cli/customize-copilot/use-hooks), [Hooks reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-hooks-reference).
- **Configuration location:** repository-level hooks in `.github/hooks/*.json`; user-level in `~/.copilot/hooks/` (or `$COPILOT_HOME/hooks/`). Hooks may also be defined inline in `~/.copilot/settings.json`. File shape: `{ "version": 1, "hooks": { <event>: [ ... ] } }`. Each entry: `{ "type": "command", "bash": "...", "powershell": "...", "cwd": "...", "timeoutSec": 30, "env": { ... } }` (`bash` and/or `powershell`; `timeoutSec` default 30).

### Event types

Source: [Copilot CLI hooks reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-hooks-reference).

| Event (camelCase / PascalCase alias) | Trigger | Matcher | Decision keys |
|---|---|---|---|
| `sessionStart` / `SessionStart` | New or resumed session begins | (none) | `additionalContext` |
| `sessionEnd` / `SessionEnd` | Session terminates | (none) | advisory |
| `userPromptSubmitted` / `UserPromptSubmit` | User submits a prompt | (none) | `additionalContext` |
| `preToolUse` / `PreToolUse` | Before a tool runs | `toolName` | `permissionDecision: allow\|deny\|ask`, `permissionDecisionReason`, `modifiedArgs` |
| `postToolUse` / `PostToolUse` | After a tool completes successfully | (none) | `additionalContext` |
| `postToolUseFailure` / `PostToolUseFailure` | Tool completes with failure | (none) | `additionalContext` |
| `agentStop` / `Stop` | Main agent finishes a turn | (none) | `decision: block\|allow`, `reason` |
| `subagentStart` | Subagent spawned | `agentName` | advisory |
| `subagentStop` / `SubagentStop` | Subagent completes | (none) | `decision: block\|allow`, `reason` |
| `errorOccurred` / `ErrorOccurred` | Error during execution | (none) | advisory |
| `preCompact` / `PreCompact` | Context compaction begins | `trigger` | advisory |
| `permissionRequest` | Before the permission service runs | `toolName` | `behavior: allow\|deny`, `message`, `interrupt` |
| `notification` | CLI emits a system notification | `notification_type` | `additionalContext` |

### Matcher syntax

- **Anchored regex.** Matchers are wrapped `^(?:pattern)$` and must match the full value. Invalid regex causes the hook entry to be skipped.
- The matcher sees only one field per event: `toolName` (`preToolUse` / `permissionRequest`), `agentName` (`subagentStart`), `trigger` (`preCompact`), or `notification_type` (`notification`). **No content predicate** on `toolArgs` or other input fields at config-load time.

### Hook input JSON (stdin), common fields

```json
{
  "sessionId": "string",
  "timestamp": "number (ms)",
  "cwd": "string"
}
```

Event-specific additions: `preToolUse` adds `toolName`, `toolArgs` (typed `unknown`; note [#3349](https://github.com/github/copilot-cli/issues/3349) on safe parsing when `toolArgs` is a JSON-encoded string); `postToolUse` adds `toolResult` (`{ resultType, textResultForLlm }`); `postToolUseFailure` adds `error`; `userPromptSubmitted` adds `prompt`; `sessionStart` adds `source` (`startup|resume|new`) and optional `initialPrompt`; `sessionEnd` adds `reason` (`complete|error|abort|timeout|user_exit`); `agentStop`/`subagentStop` add `transcriptPath` and `stopReason`; `subagentStart`/`subagentStop` add `agentName`; `preCompact` adds `trigger` (`manual|auto`) and `customInstructions`; `errorOccurred` adds an `error` object plus `errorContext` and `recoverable`; `notification` adds `message`, `notification_type`, and `hook_event_name`.

### Hook output JSON

There is no single universal output block. Decision shape is per-event (see the table). For `preToolUse`:

```json
{
  "permissionDecision": "allow|deny|ask",
  "permissionDecisionReason": "string",
  "modifiedArgs": { }
}
```

For `agentStop` / `subagentStop`: `{ "decision": "block|allow", "reason": "string" }`. For `permissionRequest`: `{ "behavior": "allow|deny", "message": "string", "interrupt": boolean }`. For `postToolUseFailure` / `notification`: `{ "additionalContext": "string" }`.

### Exit-code semantics

- `0`, success; `stdout` is parsed as the JSON hook output if present.
- `2`, treated per-event: for `permissionRequest`, equivalent to `{"behavior":"deny"}`; for `postToolUseFailure`, treated as additional context.
- Other non-zero, logged as a hook failure; **execution continues (fail-open)**.
- Per-hook timeout via `timeoutSec` (default 30s).

### Trust model

`(none)` documented. The public docs do not describe a trust-prompt or `trusted_hash`-style approval step before repository-level hooks fire. Treat repository `.github/hooks/*.json` as executing without an explicit per-hook trust gate; this is an open question (see Open questions).

## 6. MCP support

- **Client present:** yes. The GitHub MCP server is built in and available with no configuration.
- **Transports:** local stdio (`type: local`), Streamable HTTP (`type: http`), and legacy SSE (`type: sse`, deprecated in the MCP spec but still supported). Source: [Add MCP servers](https://docs.github.com/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers).
- **Configuration location:** `~/.copilot/mcp-config.json` (user). Shape: an `mcpServers` object of named entries. Local entries take `command`, `args`, `env`, `tools`; HTTP/SSE entries take `url`, `headers`, `tools`. The `tools` field is `"*"` (all) or a comma-separated tool list.
- **OAuth:** `(none)` documented in the MCP config doc; HTTP servers use a `headers` object for auth.
- **Helper / discovery commands:** `/mcp add` (interactive guided form), `/mcp show`, `/mcp show <server>`, `/mcp edit <server>`, `/mcp delete <server>`, `/mcp disable <server>`, `/mcp enable <server>`.
- **Copilot-as-server:** `(none)` documented, the CLI is an MCP client; no documented mode to expose itself as an MCP server.

## 7. Slash commands / explicit invocation

- **Built-in catalog:** large. Includes `/help`, `/exit`, `/quit`, `/clear`, `/new`, `/reset`, `/resume`, `/continue`, `/session`, `/delegate`, `/review`, `/pr`, `/fleet`, `/chronicle`, `/model`, `/agent`, `/theme`, `/instructions`, `/context`, `/env`, `/diff`, `/skills`, `/mcp`, `/login`, `/feedback`, `/lsp`, `/reset-allowed-tools`. See [CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference).
- **User custom commands:** **prompt files.** Format: Markdown with the `.prompt.md` extension, located in `.github/prompts/`. These are reusable single-task templates, triggered manually. Source: [Customization cheat sheet](https://docs.github.com/en/copilot/reference/customization-cheat-sheet).
- **Argument injection:** `(reported, not verified)`, prompt files accept parameterization in the broader Copilot prompt-file format; the CLI-specific injection token is not pinned down in the CLI docs.
- **Shell injection:** `(none)` documented for prompt files (no `!{...}`-style inline shell expansion in a command file).
- **File injection:** `(none)` documented as a `@{file}`-style token inside a prompt file.
- **Reloadability:** `/skills reload` for skills. Prompt files do not document a dedicated in-session reload command.
- **Surface status:** not deprecated; skills, custom agents, and prompt files coexist as distinct customization mechanisms.

## 8. Permission model

- **Approval modes:** `(none)` as named modes (no `default`/`acceptEdits`/`plan` enum). The permission model is **pattern-based tool gating**: `--allow-tool` and `--deny-tool` take a comma-separated list of tool kinds with optional subcommand/path patterns. `--allow-all` (alias `--yolo`) permits all tools without runtime prompts. **Deny rules always take precedence over allow rules, even with `--allow-all` set.**
- **Pattern granularity:** per-tool and per-pattern. Examples: `shell` (all shell), `shell(git commit)` (exact command), `shell(git:*)` (wildcard), `write(.github/copilot-instructions.md)` (path-scoped write), `MyMCP(create_issue)` (MCP tool). Source: [Allowing tools](https://docs.github.com/en/copilot/how-tos/copilot-cli/allowing-tools).
- **Persistence:** runtime grants persist to `~/.copilot/permissions-config.json` ("tool and directory permission decisions"); grants can scope to a task, the session, or all sessions. `/reset-allowed-tools` reverts to defaults or to the flags supplied at startup.
- **Operational modes:** Interactive (default; prompts on mutating tools), Autopilot (autonomous continuation; on entry, a three-option prompt: enable all permissions / continue with limited permissions / cancel), Plan mode (read-only). `--no-ask-user` suppresses clarifying questions but is not full autonomy.
- **Elevation:** the model surfaces approval prompts to the user mid-run; on entering Autopilot the user is prompted to grant all permissions. There is no documented model-initiated permission-mode change tool.
- **Sandbox integration:** `(none)`, no OS sandbox; the permission layer is the only containment mechanism.
- **Auto-review:** `(none)` documented as an automatic reviewer-agent gate on approvals; `/review` is a user-invoked code-review command.
- **Project trust:** `(none)` documented as an untrusted-project model that suppresses repository-scoped config.
- **Managed/enterprise:** for Copilot Business and Enterprise, an administrator must enable Copilot CLI from the Policies page before it is usable.

## 9. Default model & coding behavior

- **Default model:** **Claude Sonnet 4.5** (Anthropic), per [About GitHub Copilot CLI](https://docs.github.com/en/copilot/concepts/agents/about-copilot-cli); GitHub notes the default may change. The model is selectable at startup with `--model` and mid-session with `/model`.
- **Routing:** an `Auto` option selects the best available model; otherwise a single model per session.
- **Available alternates:** Copilot CLI surfaces the broader Copilot model catalog: Claude Opus 4.6, Claude Sonnet 4.6, Claude Haiku 4.5, GPT-5 mini, GPT-5.3-Codex, GPT-5.4, GPT-5.4 mini, Gemini 3 Pro, and others (catalog varies by plan and org policy; BYOK Azure provider also supported, see [#2576](https://github.com/github/copilot-cli/issues/2576)).
- **Reasoning effort knob:** `(none)` documented as a CLI-exposed reasoning-effort setting; `CTRL+T` toggles display of model thinking.
- **Context window:** `(reported, not verified)`, not surfaced in the CLI docs. Claude Sonnet 4.5 is advertised by Anthropic at 200K tokens (1M in extended context). Issue [#3355](https://github.com/github/copilot-cli/issues/3355) notes Copilot caps Claude Opus 4.6 at 200K despite a larger model capability, indicating Copilot applies its own context caps.
- **Max output tokens:** `(reported, not verified)`, not surfaced in CLI docs.
- **Compaction:** automatic at ~95% token usage; manual via `/compact`. The `preCompact` hook fires before compaction with `trigger: manual|auto`.
- **Known weaknesses (primary-source citations, `github/copilot-cli` issue tracker):**
  - **Long-session degradation / compaction faults:** [#2500](https://github.com/github/copilot-cli/issues/2500) "Compaction killed the session", [#3054](https://github.com/github/copilot-cli/issues/3054) "Checkpoints not getting recorded after compaction", [#1614](https://github.com/github/copilot-cli/issues/1614) "Session hangs ~8 minutes after compaction when prompt cache misses", [#947](https://github.com/github/copilot-cli/issues/947) request to disable auto-compaction.
  - **Long-session memory / stability:** [#2132](https://github.com/github/copilot-cli/issues/2132) "CLI crashes during parallel background agent execution (OOM ... in long sessions)", [#3358](https://github.com/github/copilot-cli/issues/3358) "/remote toggle stops working in long-running sessions".
  - **Background subagent stalls:** [#3350](https://github.com/github/copilot-cli/issues/3350) "Background sub-agents get stuck indefinitely, never complete".
  - **Content-filter false positives on technical reasoning:** [#3348](https://github.com/github/copilot-cli/issues/3348) "Repeated 'response was blocked by content filtering' on legitimate technical reasoning turns".
  - **Cross-platform startup drift:** [#3351](https://github.com/github/copilot-cli/issues/3351) "Copilot CLI in windows fails to start without any output".
- **Pricing:** premium-request model, not per-token. Each user prompt counts as one premium request; autonomous tool calls within a turn do not. Model multipliers (paid plans): Claude Sonnet 4.5 = **1x**; Claude Haiku 4.5 about 0.33x; Claude Opus variants 3x to 15x; GPT-5.5 = 7.5x; GPT-5 mini / GPT-4.1 = 0x (included, no premium cost). Source: [Copilot premium requests](https://docs.github.com/en/copilot/concepts/billing/copilot-requests). Per-plan monthly allowances are documented separately and not pinned here. Copilot CLI requires a Copilot Pro, Pro+, Business, or Enterprise plan.
- **Enterprise lag caveat:** Business/Enterprise admins must enable Copilot CLI via the Policies page, and org policy can restrict the model catalog (e.g., to data-resident or FedRAMP-compliant models). Large orgs therefore see a narrower and slower-moving model set than consumer Pro users.

---

## Practical takeaways

Pointers from the README's portable-skill rules, with Copilot-CLI-specific manifestations:

- **Rule 1 (hooks are vendor-specific).** Copilot's hook JSON (`.github/hooks/*.json`, `{ "version": 1, "hooks": {...} }`) is a different file shape from Claude's `settings.json` `hooks` block. Ship a Copilot-specific hooks JSON snippet alongside any skill that needs hooks.
- **Rule 2 (don't extract instructions into single-harness hooks).** Copilot's event list is close to Claude's but not identical (no `FileChanged`, `PostToolBatch`, `ConfigChange`, etc.). An instruction that depends on those Claude events has no Copilot hook equivalent.
- **Rule 3 (no hard-coded tool names) is load-bearing here.** Copilot tool kinds are lowercase snake_case (`shell`, `write`, `edit`, `read`, `web_fetch`) versus Claude's CamelCase (`Bash`, `Write`, `Edit`, `Read`, `WebFetch`). Skills that reference tool names by string will misbehave. Prefer the *action* phrasing.
- **Rule 5 (no hard-coded memory filename).** Copilot's default is `.github/copilot-instructions.md`, but it also reads `AGENTS.md`, `Copilot.md`, `GEMINI.md`, and `CODEX.md`. Reference "the always-loaded memory file" generically.
- **Rule 6 (no `ask` UX).** Copilot's `preToolUse` hook *does* accept `permissionDecision: "ask"`, closer to Claude than Gemini or Codex. A skill relying on mid-run confirmation has a real Copilot path, but it is still safer to design for hard-deny since portability across all harnesses is the goal.
- **Rule 7 (no content predicate in matchers).** Copilot matchers are anchored regex against one field (`toolName`, `agentName`, etc.); they cannot see `toolArgs`. Filter inside the hook script body.
- **Rule 9 (long-session behaviors).** Copilot has documented compaction and long-session faults (see section 9). Skills needing multi-turn adherence should ship explicit `/compact` checkpoints; auto-compaction at 95% cannot be disabled today ([#947](https://github.com/github/copilot-cli/issues/947)).

---

## Cross-harness portability notes (vs Claude Code)

*Comparison anchor: 2026-05-17. Anchored against Claude Code as of 2026-05-14. See [`README.md#claude-code-reference-state`](./README.md#claude-code-reference-state) for the Claude state-of-truth this section anchors against. When Claude rev's, the anchor in the README updates; this section becomes re-anchored automatically.*

**Headline observation:** Copilot CLI is structurally close to Claude Code: hook events ship with PascalCase Claude-style aliases, the `preToolUse` decision vocabulary is Claude's (`allow|deny|ask`), and skills load directly from `.claude/skills/`. The biggest divergences are the closed-source codebase (no source-level verification), lowercase snake_case tool names, a pattern-based permission model instead of named modes, and the absence of an OS sandbox.

### Hook event equivalents

| Copilot event | Claude equivalent | Portability |
|---|---|---|
| `preToolUse` / `PreToolUse` | `PreToolUse` | FULL (same alias name; `permissionDecision: allow\|deny\|ask`; `modifiedArgs` vs Claude `updatedInput`) |
| `postToolUse` / `PostToolUse` | `PostToolUse` | FULL (same alias name) |
| `postToolUseFailure` / `PostToolUseFailure` | `PostToolUse` (failure case) | PARTIAL (Copilot splits success/failure into two events; Claude has one) |
| `userPromptSubmitted` / `UserPromptSubmit` | `UserPromptSubmit` | FULL |
| `agentStop` / `Stop` | `Stop` | FULL (`decision: block` triggers continuation) |
| `subagentStop` / `SubagentStop` | `SubagentStop` | FULL |
| `subagentStart` | (none) | COPILOT-ONLY, fires when a subagent spawns |
| `sessionStart` / `SessionStart` | `SessionStart` | FULL (source values `startup\|resume\|new`; Claude adds `compact`) |
| `sessionEnd` / `SessionEnd` | `SessionEnd` | FULL (advisory in both) |
| `preCompact` / `PreCompact` | `PreCompact` | FULL (advisory on Copilot) |
| `permissionRequest` | `PermissionRequest` | FULL (Copilot uses `behavior: allow\|deny` + `interrupt`) |
| `notification` / `Notification` | `Notification` | FULL |
| `errorOccurred` / `ErrorOccurred` | (none) | COPILOT-ONLY, fires on execution error with `recoverable` flag |
| (none) | `PostCompact` | CLAUDE-ONLY, Copilot has only `preCompact` |
| (none) | `FileChanged` | CLAUDE-ONLY, blocks file-mtime-watching patterns |
| (none) | `PostToolBatch` | CLAUDE-ONLY, no batch-resolution event |
| (none) | `ConfigChange`, `CwdChanged`, `WorktreeCreate`/`Remove`, `Setup`, `TaskCreated`/`Completed`, `TeammateIdle`, `InstructionsLoaded`, `Elicitation`/`Result`, `StopFailure`, `PermissionDenied` | CLAUDE-ONLY (lifecycle / state-watch surface) |

### What doesn't exist here (that Claude Code has)

- **`PostCompact` hook.** Copilot fires only `preCompact`; no post-compaction event.
- **`FileChanged` hook.** No native file-mtime watch.
- **`PostToolBatch` hook.** No batch-resolution event.
- **`if:` content predicate on hooks.** Copilot matchers are anchored regex on one field (`toolName`, `agentName`, etc.) and cannot filter on `toolArgs` at config-load time. Claude's `if:` predicate is Claude-only.
- **`defer` decision value.** Claude has four (`allow`/`deny`/`ask`/`defer`); Copilot's `preToolUse` has three (`allow`/`deny`/`ask`).
- **Named permission modes.** Claude has `default`/`acceptEdits`/`bypassPermissions`/`plan` as first-class modes. Copilot has operational modes (Interactive/Autopilot/Plan) but gates tools through `--allow-tool`/`--deny-tool` patterns rather than a mode enum.
- **OS sandbox.** Claude Code and Codex have syscall sandboxing; Copilot CLI documents none, containment is permission-pattern-only.
- **Open-source verifiability.** Codex and Gemini CLI are open-source; Copilot CLI is closed-source, so internal facts (exact tool list, parallelism caps) cannot be source-verified.
- **A documented hook trust gate.** Codex has an explicit `trusted_hash` workflow; Copilot's docs do not describe a per-hook trust prompt for repository `.github/hooks/*.json` (treated here as fail-open and an open question).
- **Many lifecycle / state-watch events**, see table above.

### What exists here that Claude Code doesn't

- **`subagentStart` and `errorOccurred` hooks.** Copilot fires a hook when a subagent spawns and when an execution error occurs (with a `recoverable` flag), neither has a Claude equivalent.
- **Split `postToolUse` / `postToolUseFailure`.** Copilot separates the success and failure post-tool events; Claude folds both into one `PostToolUse`.
- **Dual-named hook events.** Each event carries both a Copilot camelCase key and a Claude-style PascalCase alias, an explicit cross-harness compatibility affordance.
- **Multi-vendor model catalog under one CLI.** Copilot CLI switches between Anthropic, OpenAI, and Google models mid-session via `/model`; Claude Code runs Anthropic models only.
- **Premium-request billing.** Copilot CLI bills per prompt with model multipliers and includes zero-cost models (GPT-5 mini, GPT-4.1), a billing model Claude Code does not use.
- **Built-in GitHub MCP server.** The GitHub MCP server is bundled and active with no configuration.
- **`.github/hooks/`, `.github/agents/`, `.github/skills/` repository convention.** Customization colocates under `.github/`, the GitHub repository-config convention.
- **Reads `.claude/skills/` directly.** Copilot loads skills from `.claude/skills/` in addition to its own paths, so Claude-authored skill bundles install with no path change.

---

## Known volatile (re-verify first on revisit)

- **Default model identifier.** Claude Sonnet 4.5 today; GitHub explicitly reserves the right to change it. The Copilot model catalog rotates frequently.
- **Hook event coverage.** Hooks are recent; the event list (dual-named) and the per-event decision keys may expand. `permissionRequest` and `subagentStart` are newer entries.
- **Subagent / background-agent stability.** Background subagents stall and crash in open issues ([#3350](https://github.com/github/copilot-cli/issues/3350), [#2132](https://github.com/github/copilot-cli/issues/2132)); parallelism behavior is unverified and moving.
- **Autopilot / Plan mode surface.** Autopilot is described as experimental in parts of the docs; mode behavior may change.
- **Prompt-file injection syntax.** The CLI-specific argument/shell/file injection tokens for `.prompt.md` files are not pinned in the CLI docs.
- **Pricing / premium-request multipliers.** Multipliers and included-model lists change with the Copilot catalog; re-verify against the billing docs.
- **Version churn.** `v1.0.x` releases ship on a near-daily cadence (`v1.0.48` stable, `v1.0.49` pre-releases within days); facts can rev fast.

## Known stable (low rot risk)

- **`SKILL.md` filename** and the `.claude/skills/` / `.agents/skills/` directory layout (cross-vendor; externally referenced).
- **`.github/` repository-config convention** for hooks, agents, skills, instructions, and prompts.
- **`@github/copilot` npm package** and `copilot` binary name.
- **MCP client support**, MCP is upstream of the harness; Copilot's role is consumer, with the GitHub MCP server bundled.
- **`--allow-tool` / `--deny-tool` pattern syntax** with deny-precedence, documented and externally referenced.
- **Dual-named hook events**, the Claude-Code-style PascalCase aliases are a deliberate compatibility design and costly to break.

## Open questions

- **Hook trust model.** Do repository-level `.github/hooks/*.json` hooks fire without an explicit trust prompt? The public docs do not describe a trust gate; this should be confirmed before relying on hooks in untrusted repos.
- **Subagent parallelism caps.** Whether custom-agent subagents run concurrently and any `max_threads`/`max_depth` equivalent, undocumented; closed-source prevents source verification.
- **Context window and max output tokens** for the default Claude Sonnet 4.5 routing inside Copilot, given Copilot applies its own caps ([#3355](https://github.com/github/copilot-cli/issues/3355)).
- **Prompt-file injection tokens**, exact argument/shell/file interpolation syntax for `.prompt.md` in the CLI.
- **Whether duplicate skill names across scopes** (`.github/skills/` vs `~/.copilot/skills/`) merge or override.
- **Enterprise model lag**, the specific model-catalog delta for Business/Enterprise orgs is policy-dependent and not published as an interval.
