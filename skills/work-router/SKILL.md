---
name: work-router
description: Use when deciding, for a unit of work (an assignment with a definable done-state), whether to keep it in the main thread or route it off-thread, and by which mechanism. Fires at plan/decomposition time over each unit of the work-list, and reactively when a sub-task reveals itself as routable. Produces a keep-or-route decision carrying lane, mechanism, autonomy posture, done-contract, and return path. Triggers on "should I delegate this," decomposing a plan into units, a verbose/parallel/long-running sub-task surfacing mid-work, and any "keep this in the conversation or push it off-thread" moment.
---

# Work Router (policy core)

The standing disposition that decides, for each unit of work, whether it stays with you in the main thread or gets routed off-thread, and by what mechanism. This file is the portable policy: it holds for any agent CLI and any model and names no harness or tool. The per-harness binding (which concrete mechanism each routing decision resolves to) is the adapter edge: read the mechanisms file matching your harness ([MECHANISMS-claude.md](MECHANISMS-claude.md) for Claude Code, [MECHANISMS-codex.md](MECHANISMS-codex.md) for Codex). The full reasoning, the eleven resolved decisions, and what was rejected live in [DESIGN.md](DESIGN.md); do not re-derive them here.

## Core idea (one line)

Only HITL work stays in the main thread; everything else is routed off-thread by whatever mechanism fits its shape. Action by default, always reviewable, always reversible-or-gated. When a call is genuinely ambiguous, that sentence is the tie-breaker.

## This is a disposition, not a workflow

The router runs live in the main loop on every assignment, so it stays always-on, reactive, and able to stop and ask when unsure. A detached batch job could not do that. A parallel-breadth fan-out is one mechanism the router dispatches to, never the router itself; an authored composition is likewise a dispatched mechanism, never the router, the router decides interactively and the composed shape executes. Do not rebuild this as a thing that runs detached.

## The interface (what one routing decision returns)

Given an assignment, return one of:

- **keep** -- the unit stays in the main thread (Gate 1 kept it).
- **route(lane, mechanism, posture, done-contract, return-path)** -- the unit goes off-thread, plus a live-ledger entry.

Invariants this interface holds to:

