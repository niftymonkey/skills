# Research prompt — harness study (general)

*Refresh-or-author recipe for any `<harness>.md` in this directory. Feed this prompt to a general-purpose research agent **after prepending a short preamble** with harness-specific values.*

---

## Use this when

- Adding a new harness study (`codex.md`, `cursor.md`, etc.) for the first time.
- Re-verifying an existing harness study after a major release (new hook events, default-model rotation, sub-agent changes, etc.).
- Calendar-based refresh — every ~6 months at minimum.
- Before relying on a specific affordance for production work.

## How to run

1. **Construct a preamble** with these fields. Required for any run:
   - `Harness name:` (e.g., `Gemini CLI`, `OpenAI Codex CLI`)
   - `Canonical implementation:` (e.g., `https://github.com/google-gemini/gemini-cli`, `https://github.com/openai/codex`)
   - `Companion study file:` (e.g., `gemini.md`, `codex.md`)
   - `Distribution / package identifier:` (optional but useful — npm package, binary name, etc.)
2. **If refreshing** (vs. authoring from scratch), append:
   - `Known volatile from prior study:` followed by the bullet list from the existing study's "Known volatile" section. Focuses the agent on what's likely to have changed.
   - `Known stable from prior study:` (optional — helps the agent skip re-verifying low-rot facts).
3. Spawn a `general-purpose` agent with `<your preamble> + <the Prompt body below>` as the input. Run in background; ~2–3 minutes typical.
4. Compare the agent's output to the current `<harness>.md` section-by-section. Update only what changed. Refresh the `Verified:` header.

### Example preamble

```
Harness name: Gemini CLI
Canonical implementation: https://github.com/google-gemini/gemini-cli
Distribution: @google/gemini-cli (npm)
Companion study file: gemini.md

Known volatile from prior study:
- Hook event list (still expanding)
- Subagent stability (recently shipped)
- Default model identifier and routing logic
- Pricing (secondary-source)
```

---

## Prompt body

Research task — design context: We maintain a personal skills library (`niftymonkey/skills`) and a cross-harness portability framework at `docs/portability/`. Each harness study is a snapshot of one harness's affordances, structured against a framework of nine categories. Your job is to research the **harness named in the preamble at the URL provided in the preamble**, covering all nine categories so we can author or refresh the corresponding `<harness>.md`.

Use Ref (`mcp__ref__ref_search_documentation`), Exa via `exa-cli` Bash (`exa search`, `exa contents`, `exa answer`), and direct reads of the canonical implementation's GitHub repo (use `gh` CLI or raw GitHub URLs via `exa contents`). Prefer primary sources — official docs, source code, configuration schema, the repo's README and `docs/`. Avoid WebSearch unless others fail. Mark uncertain answers as `(reported, not verified)` — do not invent.

Confirm the canonical implementation supplied in the preamble is correct; note any major active forks or competing implementations.

For each of the nine framework categories below, fill in every facet. Use `(none)` for absent affordances. Include primary-source URLs inline as `[source label](url)` for high-stakes facts: event names, tool names, model identifiers, JSON shapes, version-shipped-in numbers.

If a "Known volatile from prior study" list is in the preamble, prioritize re-verifying those items first and flag any that have changed.

### 1. Skill loading & discovery
- Workspace skill path; user-scope skill path; admin-scope path (if any).
- Auto-activation mechanism (description matching? explicit invocation only? both?).
- Standards followed (agentskills.io? `.agents/skills/`? proprietary format?).
- Frontmatter honored (`name`, `description`, `allowed-tools`, sidecar metadata, others).
- Scope precedence rules when same skill name appears in multiple scopes.
- Discovery slash commands or other introspection.

### 2. Memory / context files
- Default filename.
- Override-filename convention (per-directory `.override.md`, etc.).
- Configurable filename — yes/no, and via what mechanism.
- Scope and load order (hierarchical / global / project; precedence rules).
- Size cap and how it's configurable.
- Import syntax (`@./`, `<include>`, etc.) — or note absence.
- Inspection / scaffold / update slash commands.

