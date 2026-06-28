# Work Router mechanisms (Claude Code adapter edge)

The Claude-specific binding for the harness-agnostic policy in [SKILL.md](SKILL.md). This is the **only** place concrete tools are named. A new harness swaps this one file; the policy is untouched. The empty slot for the next harness is at the bottom.

## How to use this file

Reference and priors, not a lookup table or a set of lanes. The policy's job is to build or pick the mechanism that fits a unit; this file is the raw material for that: the **capability profiles** (what each primitive can and cannot do, so you can compose well), an **axis grid** of common shapes and a primitive that has fit them (priors, not a resolution you must obey), and **trigger illustrations** (signal -> reach for this -> why). Reach for the cheapest primitive that cleanly fits; compose a bespoke shape the moment a better-fitting one exists. If a shape none of these list fits the work better, build that and log it (see Calibration record) so it becomes a real prior.

## Axis grid (common shapes, as priors, not lanes)

| breadth | lifespan | substrate | cost | Mechanism | Return path | Done-contract enforcement |
|---|---|---|---|---|---|---|
| singular | inline-watchable | in-session | any | Single subagent: `Agent` tool (use the `Explore` subagent for read-only search, `Plan` for design scouting) | Summary folded inline (hygiene lane) | Brief-instructed |
| singular | long-unattended | in-session | any | Background run: `Agent` with `run_in_background: true` for a self-checking agent run, or `Bash` with `run_in_background: true` for a build / test / long command | review packet + one-line ping | Deterministic where a command or test gates it (exit 0); else brief-instructed with a declared spot-check |
| parallel | any | in-session | high | `Workflow` tool (pipeline by default; parallel only for a genuine barrier) | review packet | Per-stage brief + schema-forced structured return + an adversarial verify stage |
| parallel | any | in-session | low | A few `Agent` calls (2-4) issued in one message | review packet (write the overlay yourself) | Brief-instructed |
| recurring | on an interval | in-session | any | `/loop` skill; `ScheduleWakeup` for self-paced dynamic pacing | Ledger update or packet per cycle | The loop's exit condition is the gate |
| any | long-unattended | survives-session-end | any | **[LATER RUNG]** Scheduled cloud agent: `CronCreate` / Routines (the `schedule` skill) | Packet next session / next morning | Deterministic: success criteria + Stop discipline. Requires a sandbox and a reviewer pass |

Notes:

