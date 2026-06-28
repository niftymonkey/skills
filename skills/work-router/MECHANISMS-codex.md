# Work Router mechanisms (Codex CLI adapter edge)

The Codex-specific binding for the harness-agnostic policy in [SKILL.md](SKILL.md). This is the **only** place concrete Codex tools are named; the policy is untouched. The empty slot for the next harness is at the bottom. Bound to the Codex CLI as of mid-2026 (the `rust-v0.142.x` era); Codex primitives move fast, so treat these bindings as a snapshot and refresh them when they drift (the same caveat the Claude adapter carries).

## How to use this file

Reference and priors, not a lookup table or a set of lanes. The policy's job is to build or pick the mechanism that fits a unit; this file is the raw material: the **capability profiles** (what each Codex primitive can and cannot do, so you can compose well), an **axis grid** of common shapes and a Codex primitive that has fit them (priors, not a resolution you must obey), and **trigger illustrations** (signal -> reach for this -> why). Reach for the cheapest primitive that cleanly fits; compose a bespoke shape the moment a better-fitting one exists. Where Codex has no native primitive for a routing decision, this file says so explicitly (see Gaps) rather than inventing one.

## Axis grid (common shapes, as priors, not lanes)

| breadth | lifespan | substrate | cost | Mechanism | Return path | Done-contract enforcement |
|---|---|---|---|---|---|---|
| singular | inline-watchable | in-session | any | Native subagent: spawn one (`explorer` for read-only digs, `worker` for scoped work; manage with `/agent`). Spawned only when explicitly asked | Summary folded inline (hygiene lane) | Brief-instructed |
| singular | long-unattended | in-session | any | Headless `codex exec` (alias `codex e`) for a self-checking run: `--sandbox`/`--ask-for-approval` scope its autonomy, `--json` streams events, `--output-schema` constrains the final answer | review packet + one-line ping | Deterministic where a command or test gates it (exit 0) or `--output-schema` validates; else brief-instructed with a declared spot-check |
| parallel | any | in-session | high | Native parallel subagents (spawn one per unit, wait for all, summarize; `agents.max_threads`, default 6, tunable), or `spawn_agents_on_csv` (one worker per row, with `output_schema`) | review packet | Per-agent brief + `output_schema` structured return + a separate reviewer/checker agent |
| parallel | any | in-session | low | A couple of subagent spawns in one ask, or a couple of `codex exec` processes | review packet (write the overlay yourself) | Brief-instructed |
| recurring | on an interval | app / external | any | **[APP-ONLY / EXTERNAL]** Codex app Automations or thread automations (minute / daily / weekly / cron). No CLI-binary loop exists; for headless, an external scheduler (cron, systemd timer, CI `schedule:`) wrapping `codex exec` | Ledger update or packet per cycle | The automation's success criteria, or the wrapped run's gate |
| any | long-unattended | survives-session-end | any | Codex cloud: `codex cloud exec --env <ENV_ID>` (or `@codex <task>` on a GitHub issue/PR) runs remotely in a managed container and opens a PR; `codex apply <task_id>` pulls the diff locally. Scheduling via app Automations | PR / packet next session | Deterministic: success criteria + a reviewer pass; the cloud sandbox plus a review gate before merge |

Notes:

- **Verbose AND parallel: breadth wins** (policy rule). Route parallel; each leaf summarizes back.
- **The cost read is the downgrade lever.** It moves a `parallel / in-session` unit between a full native fan-out (raise `agents.max_threads`) and a couple of spawns.
- **The growth read is the right-size lever.** A `singular` unit read as `accumulating` and projected past its context-health budget is split at milestone seams and routed as a **relay** (a `codex exec resume` chain, or sequential `codex exec` legs each seeded by an externalized baton) rather than one long run. `discardable` singular units are unaffected, and `parallel` rows already bound each leaf.
- **Fan-out width is lower than a workflow-style harness.** Codex parallel subagents default to `agents.max_threads = 6` (tunable) with `max_depth = 1` (no deep recursion). Size for that; raise the cap deliberately rather than assuming a large pool.
- **The survives-session-end row is genuinely live on Codex today** (via cloud), unlike a dormant overnight rung. What lives in the app rather than the CLI binary is the *scheduling* of those runs (Automations).
- **A custom `.toml` agent is a modifier on the subagent rows, not a new row.** The `singular` and low-cost `parallel` rows default to the built-in `explorer` / `worker` / `default`. When a routed lane needs its sandbox and approval scope narrowed or its model pinned, define a custom agent in its TOML (instructions as the system prompt, plus `model` and the `(sandbox_mode, approval_policy)` pair, both settable per custom-agent TOML) and spawn that named agent instead. It changes the agent's identity, not the unit's breadth / lifespan / substrate, so it is a lever like the cost and growth reads, applied to the same rows.

