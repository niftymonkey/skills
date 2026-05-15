# OpenAI Codex CLI — harness study

| | |
|---|---|
| **Harness** | [`openai/codex`](https://github.com/openai/codex) (Apache 2.0; Rust workspace under `codex-rs/`; distributed as `@openai/codex` on npm and `codex` binary) |
| **Verified** | 2026-05-14 |
| **Harness version studied** | [`rust-v0.130.0`](https://github.com/openai/codex/releases/tag/rust-v0.130.0) (2026-05-08); `[features] hooks = true` Stable; subagents (`multi_agent`) Stable; default model `gpt-5.5` |
| **Framework** | [`README.md`](./README.md) |
| **Refresh recipe** | [`codex.research-prompt.md`](./codex.research-prompt.md) |

A snapshot of OpenAI Codex CLI's affordances, structured against the framework's nine categories. Treat facts as authoritative only against the `Verified:` date; primary-source links inline are how to re-verify.

**Headline:** Codex CLI is *substantially* Claude-Code-compatible — hook event names match exactly, MCP tool naming (`mcp__server__tool`) matches, and the wire JSON for hook output mirrors Claude's. Codex's own docs say the divergence from Claude's hook spec is intentionally minimal (just a `turn_id` extension). This makes Codex the *least* surprising port target from Claude, structurally — though tool naming for edits (`apply_patch` vs `Edit`/`Write`) is a real divergence.

---

## 1. Skill loading & discovery

- **Workspace skill path:** `$CWD/.agents/skills/` (walked from CWD up to project root); legacy `.codex/skills/` still works. Source: [Codex Skills docs](https://developers.openai.com/codex/skills).
- **User-scope skill path:** `$HOME/.agents/skills/` (current); `$CODEX_HOME/skills/` (deprecated, kept for back-compat). Admin scope: `/etc/codex/skills/` (Unix). System scope: bundled with Codex; cached to `$CODEX_HOME/skills/.system/`.
- **Auto-activation:** description-based — Codex implicitly invokes skills whose `description` matches user prompt context. Initial skill catalog is capped at ~2% of context window or 8000 chars; descriptions get shortened before whole-skill omission. Explicit invocation via `$skill-name` token in the composer.
- **Manifest standard:** [agentskills.io](https://agentskills.io) open spec — `SKILL.md` + optional `scripts/`, `references/`, `assets/`, plus a Codex-specific `agents/openai.yaml` sidecar for vendor metadata (display name, icon, brand color, default prompt, dependencies).
- **Frontmatter honored:** `name` (required, ≤64 chars), `description` (required, ≤1024 chars), `metadata.short-description` (≤1024).
- **Scope precedence:** Repo > User > System > Admin (Repo highest); duplicates by `name` are *not* merged — both appear.
- **Disable without delete:** `[[skills.config]] path = "..." enabled = false` in `config.toml`.
- **Discovery commands:** `/skills` to browse; `$` prefix in composer triggers a fuzzy picker. Built-in system skills include `skill-creator` and `skill-installer`.
- **Plugin namespacing:** plugin-bundled skills get a `plugin:skill` prefix.

## 2. Memory / context files

- **Default filename:** `AGENTS.md` ([source](https://developers.openai.com/codex/guides/agents-md)) — per the cross-vendor `agents.md` convention.
- **Override filename:** `AGENTS.override.md` — when present in a directory, takes precedence over `AGENTS.md` in that same directory.
- **Configurable extras:** `project_doc_fallback_filenames = ["TEAM_GUIDE.md", ...]` in `config.toml`.
- **Scope and load order:**
  1. Global: `$CODEX_HOME/AGENTS.override.md` else `AGENTS.md`.
  2. Project: walk from project root (Git root) down to CWD, picking one file per directory (override > AGENTS > fallbacks).
  3. Concatenate root-to-CWD, blank-line separated; nearer files override earlier ones.
- **Size cap:** `project_doc_max_bytes` default **32 KiB**; raise for larger guidance.
- **Import syntax:** `(none)` — issue [#17401 "feat: @include directive for composable AGENTS.md files"](https://github.com/openai/codex/issues/17401) is open. Composition is via the directory walk only.
- **Generate scaffold:** `/init`.
- **Reload:** no live reload — rebuilt every session start.

## 3. Tool ecosystem

Source: `codex-rs/core/src/tools/` plus [`hook_names.rs`](https://github.com/openai/codex/blob/main/codex-rs/core/src/tools/hook_names.rs).

| Tool | Hook name (canonical) | Matcher aliases | Default | Notes |
|---|---|---|---|---|
| `shell` / `unified_exec` | `Bash` | — | enabled | `unified_exec` is PTY-backed; replaced legacy `shell` |
| `apply_patch` | `apply_patch` | `Write`, `Edit` (Claude-Code compat) | enabled | Single canonical edit tool; uses a Lark grammar |
| `view_image` | (n/a) | — | enabled | Image input |
| `plan` | (n/a) | — | enabled | Plan-mode planning surface |
| `goal` | (n/a) | — | gated `features.goals` | Experimental persistent goal |
| `multi_agents` / `spawn_agents_on_csv` / `report_agent_job_result` | (n/a) | — | enabled (`features.multi_agent=true`) | Subagent fan-out |
| `request_permissions` | (n/a) | — | enabled | Model-initiated elevation |
| `request_user_input` | (n/a) | — | enabled | Model-asks-user prompt |
| `request_plugin_install` | (n/a) | — | enabled | Model can request a plugin install |
| `web_search` | (not hookable) | — | `web_search = "cached"` default | First-party tool with OpenAI-maintained cache |
| `tool_search` | (n/a) | — | enabled | Tool catalog search |
| MCP servers | `mcp__<server>__<tool>` | — | per-server | All MCP tool calls — matches Claude's naming |

**Naming convention:** Mixed. Hook-facing canonical names are `Bash` and `apply_patch` (CamelCase verb-style intentionally compatible with Claude). Internal Rust modules use snake_case. **There is no separate `Read` tool** — file reads flow through `shell` (cat/sed/etc.); Codex assumes the shell tool is the universal read path.

**Approval modes (top-level `approval_policy`):** `untrusted`, `on-request` (default), `never`. Plus a granular form: `approval_policy = { granular = { sandbox_approval, rules, mcp_elicitations, request_permissions, skill_approval } }`. Permission-mode strings exposed to hooks: `default`, `acceptEdits`, `plan`, `dontAsk`, `bypassPermissions`.

**Sandbox modes:** `read-only`, `workspace-write` (default), `danger-full-access`. Implementations:
- **macOS:** Seatbelt via `sandbox-exec -p`.
- **Linux:** `bwrap` + `seccomp` (since 0.115).
- **Windows:** native Windows sandbox crate, modes `unelevated`/`elevated`.
- **WSL2:** uses the Linux sandbox.

Protected read-only paths inside writable roots: `.git`, `.agents`, `.codex` (recursive).

## 4. Sub-agents / delegation

- **Available:** yes — `features.multi_agent = true` (Stable). See [Subagents docs](https://developers.openai.com/codex/subagents).
- **Definition format:** **TOML file per agent** under `~/.codex/agents/` (personal) or `.codex/agents/` (project). One agent per file.
- **Required fields:** `name`, `description`, `developer_instructions`.
- **Optional fields:** `nickname_candidates[]`, `model`, `model_reasoning_effort`, `sandbox_mode`, `mcp_servers`, `skills.config`, any `config.toml` key.
- **Parallel execution:** yes — Codex orchestrates spawn / wait / consolidate.
- **Concurrency caps:** `agents.max_threads` (default **6**), `agents.max_depth` (default **1** — direct children only).
- **Context isolation:** yes — each subagent has its own thread; results consolidated by parent.
- **Invocation:** natural-language only ("spawn one agent per point"); Codex spawns only when explicitly asked.
- **Built-in agents:** `default`, `worker`, `explorer`. Custom agents with same name override built-ins.
- **Background / async:** CSV-batch tool `spawn_agents_on_csv` (experimental). Per-worker timeout `agents.job_max_runtime_seconds` (default 1800s).
- **Approval inheritance:** subagents inherit parent's runtime sandbox/approval overrides (incl. `/approvals` changes and `--yolo`) — even if the agent's TOML defaults differ.
- **Slash management:** `/agent` to switch the active thread.

## 5. Hooks / deterministic enforcement

**Headline:** Codex's hook system is **Claude-Code-compatible by design.** Codex's own [hook reference](https://developers.openai.com/codex/hooks) calls out that divergence from Claude's spec is intentionally minimal — Codex adds a `turn_id` extension and uses `permission_mode` strings that include Claude's vocabulary.

- **Available since:** `[features] hooks = true` is Stable, default `true`.
- **Configuration location:** `~/.codex/config.toml` (user) or `.codex/config.toml` (project, trusted only); or `hooks.json` referenced from `config.toml`. Hook state (`[hooks.state]`) is writable only at User and SessionFlags layers.

### Event types

Source: [`schema.rs#L74-L92`](https://github.com/openai/codex/blob/main/codex-rs/hooks/src/schema.rs#L74-L92).

| Event | Trigger | Matcher applied to | Decision keys |
|---|---|---|---|
| `SessionStart` | Session begin: `startup` / `resume` / `clear` | `source` | `additionalContext` |
| `UserPromptSubmit` | User submits a prompt | (none) | `additionalContext`, `decision:block` |
| `PreToolUse` | Before tool runs | `tool_name` + matcher aliases | `permissionDecision: allow|deny|ask`, `permissionDecisionReason`, `additionalContext` |
| `PermissionRequest` | Before Codex surfaces an approval prompt | `tool_name` + aliases | `decision.behavior: allow|deny`, `decision.message` |
| `PostToolUse` | After tool produces output (incl. non-zero exit) | `tool_name` + aliases | `additionalContext`, `decision: block` |
| `PreCompact` | Before context compaction | (none — `trigger` available) | advisory |
| `PostCompact` | After compaction | (none) | advisory |
| `Stop` | About to end turn | (none) | `decision: block` triggers continuation |

### Matcher syntax

- **Regex** against the canonical `tool_name` plus its matcher aliases (e.g., `^Bash$`, `Write|Edit`, `startup|resume`).
- **Matcher does not see anything beyond `tool_name`** (except `SessionStart` which matches `source`). No content predicate on `tool_input`.

### Hook input JSON (stdin) — common fields

```json
{
  "session_id": "string",
  "turn_id": "string",
  "transcript_path": "string|null",
  "cwd": "string",
  "hook_event_name": "PreToolUse",
  "model": "string",
  "permission_mode": "default|acceptEdits|plan|dontAsk|bypassPermissions"
}
```

Event-specific additions: `PreToolUse` adds `tool_name`, `tool_input`, `tool_use_id`; `PostToolUse` adds `tool_response` (full JSON); `UserPromptSubmit` adds `prompt`; etc.

### Hook output JSON — universal

```json
{
  "continue": true,
  "stopReason": "string|null",
  "suppressOutput": false,
  "systemMessage": "string|null"
}
```

Plus per-event additions in `hookSpecificOutput` (camelCase). For `PreToolUse`: `permissionDecision: "allow"|"deny"|"ask"`, `permissionDecisionReason`, `updatedInput`, `additionalContext`. **`ask`, `updatedInput`, and `continue: false` are parsed but currently fail open** on Codex.

### Exit-code semantics

- `0` with JSON on stdout — canonical.
- `0` with plain text on stdout — accepted as `additionalContext` for `SessionStart` / `UserPromptSubmit`; ignored for tool events.
- `2` — `decision: block` shorthand for `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`.
- Default timeout: **600s**, configurable per-handler via `timeout`.

### Trust model

`/hooks` slash command browses hooks and walks the trust workflow. New / changed hooks must be trusted before they fire (`trusted_hash` state). Managed (enterprise) hooks are not user-disablable.

**Known partial coverage** (from primary docs): hooks don't intercept *all* shell calls yet (only simple ones via `unified_exec`); `WebSearch` and other non-shell, non-MCP tool calls are not hookable.

## 6. MCP support

- **Client present:** yes — `codex-rs/rmcp-client/` and `codex-rs/mcp-server/` (Codex can act as either).
- **Transports:** stdio (with env forwarding) and streamable HTTP (with bearer token + OAuth).
- **Configuration location:** `~/.codex/config.toml` (user) or `.codex/config.toml` (project, trusted only).
- **Per-server table:** `[mcp_servers.<name>]`. Fields: `command`/`args`/`env`/`env_vars` for stdio, `url`/`bearer_token_env_var`/`http_headers` for HTTP. Common: `startup_timeout_sec` (10s), `tool_timeout_sec` (60s), `enabled`, `required`, `enabled_tools`/`disabled_tools`.
- **OAuth:** `codex mcp login <server>`; configurable callback port/URL.
- **Helper CLI:** `codex mcp add`, `codex mcp` (list/manage).
- **Discovery commands:** `/mcp` (active servers); `/mcp verbose` (server diagnostics).
- **Codex-as-server:** `codex mcp-server` subcommand.

## 7. Slash commands / explicit invocation

- **Built-in catalog:** large — `/permissions`, `/agent`, `/plugins`, `/hooks`, `/mcp`, `/skills`, `/plan`, `/goal`, `/personality`, `/fork`, `/side`, `/resume`, `/new`, `/init`, `/review`, `/diff`, `/status`, `/compact`, `/model`, `/fast`, `/clear`, and more. See [slash commands doc](https://developers.openai.com/codex/cli/slash-commands).
- **User custom commands:** **deprecated** — use skills instead. Legacy: `~/.codex/prompts/*.md` (user-scope only; not repo-scoped).
- **File format (custom prompts, deprecated):** Markdown with YAML frontmatter (`description`, `argument-hint`).
- **Argument injection (custom prompts):** `$1`–`$9` positional, `$ARGUMENTS` all, `$NAMED` for `KEY=value` pairs; literal `$$` for `$`.
- **Shell injection:** `!` prefix in the composer runs a shell command and treats output as user-provided context — separate from slash commands.
- **File injection:** `/mention <path>` adds a file to context; `@` in composer triggers fuzzy file picker.
- **Reload:** restart Codex session.

Codex's preferred surface for reusable prompts is **skills**, not custom prompts.

## 8. Permission model

- **Approval modes:** `untrusted` / `on-request` (default = "Auto" preset) / `never`, plus the granular form (see section 3).
- **Sandbox modes:** `read-only` / `workspace-write` (default) / `danger-full-access` (alias `--yolo`).
- **Named profiles:** `default_permissions = ":workspace"` etc. — built-ins `:read-only`, `:workspace`, `:danger-no-sandbox`. Custom `[permissions.<name>.filesystem]` with glob patterns mapping to `"none"`/`"read"`/`"write"`.
- **Profile presets:** `[profiles.<name>]` blocks; select with `codex --profile <name>`.
- **Network access:** default off in `workspace-write`; opt-in via `[sandbox_workspace_write] network_access = true`.
- **Web search separate:** `web_search = "cached"|"live"|"disabled"` — controllable without granting full network.
- **Elevation:** model can call `request_permissions` tool; user can call `/permissions` mid-session.
- **Auto-review:** `approvals_reviewer = "auto_review"` routes eligible approvals through a reviewer agent; default `"user"`. Replaceable via `guardian_policy_config` (enterprise).
- **Project trust:** untrusted projects skip project-scoped `.codex/` layers (config, hooks, rules); user/system still load.
- **Managed/enterprise:** `requirements.toml` constrains allowed values via MDM (e.g., disallow `never` or `danger-full-access`).

## 9. Default model & coding behavior

- **Default model:** **`gpt-5.5`** (when available); fallback `gpt-5.4`. See [Codex models doc](https://developers.openai.com/codex/models). At earlier release points (2025-12-18 → 2026-04-07), default was `gpt-5.2-codex`; **the default has rotated 4+ times since GA** and is the single most-volatile fact in this study.
- **Routing:** none — single model per thread; `/model` to switch. `/fast` toggles "Fast mode" service tier (`service_tier = "fast"`) on supported models.
- **Reasoning effort knob:** `model_reasoning_effort = "minimal"|"low"|"medium"|"high"|"xhigh"` (`xhigh` added 2025-11-18).
- **Context window:** *(reported, not verified)* — not surfaced in CLI docs; GPT-5.x family typically advertised at 400k tokens via the Responses API. `/status` shows remaining capacity.
- **Max output tokens:** *(reported, not verified)*.
- **Compaction:** automatic via `PreCompact`/`PostCompact` lifecycle; `/compact` manual; `auto` trigger fires near context limit.
- **Available alternates:** `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex`, `gpt-5.3-codex-spark` (ChatGPT Pro only, research preview), `gpt-5.2`.

**Known weaknesses with primary citations:**

- **Long-session degradation:** [#10823](https://github.com/openai/codex/issues/10823) "Unable to compact context in VERY long running session", [#9546](https://github.com/openai/codex/issues/9546) "Context window explodes after long session even when auto-compacting", [#14077](https://github.com/openai/codex/issues/14077) "/review fails after long session", [#18568](https://github.com/openai/codex/issues/18568) "Long Chat = Extreme Unresponsive."
- **Memory leaks in long sessions:** [#19600](https://github.com/openai/codex/issues/19600) "135GB RAM / 25GB swap after long session", [#16828](https://github.com/openai/codex/issues/16828) "long-lived Codex session can hard-freeze host."
- **Session-resume context drift:** [#8310](https://github.com/openai/codex/issues/8310) "Session resume after rate limit loses task intent."
- **Mode-switching residue:** [#10185](https://github.com/openai/codex/issues/10185) "Mode switch Plan → Code still behaves like Plan."
- **Self-correction loops billing through usage:** [#22195](https://github.com/openai/codex/issues/22195).
- **Cross-platform shell drift:** [#13741](https://github.com/openai/codex/issues/13741) "Windows PowerShell sessions being steered by Unix/bash startup instructions."

**Pricing:** *(reported, not verified)* — see [Codex pricing](https://developers.openai.com/codex/pricing). ChatGPT subscription credits or API-key billing.

**Enterprise lag caveat:** managed configuration (`requirements.toml` + MDM) can pin specific models; large orgs typically lag the consumer default. No specific lag interval documented.

---

## Practical takeaways

Pointers from the README's portable-skill rules with Codex-specific manifestations:

- **Rule 3 (no hard-coded tool names) is partially mitigated here.** Codex's hook matcher accepts Claude's `Write`/`Edit` names as aliases for the canonical `apply_patch`; `Bash` matches as-is. Skills that say "edit the file" port cleanly. **However, the model tool surface is different** — Codex has one `apply_patch` tool, not two distinct `Edit`/`Write` tools, and no separate `Read` tool. Skills that reason about the *count* of available tools or assume a `Read` tool are wrong on Codex.
- **Rule 5 (no hard-coded memory filename).** Codex's `AGENTS.md` follows the cross-vendor `agents.md` convention. Plus `AGENTS.override.md` enables per-directory overrides Claude doesn't have. Reference *"the always-loaded memory file"* in skills, not a specific filename.
- **Rule 6 (no `ask` UX).** Codex *parses* `permissionDecision: "ask"` but **fails open** today — the discipline isn't enforced. Treat `ask` as Claude-only at runtime even though the schema accepts it.
- **Rule 7 (no content predicate in matchers) applies here too.** Codex matchers are regex against `tool_name` + aliases. Filter inside the shell script body, not in the matcher.
- **Rule 9 (long-session behaviors).** Codex has well-documented long-session weaknesses (memory leaks, context-window explosion, resume drift) — see citations in section 9. The `PreCompact`/`PostCompact` events plus `/compact` are the supported mitigations.

---

## Cross-harness portability notes (vs Claude Code)

*Comparison anchor: Claude Code as of 2026-05-14. See [`README.md#claude-code-reference-state`](./README.md#claude-code-reference-state) for the Claude state-of-truth this section anchors against. When Claude rev's, the anchor in the README updates; this section becomes re-anchored automatically.*

**Headline observation:** Codex is *substantially* Claude-Code-compatible by intentional design — much more so than Gemini. Hook event names match exactly, MCP tool naming matches, hook wire format mirrors Claude's. The biggest divergences are in the tool surface (`apply_patch` instead of `Edit`/`Write`, no separate `Read`) and in a handful of decision-value gaps where Codex *parses* Claude's vocabulary but fails open.

### Hook event equivalents

| Codex event | Claude equivalent | Portability |
|---|---|---|
| `PreToolUse` | `PreToolUse` | FULL (same name, same wire format with `turn_id` extension on Codex) |
| `PostToolUse` | `PostToolUse` | FULL (same name) |
| `UserPromptSubmit` | `UserPromptSubmit` | FULL |
| `Stop` | `Stop` | FULL |
| `SessionStart` | `SessionStart` | FULL (matcher source values: Codex has `startup`/`resume`/`clear`; Claude adds `compact`) |
| `PreCompact` | `PreCompact` | FULL (Codex is advisory; Claude can block — verify per use case) |
| `PostCompact` | `PostCompact` | FULL |
| `PermissionRequest` | `PermissionRequest` | FULL (Codex uses `decision.behavior`; Claude uses similar shape) |
| (none) | `SubagentStop` | CLAUDE-ONLY (despite Codex having subagents — no per-subagent hook) |
| (none) | `SessionEnd` | CLAUDE-ONLY |
| (none) | `Notification` | CLAUDE-ONLY |
| (none) | `FileChanged` | CLAUDE-ONLY — blocks file-mtime-watching patterns |
| (none) | `PostToolBatch` | CLAUDE-ONLY |
| (none) | `ConfigChange`, `CwdChanged`, `WorktreeCreate/Remove`, `Setup`, `TaskCreated/Completed`, `TeammateIdle`, `InstructionsLoaded`, `Elicitation`/`Result`, `StopFailure`, `PermissionDenied` | CLAUDE-ONLY (lifecycle / state-watch surface) |

### What doesn't exist here (that Claude Code has)

- **Working `ask` permission decision.** Codex's hook schema *parses* `permissionDecision: "ask"` but fails open today — equivalent of being unimplemented. Treat as Claude-only.
- **Working `updatedInput` rewrite on `PreToolUse`.** Parsed but fails open on Codex. Skills that mutate tool input via hooks (e.g., normalizing file paths) work on Claude but not Codex.
- **`continue: false` enforcement.** Parsed but fails open on Codex.
- **`suppressOutput` on `PostToolUse`.** Parsed but not yet supported.
- **`if:` content predicate on hooks.** Both lack it actually — wait, Claude *has* this. Codex doesn't. So this is a real gap: filtering by `tool_input.file_path` glob at config-load time is Claude-only.
- **`SessionEnd` hook.** Codex has only `SessionStart`.
- **`FileChanged` hook.** No way to watch file mtimes natively on Codex.
- **`SubagentStop`, `PostToolBatch`, and many lifecycle events.** See table above.
- **`defer` decision value.** Claude has 4 (`allow`/`deny`/`ask`/`defer`); Codex effectively has 2 working (`allow`/`deny`).
- **Distinct `Edit` and `Write` tools.** Codex has one `apply_patch` with `Edit`/`Write` aliases — operationally equivalent for most cases, but skills that depend on the *count* or *separation* of these tools will be off.
- **`Read` as a separate tool.** Codex routes reads through `shell` (cat / sed / etc.). Hooks that fire on file reads in Claude won't have a counterpart on Codex.

### What exists here that Claude Code doesn't

- **`AGENTS.override.md`** — per-directory override of `AGENTS.md` content. Useful for personal-machine tweaks that shouldn't be committed.
- **`turn_id` field in hook input.** Codex extension to the Claude spec; gives hooks turn-level correlation.
- **Reasoning effort knob** (`model_reasoning_effort = "minimal"..."xhigh"`).
- **TOML-based sub-agent definitions** with `model`, `model_reasoning_effort`, `sandbox_mode`, and full `config.toml` key support per agent.
- **Sandbox modes as first-class permission concept** — Linux `bwrap+seccomp`, Windows native sandbox, macOS Seatbelt, all with `workspace-write` (default) limiting writes to a project tree.
- **`workspace-write` network gate** — default-off; opt-in per profile. Network access decoupled from filesystem write.
- **`web_search` modes** (`cached`/`live`/`disabled`) — independent of general network access.
- **`approvals_reviewer = "auto_review"`** — route eligible approvals through a reviewer agent instead of the user.
- **Project trust model** — untrusted projects skip project-scoped `.codex/` layers (config, hooks, rules) automatically.
- **`spawn_agents_on_csv`** — CSV-batch subagent fan-out with structured result collection.
- **Built-in system skills** (`skill-creator`, `skill-installer`) bundled with the CLI.
- **`request_permissions`, `request_user_input`, `request_plugin_install` tools** — model-initiated UX flows Claude doesn't expose as discrete tools.

---

## Known volatile (re-verify first on revisit)

- **Default model identifier.** Rotated 4+ times since GA (`gpt-5-codex` → `gpt-5.1-codex` → `gpt-5.1-codex-max` → `gpt-5.2-codex` → `gpt-5.4`/`5.5`). Expect another rotation within 3–6 months.
- **Hook event coverage gaps.** `WebSearch` not hookable; complex shell flows partially intercepted; `ask` / `updatedInput` / `suppressOutput` parsed-but-fail-open. These will likely close over time.
- **Sub-agent system.** `multi_agents_v2.rs` coexists with `multi_agents.rs`; CSV batch tooling marked experimental; format may evolve.
- **Feature flags table.** `apps`, `codex_git_commit`, `goals`, `plugin_hooks`, `memories`, `undo` all sub-stable; expect flips.
- **Plugin system + plugin-bundled hooks** (`plugin_hooks = "Under development"`).
- **Auto-review** (`approvals_reviewer = "auto_review"`) — new in 2026-04.
- **Pricing.** Tracked via secondary aggregator and ChatGPT subscription credits; verify against [pricing page](https://developers.openai.com/codex/pricing) on revisit.

## Known stable (low rot risk)

- **`AGENTS.md` filename** and directory-walk discovery (externally referenced `agents.md` standard).
- **`SKILL.md` filename** and `.agents/skills/` layout (agentskills.io spec; externally referenced).
- **`mcp__server__tool`** MCP tool-naming convention (Claude Code interop; widely adopted).
- **Hook event names** (`PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `SessionStart`, `PreCompact`, `PostCompact`, `PermissionRequest`). Claude-Code compatibility is an explicit design goal; would be costly to break.
- **TOML configuration** with `~/.codex/config.toml` user + `.codex/config.toml` project (trusted-only) layering.
- **`apply_patch` as the canonical edit tool** with `Write`/`Edit` matcher aliases.
- **Sandbox primitives:** Seatbelt / bwrap+seccomp / Windows-native.

---

## Open questions

- **Exact hook coverage for `unified_exec`** — docs say "incomplete" but don't enumerate.
- **Hook stdin encoding for MCP `tool_input`** — server-dependent; not standardized.
- **`features.hooks` default-true ship date** — Stable now but no changelog entry pinpoints when the default flipped.
- **Custom-agent file precedence** when same `name` appears in both `~/.codex/agents/` and `.codex/agents/`.
- **Context window and max output tokens** for `gpt-5.5` / `gpt-5.4-codex` — not surfaced in Codex CLI docs.
- **`updatedInput` rewrite capability** — parsed but fails open; roadmap timing unclear.
- **`plugin_hooks` maturity** — "Under development" with no public roadmap date.