- **Verbose AND parallel: breadth wins** (policy rule). Route parallel; each leaf summarizes back.
- **The cost read is the downgrade lever.** It moves a `parallel / in-session` unit between the `Workflow` row (high) and the few-`Agent`-calls row (low). Magnitude framing: a single subagent runs roughly 4x the tokens of a plain chat turn and a multi-agent fan-out roughly 15x (directional, from Anthropic's research-agent measurement); concurrency caps at 16 agents at once and 1000 per run. Scope small first (pilot 2-3 units) to gauge real usage before a large fan-out.
- **The growth read is the right-size lever.** It acts on the `singular` rows: a `singular` unit read as `accumulating` and projected past its context-health budget is split at milestone seams and routed as a **relay** (a `Workflow` pipeline, each stage a fresh-context leg, journaled so a stop resumes from cached legs; or a chain of `Agent` calls each seeded by the prior leg's `continue` baton) rather than one long agent. `discardable` singular units are unaffected, and `parallel` rows already bound each leaf.
- **The survives-session-end row is the overnight rung** and is not active yet. It is listed so the bridge invariant is visible as one more row under the same policy, not a separate system.
- **Custom subagent type is a modifier on the subagent rows, not a new row.** The `singular` and low-cost `parallel` rows default to the built-in `Agent` (or `Explore` / `Plan`). When a routed lane needs its tools narrowed to an enforced allowlist or its model pinned, define a custom agent type in `.claude/agents/` (frontmatter `tools` and `model`, body as the agent's system prompt) and select it via the `Agent` tool's `agentType` (or `agent(..., {agentType})` inside a `Workflow`). It changes the agent's identity, not the unit's breadth / lifespan / substrate, so it is a lever like the cost and growth reads, applied to the same rows.

## Trigger illustrations (priors, not a menu; invent beyond these)

Concrete shapes that fit common signals, here to prime recognition, not to enumerate the options. Six spanning the range; the full trigger map lives in [DESIGN.md](DESIGN.md), and the calibration record fills in real priors as runs accrue.

- **Tangential dig that would flood context** (read-many-to-answer-one, a log or codebase search) -> a single subagent (`Explore` for read-only), returns a summary. The cheap hygiene lane.
- **Want to learn a codebase before doing it for real** -> a scout / throwaway probe: hand a subagent the hard task with no intent to keep the code, just to find which files matter, then discard it.
- **React to output as it streams** (a build, a log, a long command, waiting for an error or a status flip) -> `Monitor`, not a re-polling `/loop`.
- **Verifiable done-state, unknown effort, you want to walk away** -> `/goal`, with a *separate* check of the stop condition (maker != checker) and a hard iteration cap.
- **Too parallel or too multi-stage for one conversation, and each unit self-evaluates** (an audit / migration / sweep across many files, a finding-per-X) -> a `Workflow` (pipeline by default) with an adversarial-verify stage so results are not self-graded. Reach for it only past the fan-out gate; a subagent or a few `Agent` calls handle the small cases.
- **A one-off task whose control-flow shape no fixed pipeline matches, long or structured or adversarial enough that one context would drift** ("form competing theories and don't stop until one survives") -> author a bespoke (dynamic) `Workflow` for this task, the compose step on the detached substrate. Trigger word: `ultracode`.

Built-ins to suggest by name, since the standing failure is never mentioning them: `Monitor` (stream-react instead of re-poll), `Routines` / `CronCreate` (runs that survive the session, on a schedule or a GitHub event, the overnight substrate), the effort dial and `ultracode` (bump reasoning, or make auto-workflow-planning the session reflex for a hard stretch), and the self-repair loop (run the review gate in a loop until it returns no findings).

## The aggression dial (effort and ultracode, Claude binding)

The policy's effort-scoped aggression dial (see SKILL.md, Autonomy posture) binds to Claude's session effort setting. `/effort ultracode` is not a per-unit mechanism; it is a standing session mode (xhigh reasoning plus auto-fan-out a workflow on every substantive task). Keep the distinction straight: a dynamic `Workflow` is one task done by a swarm, whereas ultracode is the setting that makes reaching for that swarm the reflex for the whole session. Reach for it, or suggest it, when entering a sustained stretch of hard, high-value work where you would otherwise pitch a `Workflow` unit after unit; frame it as bounded (on for the stretch, dropped back after). The dial shifts where units land across the autonomy tiers; it never removes the gates or the tiers.

## Capability profiles (what each primitive can do; the new-harness contract)

Consult these to compose well and to know each primitive's limits; they are also the spec a new harness's adapter must fill. Each mechanism is described by what it can and cannot do.

- **Single subagent (`Agent` / `Explore` / `Plan`):** parallelism none; can-stop-to-ask no mid-run, but it returns to the main loop, which can then ask (interactivity is preserved one hop up); survives-session-end no; duration-fit short to medium; return inline summary; enforcement brief-instructed. It can return a "blocked at X" status instead of a forced answer.
- **Custom subagent type (`.claude/agents/*.md`, selected via `agentType`):** same execution profile as the single subagent above (no mid-run parallelism, returns to the main loop so a fork can surface one hop up, no survives-session-end), with two structural gains over a briefed generic `Agent`. **Tool-restriction** turns the brief's "use only these tools" into an enforced allowlist the agent cannot reach past, which makes the vetted-leaves invariant structural rather than instructed. **Model-pinning** runs the lane on a chosen model (a small model for a cheap hygiene lane, a larger one for a hard verify). Reach for it when a lane recurs often enough that encoding its discipline once beats re-briefing it each time, or when tool-restriction is a safety property rather than a preference; a one-off unit is cheaper as a built-in `Agent` with a good brief.
- **`Workflow` tool:** parallelism yes (up to 16 concurrent, 1000 per run); **can-stop-to-ask no** (runs detached, returns only when the whole run ends, this is the precise reason the router is not a workflow); survives-session-end no (backgrounded within the session, notifies on completion); duration-fit medium to long; return structured results plus an review packet; enforcement per-stage brief + schema + adversarial verify. Any unit that needs to surface a fork mid-run must NOT route here. How to build the workflow well (pipeline vs parallel, adversarial verify, fan-out sizing, loop-until-dry, logged coverage caps) lives in the `Workflow` tool's own description; do not duplicate it here.
- **Background run (`Agent` or `Bash` with `run_in_background: true`):** parallelism single (a couple at most); can-stop-to-ask no; survives-session-end no (tied to the session, re-invokes the loop on exit); duration-fit long; return review packet plus ping; enforcement deterministic when a command or test gates it, the cleanest long-unattended case.
- **Relay (a fresh-context `Agent` chain via `continue` batons, or a journaled `Workflow` pipeline):** parallelism none for the chain form (sequential legs), pipelined for the workflow form; can-stop-to-ask between legs in the chain form (it returns to the main loop at each baton, so a fork can surface there), none mid-run in the workflow form; survives-session-end no; duration-fit long *and* accumulating, the case a single agent cannot hold without degrading; return a baton per leg plus a final review packet; enforcement per-leg brief + the context-health budget bounding each leg + the baton as the hand-off contract. Relay, never resume: a fresh leg resets the window and drops stale context; resuming carries the bloat forward.
- **`/loop` + `ScheduleWakeup`:** recurring on an interval; survives-session-end no (an in-session interval firing back into the thread); enforcement the loop's exit condition. `ScheduleWakeup` paces a dynamic loop; pick the delay by the cache-window guidance in the tool docs.
- **Scheduled cloud agent (`CronCreate` / Routines) [later]:** parallelism varies; can-stop-to-ask no (fully unattended, so it needs maximal upfront pre-authorization); survives-session-end **yes**; duration-fit long and unattended; return packet next morning; enforcement deterministic via success criteria and Stop discipline. Needs a sandbox and a reviewer pass before it is trusted.

## Composing a bespoke shape (binding)

The policy's compose step (see [SKILL.md](SKILL.md), Composing a bespoke shape) authors a control-flow shape no single primitive provides, by wiring the profiled primitives above. The substrate is chosen by the policy's stop-to-ask rule:

- **Must surface a fork mid-run (attended composition):** a `/loop` + `ScheduleWakeup` coordinator that spawns `Agent` / background runs and pauses to the main loop at each decision point. This is the substrate behind the "heartbeat orchestrator" pattern (a coordinator that polls, spawns sub-agents, and surfaces a decision before any irreversible step). `can-stop-to-ask` is the whole reason to pick it.
- **Fully pre-authorizable (detached composition):** a `Workflow` script, the harness's general orchestration language (loops, conditionals, fan-out, pipelines, judge panels, loop-until-dry, budget-scaled fleets). Detached, so it cannot surface a fork mid-run; use it only when the shape needs no pause. Build it well per the `Workflow` tool's own description; do not duplicate that here.

Gating is concrete here: an irreversible step (a `gh pr merge`, a push, a publish) inside a composed shape is either lifted to a main-loop checkpoint or, for a merge, passed through a pre-merge gate hook (a PreToolUse hook that blocks `gh pr merge` until the review gate has run on the final commits). A composed shape never holds merge authority on its own. Fully-autonomous composed shapes that run irreversible steps unattended are the `survives-session-end` rung, not active yet.

## Live ledger (Claude binding)

The policy's "live ledger of in-flight units" binds to:

- the **harness task list** (`TaskCreate` / `TaskUpdate` / `TaskList`) for durable cross-turn tracking when several units are in flight, or
- a **lightweight in-context list** when only a unit or two are out.

Fields either way: lane / mechanism / status / where the packet lands.

## Review packet (Claude binding)

The "review packet" is the output contract defined in the companion [REVIEW-PACKET.md](REVIEW-PACKET.md) (a conversation-level overlay plus one triplet per unit).

- For a `Workflow`: emit each unit's triplet from its pipeline stage (use a schema so fields come back structured, not prose), and the overlay from a final synthesis stage.
- For a few `Agent` calls: write the overlay yourself.
- Before any merge, re-run the review gate (`coderabbit:code-review`) on the EXACT final commits. A pre-merge gate hook backstops a forgotten re-run.

## Calibration record (Claude binding)

The policy's durable cross-session calibration record (see [SKILL.md](SKILL.md), The calibration record) binds, for now, to ONE hand-maintained source: a personal calibration log you keep locally, outside this repo. It is cross-project: every rung-2 reading, for a routed run in any repo, is appended to that one file, never into per-repo memory. A project memory may carry a POINTER to it, but never a second copy of the entries: a dual-write of the global log plus a per-project copy drifts (a run logged only to the project copy goes missing from the cross-project record). That is the interim source.

The intended source is an automated feedback store that captures routed-run outcomes automatically and continuously. When it lands, swap this binding from the manual log to the store; the policy in [SKILL.md](SKILL.md) does not change, and the manual log then deletes cleanly (it concentrates nothing the store does not). Same adapter-swap discipline as the next-harness slot below.

## The next harness (empty slot, do not fill speculatively)

To bind a second harness (Codex, or another CLI), copy this file to `MECHANISMS-<harness>.md` and fill the same two structures with that harness's concrete mechanisms: the **axis grid** and the **capability profiles**. The policy in [SKILL.md](SKILL.md) does not change. Leave this slot labeled and empty until that harness is actually in use; per the seam discipline, two real adapters justify the seam (in-session and survives-session-end already do), a hypothetical third does not justify abstracting further now.