## Trigger illustrations (priors, not a menu; invent beyond these)

Concrete shapes that fit common signals, here to prime recognition. The full trigger map and the rationale live in [DESIGN.md](DESIGN.md); the calibration record fills in real priors as runs accrue.

- **Tangential dig that would flood context** (read-many-to-answer-one) -> spawn an `explorer` subagent, returns a summary. The cheap hygiene lane.
- **Want to learn a codebase before doing it for real** -> a throwaway probe: a `codex exec` or `explorer` run handed the hard task with no intent to keep the result, just to find which files matter, then discard it.
- **Self-checking run you want to walk away from** -> `codex exec --sandbox workspace-write --ask-for-approval never` against a test gate; tail `--json` if you want to watch events.
- **Verifiable done-state, unknown effort** -> `/goal <objective>` plus a `Stop` / `SubagentStop` hook that blocks-to-continue until a separate checker agent passes, OR a scripted `codex exec` loop against a test gate. There is no single built-in goal-loop; this is composed (see Gaps).
- **Too parallel or too multi-stage for one conversation, and each unit self-evaluates** -> native parallel subagents (wait-all-then-summarize) or `spawn_agents_on_csv`, with a separate reviewer agent so results are not self-graded. Reach for it only past the fan-out gate.
- **A one-off task whose control-flow shape no fixed pipeline matches** -> author a bespoke composition: a driver that spawns subagents / `codex exec` runs (attended via the TUI + a `Stop` hook, or detached via a script), the compose step.

Built-ins to suggest by name, since the standing failure is never mentioning them: `/review` (local reviewer agent on a selected diff) and `@codex review` (GitHub gate); `/agent` (spawn and manage subagents); `/goal` and `/plan`; `codex exec` (headless automation); `codex cloud` (the survives-session-end substrate) and app Automations (scheduled); the effort and aggression dials (`/fast`, `model_reasoning_effort`, `--sandbox`, `--ask-for-approval`); and lifecycle hooks (`Stop`-continuation for self-verifying loops, `PreToolUse`/`PermissionRequest` deny for a merge backstop).

## The aggression dial (Codex binding)

The policy's effort-scoped aggression dial (see SKILL.md, Autonomy posture) binds to **two pairs**:

- **Aggression = `(sandbox_mode, approval_policy)`.** Sandbox: `read-only` -> `workspace-write` -> `danger-full-access`. Approval: `untrusted` -> `on-request` -> `never`. Presets: **Read-only**, **Auto** (`workspace-write` + `on-request`, the safe middle), **Full Access**. Settable globally, per-profile, per custom-agent TOML, or per `codex exec` (`-s`/`--sandbox`, `-a`/`--ask-for-approval`). `--dangerously-bypass-approvals-and-sandbox` (`--yolo`) is the no-sandbox-no-approvals extreme: the top of the dial, used only under full pre-authorization.
- **Effort = `(model, model_reasoning_effort)`.** `model` selects the tier (the frontier default down to a fast/cheap mini for read-heavy workers, or a local model via `--oss`); `model_reasoning_effort = low | medium | high` is the thinking dial; `/fast` selects a faster service tier.

The dial shifts where units land across the autonomy tiers; it never removes the gates or the tiers. The conservative default is the **Auto** preset (`workspace-write` + `on-request`).

## Capability profiles (what each primitive can do; the new-harness contract)

Consult these to compose well and to know each primitive's limits.

