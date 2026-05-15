# Research prompt — Gemini CLI harness study

*Refresh recipe for [`gemini.md`](./gemini.md). Feed this prompt to a general-purpose research agent. Diff its output against the current `gemini.md`, update changed sections, refresh the `Verified:` headers.*

---

## Use this when

- Adding the Gemini study for the first time (already done — see git history).
- Re-verifying after a major Gemini CLI release (especially v0.x major bumps, new model defaults, new hook events).
- Calendar-based refresh — every ~6 months at minimum.
- Before relying on a specific affordance for production work.

## How to run

Spawn a `general-purpose` agent with the prompt body below as the input. Run in background; ~2 minutes typical. Compare its output to the current `gemini.md` section-by-section, updating only what changed. Refresh the `Verified:` header in `gemini.md` once the diff is reviewed and integrated.

If the framework in [`README.md`](./README.md) has gained new affordance categories since the last run, add the corresponding research targets to the prompt before spawning.

---

## Prompt body

Research task — design context: We maintain a personal skills library (`niftymonkey/skills`) and a cross-harness portability framework at `docs/portability/`. Each harness study is a snapshot of one harness's affordances, structured against the framework's nine categories. Your job is to research Gemini CLI's current state across all nine categories so we can refresh `gemini.md`.

Use Ref (`mcp__ref__ref_search_documentation`), Exa via `exa-cli` Bash (`exa search`, `exa contents`, `exa answer`), and direct reads of the canonical implementation's GitHub repo (`google-gemini/gemini-cli`). Prefer primary sources — official docs, source code, configuration schema. Avoid WebSearch unless others fail. Mark uncertain answers as `(reported, not verified)` — do not invent.

For each of the nine framework categories below, fill in every facet. Use `(none)` for absent affordances. Include primary-source URLs inline as `[source label](url)` for high-stakes facts: event names, tool names, model identifiers, JSON shapes, version-shipped-in numbers.

**Canonical implementation:** [`google-gemini/gemini-cli`](https://github.com/google-gemini/gemini-cli) (Apache 2.0, npm `@google/gemini-cli`).

### 1. Skill loading & discovery
- Workspace skill path; user-scope skill path.
- Auto-activation mechanism (`activate_skill` tool, description matching).
- Standards followed (agentskills.io? `.agents/skills/`?).
- Frontmatter honored (`name`, `description`, `allowed-tools`, others).
- Discovery slash commands.

### 2. Memory / context files
- Default filename; configurable via what setting.
- Scope and load order (hierarchical / global / project; precedence rules).
- Import syntax (`@./` etc.).
- Inspection / update slash commands.

### 3. Tool ecosystem
- Built-in tools, categorized table. Capture **exact tool names** — these diverge across harnesses and are a primary portability concern.
- Default enablement vs opt-in.
- Approval modes (`default`, `auto_edit`, `yolo`, `plan`).
- Sandbox affordances (macOS Seatbelt, etc.).

### 4. Sub-agents / delegation
- Available since? Definition format; file location.
- Parallel execution support; concurrency caveats.
- Context isolation.
- Invocation pattern (`@agent-name` etc.).
- Background / async execution status (check open issues).
- Built-in agents shipped with the CLI.

### 5. Hooks / deterministic enforcement
This category is the highest-leverage to verify because the surface is still expanding rapidly.
- Available since (version).
- Configuration location and shape (`settings.json` block name).
- **Full event-type list** with one-line description each. Group by lifecycle (tool / agent / session / model / misc).
- Matcher syntax: regex vs string vs content predicate. Does the matcher see anything beyond `tool_name`?
- Hook input JSON shape on stdin — list all fields with types. For `BeforeTool` and `AfterTool` specifically, capture the `tool_response` shape too.
- Hook output JSON shape — decision values, `hookSpecificOutput`, side-effect fields (`continue`, `systemMessage`, `suppressOutput`, `additionalContext`).
- Exit-code semantics.

### 6. MCP support
- Client present? Transports (stdio / SSE / streamable HTTP).
- Configuration location and per-server fields.
- Helper CLI commands.
- Discovery slash commands.

### 7. Slash commands / explicit invocation
- File format (TOML).
- Locations (global / project), namespacing rules.
- Argument injection syntax (`{{args}}`).
- Shell injection syntax (`!{...}`).
- File injection syntax (`@{...}`).
- Reload command.

### 8. Permission model
- Modes available, with default highlighted.
- Tool granularity (per-tool overrides? pattern allow/deny?).
- Elevation mechanism (model-initiated mode change?).
- Sandbox integration.

### 9. Default model & coding behavior
- Default model identifier and any routing logic (`Auto (Gemini 3)` etc.).
- Context window; max output tokens.
- **Known weaknesses with primary-source citations** — upstream GitHub issue numbers especially. The "context rot in long sessions" issue cluster is the canonical one; check whether it's still active or has been mitigated.
- Pricing (input / output / cache) — mark `(reported, not verified)` if from secondary aggregators.
- Enterprise lag — what version users actually get inside large orgs.

### 10. Known volatile / known stable
At the end, report:
- **Known volatile** — which of the above categories or specific facts changed most recently or are most likely to change soon. Watch for: surface expansion (new events, new tools), default-model rotation, pricing changes.
- **Known stable** — which facts are unlikely to rot (standardized layouts, externally-referenced names, etc.).

### 11. Open questions
Things the research couldn't conclusively answer; flag these explicitly.

## Output format

Structured markdown matching the section layout of the current `gemini.md`. Use tables for tool lists and event mappings. Use code blocks for JSON shapes. Target ~1500–2000 words.

Cross-harness comparison content (vs Claude Code) is **not** in scope for this prompt — that lives in `gemini.md`'s "Cross-harness portability notes" section, anchored against `README.md#claude-code-reference-state`. The Claude reference state has its own refresh process.