- It names no harness or tool. The `mechanism` slot is a harness-agnostic *shape key*; the adapter edge (your harness's mechanisms file) resolves that key to a concrete tool. A new harness adds its own.
- Every `route` produces the three artifacts the overnight rung consumes (see The bridge invariant). In-session and overnight are the same policy, different rows in the table.
- It never routes a unit whose definition is still actively churning (see Presence and settledness).
- Its default leans to action (see Autonomy posture).

## When it fires

Two moments:

- **At plan / decomposition time:** classify each unit of the work-list.
- **Reactively, mid-work:** a sub-task that reveals itself as routable.

A **unit** is an assignment: a coherent sub-task with a definable done-state. Not a whole prompt, not a single tool call.

Canonical flow: **scout inline, then route.** The inline scout is the ground-truth baseline that makes delegated output checkable, and it is itself the triage step the overnight rung consumes. Do not delegate what you have not first scoped enough to check.

## Gate 1: keep vs route

Keep in the main thread if the unit is any of:

- a cascading design fork (defining it changes later steps),
- irreversible or outward-facing (a push, a merge, a publish, anything hard to take back),
- a tight back-and-forth (you would be sampled many times),
- the final synthesis or decision.

Otherwise, route it.

## Gate 2: mechanism by shape (routed work only)

Read the unit's shape along these dimensions; they inform which mechanism fits and are not a lookup key:

- **breadth:** `singular` (one coherent answer from much input) or `parallel` (many independent units or angles).
- **lifespan:** `inline-watchable` (short, you would watch it) or `long-unattended` (runs a while, no need to watch).
- **substrate:** `in-session` (returns within this session) or `survives-session-end` (must outlive the session; the overnight rung).
- **cost read (modifier):** magnitude that can downgrade a `parallel` unit from a full fan-out to a few agents.
- **growth read (modifier):** `discardable` (each tool result is consumed and forgotten, so the resident window stays flat however long the unit runs) or `accumulating` (the unit keeps state resident, so the window grows with progress). Orthogonal to lifespan. A lever like the cost read, not a selector: it can right-size a `singular` unit into a relay (see Right-sizing, below).

Then build or pick the mechanism that fits. Reach for the cheapest primitive that cleanly fits (laziness, not because composition is inferior), and compose a bespoke shape the moment a better-fitting one exists (see Composing a bespoke shape). The axis grid, capability profiles, and trigger illustrations in your harness's mechanisms file are **reference and priors** for this, what each primitive can do and shapes that have fit before, not a lookup to resolve against or lanes to stay in. If a shape the priors do not list fits the work better, build that, and log it so it becomes a real prior (see The calibration record).

One steady rule: **a unit that is both verbose and parallel, breadth wins.** Route it parallel; the verbosity just means each leaf summarizes back.

**Right-sizing accumulating units.** Only `accumulating` units can degrade as their window grows; `discardable` units route as-is however long they run (the common case, so this is usually a no-op). For an `accumulating` unit:

- projected under its context-health budget -> single agent, checkpoint protocol armed as a safety net;
- projected over budget -> split at natural milestone seams into legs that each finish in-zone, routed as a relay (a journaled `Workflow` pipeline, or an `Agent` chain handing off a `continue` baton).

The seam is the trigger; the budget is the quality-knee dial that confirms a relay is due, never a hard cliff. **Relay, not resume:** a relay resets to a fresh leg carrying only the baton (deliberate context GC); resuming carries the bloat forward and defeats the purpose.

## The fan-out gate (parallelism's real ceiling)

Routing off-thread is the default, but *parallel fan-out* specifically has one precondition, because the ceiling on parallel agent work is human verification, not generation. Fan out (a workflow, a judge panel, many agents at once) only when both hold: the units are **independent** (they do not share mutable state), and each has a **tight self-evaluation** (a test, a rubric, a review bot) so its result is trustworthy without you reading all of it. When units share state or all funnel back to one human reviewer, more parallelism just makes more for that reviewer, and it backfires. Sizing follows: a few focused workers (roughly 3-5) beat a scattered swarm, and validation caps at one. This gate is the counterweight to the action-by-default lean. It does not apply to the cheap hygiene lane (a single subagent absorbing verbose work), only to genuine fan-out.

## Composing a bespoke shape

Composition is first-class, not a last resort. Build a shape that fits the work whenever no single primitive cleanly does (a stateful multi-actor loop, a poll-fix-until-converged cycle, a spawn-and-gate sequence), rather than forcing the unit into an ill-fitting primitive or hand-running it in the main thread (the bleed work-router exists to stop). Composition is a property of the mechanism slot, not of the router: the decision to compose is main-loop and interactive, the execution is dispatched. Five invariants keep an authored shape reviewable and reversible-or-gated:

- **Lazy-first, not last.** Use the cheapest primitive that cleanly fits, and compose the moment a better-fitting shape exists. Both are first-class; the only bias is against over-building, never an authored state machine where a subagent does the job.
- **Vetted leaves, novel wiring.** A composed shape wires known capability-profiled primitives; the novelty is the control flow, not new I/O. Review crosses the wiring; the leaves are already trusted.
- **Irreversible steps gated, never embedded.** Every outward-facing step inside a composed shape is either lifted to a main-loop checkpoint (the shape pauses, returns a packet, waits) or routed through the deterministic merge gate (see [REVIEW-PACKET.md](REVIEW-PACKET.md)). A composed shape never merges, publishes, or pushes on its own authority.
- **Stop-to-ask selects the substrate.** A shape that must surface a fork mid-run is built on a coordinator that can pause; a fully pre-authorizable shape is built detached. The binding (which primitive each is) is the adapter edge.
- **Same contract, same packet.** A composed shape still carries the done-contract (floor / ceiling / budget) and still returns a review packet. The bridge invariant holds: an authored shape is still classification + injected discipline + packet.

Autonomy posture for *launching* a composed shape is conservative: announce-and-go at most, propose-and-wait if the shape contains any gated irreversible step. A fully-autonomous composed shape (one that runs irreversible steps unattended) is the trust-ladder rung-5 form (see [REVIEW-PACKET.md](REVIEW-PACKET.md)); per the 2026-06-19 calibration we are at rung 1-3, so not yet.

## Autonomy posture (three tiers, default leans to action)

Keyed to reversibility x classification-confidence x cost:

- **Silent** (do it, mention in passing): read-only, cheap, high-confidence. Nothing to veto.
- **Announce-and-go** (fire, tell immediately, redirectable, non-blocking): bigger-but-revertable, or anything that costs real tokens. The broad safe middle. Inside this tier, the announcement *is* the agreement; this is the actual inversion of "never silently escalate."
- **Propose-and-wait** (stop, ask): hard-to-revert or outward-facing (usually already kept by Gate 1), or low confidence about whether the user would want to steer. The live trigger here is uncertainty.

The effort-scoped aggression dial (a per-effort directive to maximize delegation) shifts where units land between these tiers; it never removes the tiers.

## Presence and settledness

Presence is neutral to routing. Same reflex whether the user is watching or away. Presence never throttles, never adds ceremony, never makes the router ask more. It changes only the **channel**: real-time veto and live narration when present; async notification and more upfront pre-authorization when away.

The genuine hold-off signal is **settledness**, not presence. Do not route a unit whose definition is still actively churning. If fuzzy definition is the *sole* blocker, push to define rather than parking it.

## Push-to-define (when fuzzy definition is the sole blocker)

Drive the unit to delegatability on a cost-ordered ladder, so it never nags:

1. **Scout** to resolve the gap silently.
2. **Resolve by stated default** (and say so).
3. **Ask** only the load-bearing forks scouting cannot settle.

Boundary: this applies when the gap is *clarification*. When the gap is genuine hard design that cascades, defining it IS the HITL work, and Gate 1 keeps it. Push-to-define scaled up over a backlog is the clean pre-triage the overnight rung runs on.

## The done-contract (first-class output of every routing decision)

Set when the unit is born, always before the agent starts, always carried in the brief. A two-sided bound:

- **Floor (do not stop short):** a finite acceptance checklist plus a verification gate that proves it. Done = checklist met AND gate green, never self-asserted. If the gate cannot be reached, the agent stops and reports "blocked at X" rather than declaring done.
- **Ceiling (do not spin on "one more thing"):** done is the checklist plus a green gate, NOT the absence of things to improve. Everything found outside the checklist goes to the findings-sink; it is not acted on.
- **Budget / iteration + context cap:** a unit that cannot converge stops and reports status rather than going silent. For `accumulating` units the cap gains a second dimension: a soft context-health budget (a checkpoint threshold and a soft ceiling on resident window) carried in the brief and applied at milestone seams per Right-sizing (Gate 2). A dial, not a cliff.

For code units, the gate must be **machine-checkable** (a test that goes red then green, or a command that exits 0); the cleanest form is the acceptance criteria expressed as tests. When a unit genuinely cannot be machine-verified (a visual or UX judgment), the contract must DECLARE "no machine gate, human spot-check on X" and route that into the return packet's spot-check list. Never silently degrade to a prose checklist.

Defining the contract follows the push-to-define ladder. Its enforcement strength scales with the mechanism (brief-instructed for a single agent, deterministic for long or recurring or overnight mechanisms); the binding is at the adapter edge. The three non-negotiables injected into each agent's brief (closed definition-of-done, findings-sink, budget cap) and the return artifact are specified in the companion [REVIEW-PACKET.md](REVIEW-PACKET.md).

## Return model and live ledger

- **Hygiene lane (singular subagent):** returns inline, folded into the next response. No "a subagent finished" ceremony.
- **Breadth and long-unattended lanes:** return as a review packet (see [REVIEW-PACKET.md](REVIEW-PACKET.md)) at the next natural break, with a one-line "X done, packet ready" ping. The same ping is the async notification when away.
- **Checkpoint protocol (`accumulating` lane):** the brief points the unit at the `continue` skill, fired at milestone seams. Its baton is one artifact in three roles: relay hand-off to the next leg, compaction-recovery point, and packet input. Self-imposed in the brief, since no harness exposes a subagent's live window to watch from outside.
- **Live ledger:** the main thread maintains a running list of in-flight units (lane / mechanism / status / where the packet lands). This is how you keep the lead; scaled up it is the overnight dispatch queue. Its concrete form is bound at the adapter edge.
- **Contradiction-interrupt:** a completed result that invalidates or redirects what the user is actively doing is the one case that breaks "next natural break" and interrupts now.

## The calibration record (cross-session; what the trust ladder advances on)

The live ledger above is in-session and dies with the conversation. Advancing the trust ladder (in [REVIEW-PACKET.md](REVIEW-PACKET.md)) needs more: a durable, cross-session record of how routed runs actually went. A fresh session has no memory of prior runs and will otherwise misjudge the rung from defaults (this happened on 2026-06-19). The policy depends on consuming such a record and names no source; the source is bound at the adapter edge, so it can move from a hand-maintained log to an automated feedback store without touching this core. Without the record, rung advancement is faith-based, which the action-but-always-reviewable spirit forbids.

## Keeping the lead over multi-step work

Routing work off-thread must not let the work drag the effort off its trunk. Four disciplines hold the line:

- **Pin the objective to a file as the north star** before a complex multi-step effort starts. The context window is lost to compaction; a file is not. Every routing decision serves that pinned goal.
- **Scout, then pilot, then fan out.** Enumerate the full work-list first, prove the approach on 2-3 units, then route the rest. Do not fan an unproven approach across the whole list.
- **Run phases as sequential fan-outs with the user in the loop between them.** Read each phase's results before launching the next; a later phase's shape often depends on what the last one returned.
- **Separate research from execution.** Scope and plan as their own units before the units that change things, so a half-formed plan is never executed in parallel.

## The bridge invariant (in-session must grow into overnight)

Every `route` decision must produce the three artifacts the overnight rung consumes, or it is the wrong shape:

1. **lane + mechanism classification** (this is the triage),
2. **injected discipline** (the done-contract above; see [REVIEW-PACKET.md](REVIEW-PACKET.md)),
3. **a review packet on return** (overlay plus per-unit triplet).

If those three are real in-session, the overnight rung is this same policy on a different trigger and substrate, not a new system. Reject any in-session choice those three cannot extend from.

## Composes with (do not duplicate)

- The per-harness adapter edge ([MECHANISMS-claude.md](MECHANISMS-claude.md) for Claude Code, [MECHANISMS-codex.md](MECHANISMS-codex.md) for Codex): the only place concrete tools are named. Each harness has its own.
- [REVIEW-PACKET.md](REVIEW-PACKET.md): the companion holding the return half (the discipline injected into each brief, the packet contract, the merge gate, and the trust ladder). Loaded on demand when a routed unit returns.
- [DESIGN.md](DESIGN.md): the north-star rationale, the eleven resolved decisions, and what was rejected.