- **Native subagent (`explorer` / `worker` / `default`):** parallelism yes up to `agents.max_threads` (default 6), `max_depth` default 1 (no deep nesting); can-stop-to-ask no mid-run, but it returns to the main loop, which can then ask; survives-session-end no; **spawned only on explicit request** (no silent auto-delegation); duration-fit short to medium; return a distilled summary; enforcement brief-instructed, plus a separate reviewer agent for checking. Write-heavy parallelism risks edit conflicts; prefer read-heavy fan-out.
- **Custom `.toml` agent (a named agent definition, spawned like the built-ins):** same execution profile as the native subagent above (parallel up to `agents.max_threads`, returns to the main loop so a fork can surface one hop up, no survives-session-end), with two structural gains over re-briefing a built-in. **Tool-restriction** bakes the agent's `(sandbox_mode, approval_policy)` scope into the TOML, so the limit is enforced by the harness rather than asked for in the brief, which makes the vetted-leaves invariant structural rather than instructed. **Model-pinning** sets the agent's `model` and reasoning effort, so a cheap read-heavy lane runs on a mini and a hard verify on the frontier tier. Reach for it when a lane recurs often enough that encoding its discipline once beats re-briefing it each spawn, or when sandbox-restriction is a safety property rather than a preference; a one-off unit is cheaper as a built-in with a good brief.
- **`codex exec` (headless):** parallelism single per process (script several for OS-level fan-out); can-stop-to-ask no (non-interactive; `--ask-for-approval never` for full autonomy, a less-permissive mode to pause on approvals); survives-session-end no (one invocation, though `codex exec resume --last/<id>` continues a prior run); duration-fit long; return the final message on stdout, JSONL via `--json`, structured via `--output-schema`; enforcement deterministic when a command/test gates it or `--output-schema` validates. Codex's strongest unattended primitive. Requires a git repo unless `--skip-git-repo-check`.
- **Relay (a `codex exec resume` chain, or sequential `codex exec` legs each seeded by an externalized baton):** parallelism none (sequential legs); can-stop-to-ask between legs (each returns to the driver, so a fork can surface there); survives-session-end no; duration-fit long *and* accumulating, the case one long run cannot hold without degrading; return a baton per leg plus a final packet; enforcement per-leg brief + the context-health budget bounding each leg + the baton as the hand-off contract. Relay, never resume-into-bloat: a fresh leg resets the window and carries only the baton.
- **Codex cloud (`codex cloud exec`, `@codex` on GitHub):** parallelism yes (managed containers, parallel tasks); can-stop-to-ask no (remote, so it needs maximal upfront pre-authorization); survives-session-end **yes**; duration-fit long and unattended; return a PR / packet; enforcement deterministic via success criteria and a reviewer pass, the cloud sandbox plus a review gate before merge. `codex apply <task_id>` pulls the diff locally.
- **App Automations (scheduled):** recurring on an interval or cron; survives-session-end yes (runs while the app is active, or the machine is powered for project-scoped ones); enforcement the automation's success criteria. An **app feature**, not the CLI binary.
- **Lifecycle hooks:** `SessionStart` / `UserPromptSubmit` inject context; `PreToolUse` / `PermissionRequest` can deny a command (e.g. a merge); `Stop` / `SubagentStop` can return `decision: "block"` with a continuation prompt (the self-verifying "not done yet" loop); `PreCompact` / `PostCompact` bracket compaction. Caveat from Codex's own docs: a `PreToolUse` deny is a guardrail, not a complete enforcement boundary (the model can sometimes route around an interception), so treat it as a backstop, not the sole gate.

## Composing a bespoke shape (binding)

The policy's compose step (see [SKILL.md](SKILL.md), Composing a bespoke shape) authors a control-flow shape no single primitive provides, by wiring the profiled primitives above. The substrate is chosen by the policy's stop-to-ask rule:

- **Must surface a fork mid-run (attended composition):** a driver loop in the interactive TUI (or an external script) that spawns subagents / `codex exec` runs and pauses at each decision point; a `Stop` / `SubagentStop` hook supplies the continue-or-halt control. `can-stop-to-ask` is the whole reason to pick it.
- **Fully pre-authorizable (detached composition):** a script (shell or any language) orchestrating `codex exec` invocations (loops, conditionals, fan-out, judge panels, loop-until-dry) with `--output-schema` for machine-readable hand-offs between stages, or Codex cloud for the remote substrate. Detached, so it cannot surface a fork mid-run; use it only when the shape needs no pause.