### 3. Tool ecosystem
- Built-in tools, categorized table. Capture **exact tool names** — these diverge across harnesses and are a primary portability concern. Note any matcher aliases for cross-vendor compatibility.
- Naming convention (CamelCase verb-style, snake_case, mixed).
- Default enablement vs opt-in (note feature flags).
- Approval modes — list mode names and the default.
- Sandbox affordances (macOS Seatbelt, Linux bwrap+seccomp, Windows-native, Docker, etc.).
- Protected paths inside writable roots (e.g., `.git`, `.agents`).

### 4. Sub-agents / delegation
- Available? Since which version? Stability flag if applicable.
- Definition format and file location.
- Required vs optional fields.
- Parallel execution support; concurrency caps (`max_threads`, `max_depth`, etc.).
- Context isolation.
- Invocation pattern (explicit `@name`, natural-language, both).
- Background / async execution status.
- Built-in agents shipped with the CLI.

### 5. Hooks / deterministic enforcement
**Highest-priority verification target** — the most-divergent area across harnesses and the hardest portability question for skill authors.
- Available? Since which version? Feature-flag default?
- Configuration location and shape (which settings file, which block name).
- **Full event-type list** with one-line trigger description and decision keys per event. Note any matcher aliases.
- Matcher syntax: regex vs string vs content predicate. Does the matcher see anything beyond `tool_name`?
- Hook input JSON shape on stdin — list all common fields plus event-specific fields. Capture `tool_response` shape for `PostToolUse`/`AfterTool`.
- Hook output JSON shape — universal block (`continue`, `stopReason`, `suppressOutput`, `systemMessage`), plus event-specific `hookSpecificOutput` keys. Decision values per event.
- Exit-code semantics (`0`, `2`, others).
- Trust model (do hooks need to be approved before firing?).
- Known partial coverage (events parsed but fail open, tools that aren't hookable, etc.).

### 6. MCP support
- Client present?
- Transports (stdio / SSE / streamable HTTP).
- Configuration location and per-server fields.
- OAuth support.
- Helper CLI commands.
- Discovery slash commands.
- Can the harness also act as an MCP server?

### 7. Slash commands / explicit invocation
- Built-in catalog (link to docs; don't enumerate).
- User custom commands: file format (TOML / JSON / Markdown), location, namespacing.
- Argument injection syntax.
- Shell injection syntax.
- File injection syntax.
- Reload command.
- Is the surface deprecated in favor of skills?

### 8. Permission model
- Approval modes available, with default highlighted.
- Sandbox modes available, with default highlighted.
- Named profiles or presets.
- Per-tool / per-pattern allow/deny rules.
- Network access gating.
- Elevation mechanism (model-initiated mode change?).
- Auto-review / guardian / reviewer-agent affordances.
- Project trust model — untrusted projects loading rules.
- Managed/enterprise constraints.

### 9. Default model & coding behavior
- Default model identifier and any routing logic.
- Reasoning effort or thinking-depth knobs.
- Context window; max output tokens (mark `(reported, not verified)` if not surfaced in CLI docs).
- Compaction behavior (auto vs manual, slash command).
- Available alternates.
- **Known weaknesses with primary-source citations** — upstream GitHub issue numbers. Look for issues describing instruction drift, long-session degradation, memory leaks, session-resume drift, mode-switching residue, cross-platform shell drift.
- Pricing (input / output / cache) — `(reported, not verified)` if from secondary aggregators.
- Enterprise lag — what version users actually get inside large orgs.

### 10. Known volatile / known stable
At the end, report:
- **Known volatile** — categories or specific facts changed most recently or most likely to change soon. Watch for surface expansion (new events, new tools), default-model rotation, pricing changes, sub-stable feature flags.
- **Known stable** — facts unlikely to rot (standardized layouts, externally-referenced names, etc.).

### 11. Open questions
Things the research couldn't conclusively answer.

## Output format

Structured markdown matching the section layout of an existing harness study (use `gemini.md` or `codex.md` as templates). Use tables for tool lists and event mappings. Use code blocks for JSON shapes. Target ~1500–2000 words. Use the framework section numbers (1–9) as H2 headings for easy comparison across harness studies.

Cross-harness comparison content (vs Claude Code) is **not** in scope for this prompt — that's authored separately into the harness study's "Cross-harness portability notes" section, anchored against `README.md#claude-code-reference-state`. The Claude reference state has its own refresh process.
