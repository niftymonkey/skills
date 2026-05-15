# Research prompt — OpenAI Codex CLI harness study

*Refresh recipe for [`codex.md`](./codex.md). Feed this prompt to a general-purpose research agent. Diff its output against the current `codex.md`, update changed sections, refresh the `Verified:` headers.*

---

## Use this when

- Adding the Codex study for the first time.
- Re-verifying after a major Codex CLI release (especially version bumps that add hooks, sub-agents, or change the default model).
- Calendar-based refresh — every ~6 months at minimum.
- Before relying on a specific affordance for production work.

## How to run

Spawn a `general-purpose` agent with the prompt body below as the input. Run in background; ~2 minutes typical. Compare its output to the current `codex.md` section-by-section, updating only what changed. Refresh the `Verified:` header in `codex.md` once the diff is reviewed and integrated.

If the framework in [`README.md`](./README.md) has gained new affordance categories since the last run, add the corresponding research targets to the prompt before spawning.

---

## Prompt body

Research task — design context: We maintain a personal skills library (`niftymonkey/skills`) and a cross-harness portability framework at `docs/portability/`. Each harness study is a snapshot of one harness's affordances, structured against the framework's nine categories. Your job is to research **OpenAI Codex CLI** (canonical: [`openai/codex`](https://github.com/openai/codex)) current state across all nine categories so we can author or refresh `codex.md`.

Use Ref (`mcp__ref__ref_search_documentation`), Exa via `exa-cli` Bash (`exa search`, `exa contents`, `exa answer`), and direct reads of the canonical implementation's GitHub repo. Prefer primary sources — official docs, source code, configuration schema, the repo's README and `docs/`. Avoid WebSearch unless others fail. Mark uncertain answers as `(reported, not verified)` — do not invent.

**Canonical implementation:** [`openai/codex`](https://github.com/openai/codex). Confirm this is correct; note any major active forks or competing implementations.

For each of the nine framework categories below, fill in every facet. Use `(none)` for absent affordances. Include primary-source URLs inline as `[source label](url)` for high-stakes facts: event names, tool names, model identifiers, JSON shapes, version-shipped-in numbers.

### 1. Skill loading & discovery
- Workspace skill path; user-scope skill path.
- Auto-activation mechanism (if any).
- Standards followed (agentskills.io? `.agents/skills/`? Proprietary format?).
- Frontmatter honored (`name`, `description`, `allowed-tools`, others).
- Discovery slash commands or other introspection.

### 2. Memory / context files
- Default filename (likely `AGENTS.md` per the cross-vendor convention, but verify).
- Configurable filename — yes/no, and via what mechanism.
- Scope and load order (hierarchical / global / project; precedence rules).
- Import syntax (e.g., `@./other.md`).
- Inspection / update slash commands.

### 3. Tool ecosystem
- Built-in tools, categorized table. Capture **exact tool names** — these diverge across harnesses and are a primary portability concern.
- Naming convention (CamelCase verb-style? snake_case? other?).
- Default enablement vs opt-in.
- Approval modes — list mode names.
- Sandbox affordances (macOS Seatbelt, Linux namespaces, Docker, etc.).

### 4. Sub-agents / delegation
- Available? Since which version?
- Definition format; file location.
- Parallel execution support; concurrency caveats.
- Context isolation.
- Invocation pattern.
- Background / async execution status (check open issues for partial-feature flags).
- Built-in agents shipped with the CLI, if any.

### 5. Hooks / deterministic enforcement
**This category is the highest-priority verification target** — it's the most-divergent area across harnesses and the hardest portability question for skill authors.
- Available? Since which version?
- Configuration location and shape (which settings file, which block name).
- **Full event-type list** with one-line description each. Group by lifecycle (tool / agent / session / model / misc) if applicable.
- Matcher syntax: regex vs string vs content predicate. Does the matcher see anything beyond the tool name?
- Hook input JSON shape on stdin — list all fields with types. For tool-related events specifically, capture the input fields and any tool_response shape.
- Hook output JSON shape — decision values, side-effect fields (`continue`, `systemMessage`, `suppressOutput`, etc.).
- Exit-code semantics.
- If hooks don't exist yet, note explicitly and check whether they're on the roadmap.

### 6. MCP support
- Client present? Transports (stdio / SSE / streamable HTTP).
- Configuration location and per-server fields.
- Helper CLI commands.
- Discovery slash commands.

### 7. Slash commands / explicit invocation
- File format (TOML / JSON / Markdown / other).
- Locations (global / project), namespacing rules.
- Argument injection syntax.
- Shell injection syntax.
- File injection syntax.
- Reload command.

### 8. Permission model
- Modes available, with default highlighted.
- Tool granularity (per-tool overrides? pattern allow/deny? approval prompts?).
- Elevation mechanism (model-initiated mode change?).
- Sandbox integration.

### 9. Default model & coding behavior
- Default model identifier and any routing logic.
- Context window; max output tokens.
- **Known weaknesses with primary-source citations** — upstream GitHub issue numbers especially. Look for issues describing instruction drift, hallucinated APIs, long-session degradation, or specific failure modes.
- Pricing (input / output / cache) — mark `(reported, not verified)` if from secondary aggregators.
- Enterprise lag — what version users actually get inside large orgs (if known).

### 10. Known volatile / known stable
At the end, report:
- **Known volatile** — which of the above categories or specific facts changed most recently or are most likely to change soon. Watch for: surface expansion (new events, new tools), default-model rotation, pricing changes, hook-system maturity if it's new.
- **Known stable** — which facts are unlikely to rot (standardized layouts, externally-referenced names, etc.).

### 11. Open questions
Things the research couldn't conclusively answer; flag these explicitly.

## Output format

Structured markdown matching the section layout of an existing harness study (see `gemini.md` for the template). Use tables for tool lists and event mappings. Use code blocks for JSON shapes. Target ~1500–2000 words.

Cross-harness comparison content (vs Claude Code) is **not** in scope for this prompt — that lives in `codex.md`'s "Cross-harness portability notes" section, anchored against `README.md#claude-code-reference-state`. The Claude reference state has its own refresh process.
