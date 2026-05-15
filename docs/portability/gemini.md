# Gemini CLI — harness study

| | |
|---|---|
| **Harness** | [`google-gemini/gemini-cli`](https://github.com/google-gemini/gemini-cli) (Apache 2.0, npm `@google/gemini-cli`) |
| **Verified** | 2026-05-14 |
| **Harness version studied** | ~v0.27 release line; Gemini 3 default model; subagents shipped April '26 |
| **Framework** | [`README.md`](./README.md) |
| **Refresh recipe** | [`research-prompt.md`](./research-prompt.md) (general; supply preamble with harness name, URL, companion file `gemini.md`) |

A snapshot of Gemini CLI's affordances, structured against the framework's nine categories. Treat facts as authoritative only against the `Verified:` date; primary-source links inline are how to re-verify.

---

## 1. Skill loading & discovery

- **Workspace skill path:** `.gemini/skills/<name>/SKILL.md`. *(verified 2026-05-14)*
- **User-scope skill path:** `~/.gemini/skills/<name>/SKILL.md`. *(verified 2026-05-14)*
- **Auto-activation:** description-based, via the `activate_skill` tool. The model decides which skills to read from the `description:` frontmatter field. Shipped Dec '25 / Jan '26.
- **Manifest standard:** primary path is the harness-native layout above; also reads `.agents/skills/` per the [agentskills.io](https://agentskills.io/specification) standard. The `vercel-labs/skills` CLI installs cleanly via that path.
- **Frontmatter honored:** `name`, `description` required; `allowed-tools` honored.
- **Discovery commands:** `/skills` (list available).

## 2. Memory / context files

- **Default filename:** `GEMINI.md`.
- **Configurable filename:** yes — `context.fileName` in `settings.json`. Common alternate: `AGENTS.md`.
- **Scope:** hierarchical.
- **Load order:** all matching files are concatenated and sent on every prompt:
  1. Global `~/.gemini/GEMINI.md`.
  2. Ancestors of `cwd` up to the `.git` root (closer ancestors take precedence).
  3. Subdirectory `GEMINI.md` files (respects `.gitignore` / `.geminiignore`).
- **Import syntax:** `@./other.md` for modularizing memory content.
- **Inspection / update commands:** `/memory show`, `/memory refresh`, `/memory add <text>` (appends to global).

## 3. Tool ecosystem

**Built-in tools** (all enabled by default; mutators gated by confirmation):

| Category | Tools |
|---|---|
| File system | `read_file`, `read_many_files`, `write_file`, `replace`, `glob`, `grep_search`, `list_directory` |
| Shell | `run_shell_command` |
| Web | `google_web_search`, `web_fetch` |
| Interaction | `ask_user` |
| Tasking | `write_todos`, `enter_plan_mode`, `exit_plan_mode` |
| Skills | `activate_skill` |

**Tool naming convention:** snake_case with verb-noun structure (`write_file`, `run_shell_command`). Note that this diverges from Claude Code's CamelCase verbs (`Write`, `Bash`) and is a primary portability gotcha — see README rule 3.

**Default enablement:** all tools enabled; mutators (`write_file`, `replace`, `run_shell_command`) prompt for confirmation in `default` mode.

**Approval modes:** `default` / `auto_edit` / `yolo` / `plan` (read-only). `yolo` is CLI-flag-only by design (cannot be enabled via settings).

**Sandbox:** macOS-specific sandbox built in (Seatbelt profiles via `--sandbox`).

## 4. Sub-agents / delegation

- **Available:** yes, since April '26.
- **Definition format:** markdown files in `.gemini/agents/<name>.md` (project) or `~/.gemini/agents/` (global), with frontmatter (`name`, `description`, `tools`, `model`).
- **Parallel execution:** supported. Caveat: file-edit conflicts possible if multiple agents touch the same files concurrently.
- **Context isolation:** yes — each runs with curated tools / MCP in its own context window.
- **Invocation:** explicit, via `@agent-name`.
- **Background execution:** still an open issue at research time (gemini-cli [#17754](https://github.com/google-gemini/gemini-cli/issues/17754), [#14963](https://github.com/google-gemini/gemini-cli/issues/14963) — partial).
- **Built-in agents:** `generalist`, `cli_help`, `codebase_investigator`.

## 5. Hooks / deterministic enforcement

- **Available since:** v0.26.0+ (Jan '26). Source: [hooks reference](https://github.com/google-gemini/gemini-cli/blob/main/docs/hooks/reference.md).
- **Configuration location:** `hooks` block in `settings.json` with `matcher` patterns.

### Event types

| Event | Phase | Notes |
|---|---|---|
| `BeforeTool` | tool | Fires before a tool call; can `deny` or rewrite `tool_input`. |
| `AfterTool` | tool | Fires after a tool call; can replace `tool_response` sent to model. |
| `BeforeToolSelection` | tool | Filters which tools the model can see before it picks. |
| `BeforeModel` | model | Intercept / mock the LLM request itself. |
| `AfterModel` | model | Per-chunk response filter (e.g., PII redaction). |
| `BeforeAgent` | agent | Fires when user prompt is received; can `deny` or inject `additionalContext`. |
| `AfterAgent` | agent | Fires when agent run completes; can force retry via `deny`. |
| `SessionStart` | session | Matcher values: `startup` / `resume` / `clear`. |
| `SessionEnd` | session | Advisory only (cannot block). |
| `PreCompress` | session | Advisory only — unlike Claude's `PreCompact`, Gemini cannot block compression. |
| `Notification` | misc | Fires for tool-permission events only. |

### Matcher syntax

- **Tool events:** matcher is a **regex** matched against `tool_name`. *(verified 2026-05-14 via [writing-hooks](https://github.com/google-gemini/gemini-cli/blob/main/docs/hooks/writing-hooks.md))*
- **Lifecycle events:** matcher is an **exact string** against the matcher field (e.g., `startup`).
- `*` or `""` matches all.
- Multiple definitions per event allowed; `sequential: true/false` controls ordering.
- **No content predicate.** Matcher only sees `tool_name` — there is no equivalent to filtering on `tool_input.file_path` glob or `tool_input.command` substring at config-load time. File-extension and command-pattern filtering must happen **inside the shell script body**.

### Hook input JSON (stdin), tool events

```
{
  session_id, transcript_path, cwd, hook_event_name,
  timestamp,
  tool_name, tool_input,
  mcp_context, original_request_name,
  // AfterTool also includes:
  tool_response
}
```

`tool_response` is wrapped: `{ llmContent, returnDisplay, error }`.

### Hook output JSON

```
{
  decision: "allow" | "deny",       // "block" is an alias for "deny"
  reason: "...",
  hookSpecificOutput: {
    tool_input: { ... }             // MERGES/overrides fields; does NOT replace
  },
  continue: false,                  // false kills the entire agent loop
  systemMessage: "..."
}
```

- **Decision values: 2** — `allow`, `deny`. No `ask`, no `defer`.
- `hookSpecificOutput.tool_input` *merges* into the existing input.
- `additionalContext` (when returned) is appended to the tool result text.

### Exit-code semantics

- `0` — parse stdout for the decision JSON.
- `2` — block. Safest fallback when JSON output is too complex to construct.

## 6. MCP support

- **Client available:** yes — full MCP client.
- **Transports:** stdio, SSE, streamable HTTP.
- **Configuration:** `mcpServers` block in `settings.json`. Per-server fields: `command` / `url` / `httpUrl`, `env`, `trust`, `includeTools` / `excludeTools`, `timeout`.
- **Helper CLI:** `gemini mcp add | list | remove`.
- **Discovery commands:** `/mcp`, `/mcp auth`.
- MCP servers can additionally expose **prompts as slash commands**.

## 7. Slash commands / explicit invocation

- **File format:** TOML.
- **Locations:** `~/.gemini/commands/` (global) or `<project>/.gemini/commands/` (project). Subdirectories become `:` namespaces (e.g., `git/commit.toml` → `/git:commit`).
- **Argument injection:** `{{args}}` (positional).
- **Shell injection:** `!{shell-command}`.
- **File injection:** `@{file-path}`.
- **Reloadability:** in-session via `/commands reload`.

## 8. Permission model

- **Modes:** `default` / `auto_edit` / `yolo` / `plan`.
  - `default` — confirms mutator tools on each use.
  - `auto_edit` — auto-confirms edits; still prompts for shell commands.
  - `yolo` — auto-confirms everything; **CLI-flag-only**.
  - `plan` — read-only.
- **Tool granularity:** per-tool confirmation in `default` and `auto_edit`. No per-pattern allow/deny rules at the permission layer (use hooks instead).
- **Elevation:** model cannot self-elevate; user toggles modes.
- **Sandbox integration:** `--sandbox` flag enables macOS Seatbelt profiles.

## 9. Default model & coding behavior

- **Default model identifier (early '26):** `Auto (Gemini 3)`, routing between `gemini-3-pro-preview` and `gemini-3-flash-preview`. Where `gemini-3.1-pro-preview` is enabled, that's preferred. Falls back to 2.5 Pro / Flash on quota.
- **Context window:** 1,048,576 tokens (1M).
- **Max output tokens:** 65,535.
- **Known weaknesses:**
  - **Context rot / coherence degradation in long sessions** — well-documented in upstream issues: [#10975](https://github.com/google-gemini/gemini-cli/issues/10975), [#9791](https://github.com/google-gemini/gemini-cli/issues/9791), [#11185](https://github.com/google-gemini/gemini-cli/issues/11185), [#12635](https://github.com/google-gemini/gemini-cli/issues/12635), [#21792](https://github.com/google-gemini/gemini-cli/issues/21792). Real users describe it "getting dumber with each iteration" and ignoring instructions after many turns. **Recommended mitigations** (from Google's own docs): subagents (isolate context) and `/compress` (compaction).
- **Pricing** *(reported, not primary-source verified)*: 3 Pro Preview ≈ $2.00 / 1M input, $12.00 / 1M output, $0.20 / 1M cached read. Roughly 1/3 the input cost of Claude Opus 4.x and ~1/6 the output cost; comparable to Sonnet-tier pricing with a 1M context window.
- **Enterprise lag caveat:** DoD- and large-enterprise-style environments lag behind the cutting-edge release line. Design skills for the 2.5 family or older, not Gemini 3.

---

## Practical takeaways

Pointers from the README's portable-skill rules (the durable list), with Gemini-specific manifestations:

- **Rule 3 (no hard-coded tool names) is load-bearing here.** Gemini tool names diverge entirely from Claude Code's: `Edit`→`replace`, `Write`→`write_file`, `Read`→`read_file`, `Bash`→`run_shell_command`, `Glob`→`glob`, `Grep`→`grep_search`. Skills that reference tool names by string will silently misbehave.
- **Rule 6 (no `ask` UX).** Gemini hooks can only `allow` or `deny`. Patterns that need to prompt the user mid-run (Claude's `permissionDecision: "ask"`) must redesign for hard-deny with a clear `reason`, or move the decision to a `GEMINI.md` instruction.
- **Rule 7 (no content predicate in matchers).** Gemini's matcher only sees `tool_name`. Any filter on file path, command pattern, or other `tool_input` field must live inside the shell script.
- **Rule 9 (long-session behaviors).** The context-rot weakness in section 9 is real and load-bearing for shepherd-style skills. Skills that need multi-turn adherence should ship explicit `/compress` checkpoints or delegate to subagents.

---

## Cross-harness portability notes (vs Claude Code)

*Comparison anchor: Claude Code as of 2026-05-14. See [`README.md#claude-code-reference-state`](./README.md#claude-code-reference-state) for the Claude state-of-truth this section anchors against. When Claude rev's, the anchor in the README updates; this section becomes re-anchored automatically.*

### Hook event equivalents

| Gemini event | Claude equivalent | Portability |
|---|---|---|
| `BeforeTool` | `PreToolUse` | FULL |
| `AfterTool` | `PostToolUse` | FULL |
| `BeforeAgent` | `UserPromptSubmit` | FULL |
| `AfterAgent` | `Stop` | FULL |
| `SessionStart` | `SessionStart` | FULL (matcher values differ — Claude adds `compact`) |
| `SessionEnd` | `SessionEnd` | FULL (advisory in both) |
| `Notification` | `Notification` | PARTIAL (Gemini fires only on tool-permission events) |
| `PreCompress` | `PreCompact` | PARTIAL (Gemini cannot block; Claude can) |
| `BeforeToolSelection` | (none) | GEMINI-ONLY — filters available tools before model picks |
| `BeforeModel` | (none) | GEMINI-ONLY — intercept/mock the LLM request |
| `AfterModel` | (none) | GEMINI-ONLY — per-chunk response filter |
| (none) | `SubagentStop` | CLAUDE-ONLY (despite Gemini having subagents) |
| (none) | `PostCompact` | CLAUDE-ONLY |
| (none) | `FileChanged` | CLAUDE-ONLY — blocks file-mtime-watching patterns |
| (none) | `PostToolBatch` | CLAUDE-ONLY — no batch-resolution event on Gemini |
| (none) | `ConfigChange`, `CwdChanged`, `WorktreeCreate`/`Remove`, `Setup`, `TaskCreated`/`Completed`, `TeammateIdle`, `InstructionsLoaded`, `Elicitation`/`Result`, `StopFailure`, `PermissionRequest`/`Denied` | CLAUDE-ONLY (lifecycle / state-watch) |

### What doesn't exist here (that Claude Code has)

- **`ask` permission decision.** Hooks can only `allow` or `deny`. The "prompt the user to confirm before X" UX has no native equivalent. Claude has four decision values (`allow` / `deny` / `ask` / `defer`); Gemini has two.
- **`if:` content predicate on hooks.** Claude can filter on `tool_input` fields at config-load time (e.g., `if: "Edit(*.ts)"`). Gemini's matcher only sees `tool_name` — file-path / command-pattern filtering must happen inside the shell script body.
- **`FileChanged` hook.** No way to watch file mtimes natively.
- **`PostToolBatch` hook.** No batch-resolution event.
- **`SubagentStop` hook.** No subagent lifecycle hooks (despite Gemini *having* subagents).
- **Many lifecycle / state-watch events** — see table above.
- **Hard-coded memory filename.** Claude's `CLAUDE.md` is fixed; Gemini's `GEMINI.md` is the *default* but configurable via `context.fileName`. Operationally this means a `GEMINI.md` reference in a skill might point at `AGENTS.md` on a given install.

### What exists here that Claude Code doesn't

- **`BeforeToolSelection` hook.** Filter the toolbelt *before* the model picks a tool — useful for per-context tool restriction.
- **`BeforeModel` / `AfterModel` hooks.** Intercept the LLM request, or filter per-chunk responses. Useful for PII redaction, mocking, response logging.
- **Configurable memory filename** via `context.fileName` (vs Claude's hard-coded `CLAUDE.md`).
- **Toml-based custom slash commands with shell + file injection** baked in (`!{shell}`, `@{file}`) — Claude's slash commands are Markdown-based with a different injection model.

---

## Known volatile (re-verify first on revisit)

- **Event-type list.** Hooks shipped only ~5 months before this snapshot; the surface is still expanding.
- **Subagent stability and feature set.** Shipped April '26; parallelism caveats, background execution, and built-in agents are all moving.
- **Default model identifier and routing logic.** `Auto (Gemini 3)` is a routing label; the underlying preview-model names rev quickly.
- **Pricing.** Tracked via secondary aggregator; verify against Google's official pricing page on revisit.
- **`/compress` semantics.** Recommended for the context-rot mitigation but the exact compaction strategy is undocumented at primary-source level.

## Known stable (low rot risk)

- **`GEMINI.md` hierarchical load order** — likely to remain even if defaults change.
- **agentskills.io alignment** — published cross-vendor standard; unlikely to be retracted unilaterally.
- **MCP client support** — MCP is upstream of the harness; Gemini's role is consumer.
- **Tool-name divergence from CamelCase.** Renames are user-hostile; unlikely.
- **Permission-mode names (`default`/`auto_edit`/`yolo`/`plan`).** Documented and externally referenced; rename cost is real.

## Open questions

- Exact enterprise Gemini version available in our internal trial (lag depends on org).
- Whether subagents have stabilized enough to rely on for parallel fan-out by the trial start date.
- Whether the `gemini-cli` team plans to add an `ask`-equivalent decision (would close a significant cross-harness gap).
- Whether `/compress` will gain configurable compaction strategies.