Gating is concrete here: an irreversible step (a `gh pr merge`, a push, a publish) inside a composed shape is either lifted to a driver checkpoint, or blocked by a `PreToolUse` / `PermissionRequest` hook (deny unless an explicit acknowledgment is set), or routed through `/review` / `@codex review` on the final diff first. A composed shape never holds merge authority on its own. Per the hook caveat above, keep the human checkpoint for genuinely irreversible steps until trust is earned. Fully-autonomous composed shapes that run irreversible steps unattended are the survives-session-end rung, not active yet.

## Live ledger (Codex binding)

Codex has no first-class cross-turn task-list tool. Bind the policy's "live ledger of in-flight units" to:

- a **lightweight in-context list** when only a unit or two are out, or
- a **small scratch file** in the working directory (a routing-ledger note) when several units are in flight.

Fields either way: lane / mechanism / status / where the packet lands. (Threads and `/fork` / `/side` hold parallel context but are not a status ledger.)

## Review packet (Codex binding)

The "review packet" is the output contract defined in [REVIEW-PACKET.md](REVIEW-PACKET.md) (a conversation-level overlay plus one triplet per unit).

- For native parallel subagents: have each agent return its triplet (use `output_schema`, or `spawn_agents_on_csv`'s `output_schema`, so fields come back structured), and write the overlay from a final synthesis pass.
- For a `codex exec` unit: the triplet is the structured final message (`--output-schema`); write the (small) overlay yourself.
- For a relay: the final leg emits the triplet, reading the baton trail for the decision log; write the overlay yourself.
- Before any merge, run the review gate on the EXACT final commits: `/review` locally (select the final diff) or `@codex review` on the PR. A `PreToolUse` / `PermissionRequest` hook can backstop a forgotten gate by denying the merge command unless the gate has run; `approvals_reviewer = "auto_review"` adds an automatic risk-gating reviewer.

## Calibration record (Codex binding)

The policy's durable cross-session calibration record (see [SKILL.md](SKILL.md), The calibration record) binds, for now, to ONE hand-maintained source: a personal calibration log you keep locally outside this repo, as a single cross-project file (never a per-project copy, which drifts). Every rung-2 reading, for a routed run in any repo, is appended there.

The intended source is an automated feedback store that captures routed-run outcomes continuously. When it lands, swap this binding from the manual log to the store; the policy in [SKILL.md](SKILL.md) does not change, and the manual log then deletes cleanly. Same adapter-swap discipline as the next-harness slot below.

## Gaps: routing decisions with no current native Codex mechanism (mark, do not invent)

- **React to a live stream and steer mid-flight** (the stream-reactor shape): no equivalent. Closest is parsing `codex exec --json` JSONL after the fact, or inspecting background terminals via `/ps` (read-only). A unit that needs mid-stream steering cannot route to a native primitive; keep it in the main thread or restructure it.
- **A CLI-native recurring loop** (interval / self-paced wakeup): no `codex` subcommand fires on an interval. Recurrence lives in the Codex app (Automations / thread automations) or in an external scheduler wrapping `codex exec`. In a pure-CLI/CI environment, mark recurrence as "external scheduler required."
- **A single built-in maker != checker goal-loop with an iteration cap:** not one command. `/goal` only *tracks* a target. Assemble it from `/goal` + a `Stop` / `SubagentStop` continuation hook + a separate checker agent, or a scripted `codex exec` loop against a test gate. Mark such a unit "composed," not "selected."
- **An always-firing standing reflex** (a session-loaded instruction that auto-invokes a skill every relevant turn): `AGENTS.md` is static context, not a trigger. The closest approximations are an implicit-invocation skill (`allow_implicit_invocation: true`, so it self-matches on its description) plus a `SessionStart` / `UserPromptSubmit` hook that injects the reflex each session. Treat this as **approximated, not equivalent**: the disposition that makes the router fire unprompted is weaker on Codex and must be reinforced via `AGENTS.md` prose plus hooks.

## The next harness (empty slot, do not fill speculatively)

To bind a third harness, copy this file's structure to `MECHANISMS-<harness>.md` and fill the same two structures with that harness's concrete mechanisms: the **axis grid** and the **capability profiles**. The policy in [SKILL.md](SKILL.md) does not change. Two real adapters now exist (`MECHANISMS-claude.md` and this one), which is what justifies the policy/mechanism seam; leave this slot labeled and empty until a third harness is actually in use.
