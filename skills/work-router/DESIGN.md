# Work Router: design notes

> The north-star rationale behind the policy in [SKILL.md](SKILL.md): the problem it solves, the resolved decisions, the reasoning that produced them, and what was rejected along the way. The policy core and the per-harness adapter files are the runtime; this file is the why. It names no personal infrastructure and is safe to read standalone.

## Problem / north star

A multi-threaded agent harness gets run in single-threaded mode: almost everything happens in the main driver conversation, bloating context with work that did not need to be there, while the harness's parallelism (subagents, workflows, background tasks, scheduled runs) goes mostly unused. The goal: **only human-in-the-loop (HITL) work stays in the main conversation; everything else is routed off-thread by whatever mechanism fits.** The far star is the "hand a triaged backlog to the agent, wake up to completed work" pattern, reached safely, not jumped to.

The spirit to carry through every choice: this is not about automating the human out of the loop. It is about not wasting the harness's parallelism, while only ever delegating work that can be reviewed with confidence. Every decision bends toward **action by default, but always reviewable and always reversible-or-gated.** When a call is genuinely ambiguous, that sentence is the tie-breaker.

## Scope and staging (the rungs)

Staged, in-session first. Decided explicitly.

- **Now (in-session reflex):** the router runs live in the main loop, classifies each assignment, and dispatches to in-session mechanisms (subagent, parallel fan-out, background run, recurring loop, long-running goal). The user is reachable; the review packet is the return contract. This is the daily bleed and is fully within the agent's direct control.
- **Later rung (overnight backlog handoff):** the same router run ahead of time over a pre-triaged backlog, dispatched on a schedule/cloud substrate, packets collected in the morning. This is the top of the trust ladder; do not start here.

## Bridge invariant (in-session MUST grow into overnight)

A hard constraint: nothing built for in-session may dead-end the overnight rung. Enforced by requiring every in-session routing to produce the **three artifacts the overnight rung consumes**:

1. **lane + mechanism classification** (this is triage),
2. **injected discipline** (closed definition-of-done, findings-sink, budget cap),
3. **a review packet on return** (overlay + per-unit triplet).

If those three are real in-session, the overnight rung is the same router on a different trigger and substrate, not a new system. Reject any in-session choice those three cannot extend from.

## Resolved decisions

1. **The router is a main-loop disposition, not a workflow.** A standing decision policy that runs on every assignment: classify, then dispatch to a mechanism. A detached orchestration is just one mechanism it can reach for (the parallel-breadth one). It must live in the main loop so it stays always-on, reactive, and able to prompt when unsure; a detached job could not do that.

2. **Unit = an assignment** (a coherent sub-task with a definable done-state). Not a whole prompt, not a single tool call. The router fires at two moments: at **plan/decomposition time** (classify each unit of the work-list) and **reactively mid-work** (a sub-task that reveals itself as routable). Canonical flow: **scout inline, then route** (the inline scout is the ground-truth baseline that makes delegated output checkable, and is itself the triage step the overnight rung consumes).

3. **Two-gate decision.**
   - **Gate 1, keep vs route:** does this need the user? Keep in the main thread if it is a cascading design fork, irreversible/outward-facing, a tight back-and-forth, or the final synthesis/decision. Else route out.
   - **Gate 2, mechanism by shape** (only for routed work): read the unit's shape along breadth (singular/parallel), lifespan (inline-watchable/long-unattended), substrate (in-session/survives-session-end), plus a cost read and a growth read as modifiers, then build or pick the mechanism that fits. The concrete mechanisms are bound per harness at the adapter edge.

4. **Three-tier autonomy posture, default leans to action.** Keyed to reversibility x classification-confidence x cost.
   - **Silent** (do it, mention in passing): read-only, cheap, high-confidence. Nothing to veto.
   - **Announce-and-go** (fire, tell immediately, redirectable, non-blocking): bigger-but-revertable, or anything that costs real tokens. The broad safe middle. This is the actual inversion of "never silently escalate": inside this mode, announced action is the agreement.
   - **Propose-and-wait** (stop, ask): hard-to-revert/outward-facing (usually already kept by Gate 1), OR low confidence about whether the user would want to steer. In practice the live trigger here is uncertainty.

5. **Presence is neutral to routing.** Same reflex whether the user is watching or away. Presence never throttles, never adds ceremony, never makes it ask more. It changes only the **channel**: real-time veto + live narration when present; async notification + more upfront pre-authorization when away. The genuine "hold off" signal is **settledness**, not presence: do not route a unit whose definition is still actively churning.

6. **Push-to-define when fuzzy definition is the SOLE blocker.** The router drives the unit to delegatability instead of parking it, on a cost-ordered ladder so it never nags: (1) scout to resolve silently, (2) resolve by stated default, (3) ask only the load-bearing forks scouting cannot settle. Boundary: this applies when the gap is *clarification*; when the gap is genuine hard design that cascades, defining it IS the HITL work and Gate 1 keeps it. Bonus: push-to-define scaled up produces a clean pre-triaged backlog, the overnight on-ramp.

7. **Return model + live ledger.**
   - The hygiene lane (a singular subagent absorbing verbose work) returns inline, folded into the next response. No "a subagent finished" ceremony.
   - The breadth and long-unattended lanes return as a **review packet** at the next natural break, with a one-line "X done, packet ready" ping (the same ping becomes the async notification when away).
   - The main thread maintains a **live ledger** of in-flight units (lane / mechanism / status / where the packet lands). This is the keep-the-lead mechanism and, scaled up, is the overnight dispatch queue.
   - **Contradiction-interrupt:** a completed result that invalidates/redirects what the user is actively doing is the one case that breaks "next natural break" and interrupts now.

8. **Portability: portable policy core + per-harness mechanism file.** The decision core (two gates, mechanism-by-shape, three-tier posture, presence-neutrality, push-to-define, return contract, live ledger) is harness-agnostic and is the durable artifact; it names no harness. The mechanisms are a per-harness binding (the adapter edge): each shape key resolves to a concrete tool on that harness. A new harness adds one adapter file; the core is untouched. Investment: build the policy fully + a full adapter per real harness + a labeled empty slot for the next; no speculative abstraction. This is the same seam as the bridge invariant (the overnight rung is just more rows in an adapter under the same policy).

9. **Form factor: a disposition that fires every session + an on-demand skill.** The disposition (route by default, only HITL stays, build the fitting shape) must fire every session, so its weight lives in the always-on standing instruction the harness loads each session; the reference, contract, illustrations, and priors are needed only once routing is chosen, so they live in the on-demand skill. Collapsing everything into the always-on path was rejected (it bloats every session or, thinned, loses the volunteering the skill exists to produce).

10. **Activation: globally always-on reflex, effort-scoped aggression dial.** The classification reflex is always on (every project/session) with a conservative default posture (silent only for read-only/cheap, announce-and-go for the safe middle, propose for the rest). Aggression is the opt-in per-effort dial. No separate global mode toggle; the per-unit posture is the knob.

11. **The done-contract (first-class output of every routing decision).** Set when the unit is born, always before the agent starts, always carried in the brief. A two-sided bound:
    - **Floor (do not stop short):** a finite acceptance checklist + a verification gate that proves it. Done = checklist met AND gate green, never self-asserted. If the gate cannot be reached, the agent stops and reports "blocked at X" rather than declaring done.
    - **Ceiling (do not spin on "one more thing"):** done is the checklist plus a green gate, NOT the absence of things to improve. Everything found outside the checklist goes to the findings-sink (append to a file / open a triage issue, do not act). Decouples honesty from scope creep.
    - **Budget / iteration cap:** a unit that cannot converge stops and reports status rather than going silent.
    - For code units the gate must be **machine-checkable** (a test that goes red then green, or a command that exits 0); the cleanest form is the acceptance criteria expressed as tests. When a unit genuinely cannot be machine-verified (a visual/UX judgment), the contract must DECLARE "no machine gate, human spot-check on X" and route that into the packet's spot-check list, never silently degrade to a prose checklist.
    - Enforcement strength scales with the mechanism: brief-instructed for a single agent, deterministic for long/recurring/overnight mechanisms.

## How we got here: the reasoning, the reframes, and what we rejected

The hard-won context a bare spec would lose. Each turning point, why the choice beat its alternative, and what was ruled out so it is not re-litigated.

**The framing reframe: this is a posture inversion, not a new decision aid.** The starting point was a prior delegate-vs-keep decision aid whose stance was "orchestration is the exception, never silently escalate." The ask is the opposite: make delegation the reflex. So the genuinely new work is (a) flipping the default to action-leaning, and (b) the mechanism-selector (which shape fits), which the old aid stopped short of. A session that starts rebuilding a delegate-vs-keep framework from scratch is duplicating, not extending. (That prior aid was absorbed into the skill and removed.)

**Three lanes, not one "AFK" blob.** The goal was first framed as "push AFK work off the main thread," but that bundled three animals with different triggers, risk, and return paths: context-hygiene (offload verbose output, returns a summary, always safe), parallel breadth (independent units, cost is the only risk), and unattended AFK (proceeds while away, reviewed cold, needs the gate). A unit can be context-bloating without being AFK. The router classifies into these distinct lanes; do not re-collapse them.

**The router is NOT a workflow (explicitly rejected).** Good routing *traces* like a workflow (scout, classify, dispatch, collect, synthesize). But a detached orchestration runs without the ability to stop and ask mid-run, and "prompt me when unsure" is non-negotiable. So the router is a main-loop disposition; a detached orchestration is one *mechanism* it dispatches to. Rejecting the workflow framing is what keeps it always-on, reactive, and interactive.

**Presence is neutral to routing (a correction worth preserving).** An early framing made it sound like the user being present would throttle the automation. That was pushed back on hard: being there must never make it work less well. The real "hold off" signal is not presence, it is an unsettled spec (definition still churning). Presence changes only the channel, never whether the router acts. And when fuzzy definition is the sole blocker, the router pushes to define rather than parking the unit.

**Always-on reflex, effort-scoped aggression (rejected a pure toggle-mode and rejected always-aggressive-global).** A toggle you forget to flip recreates single-threaded-by-default. Always firing fan-outs globally is reckless. Resolution: the classification reflex is always on everywhere with a conservative default posture, and aggression is the opt-in per-effort dial.

**No enforcement hook yet (deletion test said vanish).** A hook re-injecting a router reminder would just duplicate the always-on reflex, a pass-through. The one thing that must be deterministic (the pre-merge gate) is covered separately. Build a reminder hook only if the reflex is observed lapsing.

**The done-contract is a two-sided bound.** Loops and goals work because they self-verify against a defined exit. The done-contract is a floor (a finite checklist + a machine-checkable gate; do not stop until green, report "blocked at X" rather than declaring done) and a ceiling (done is the checklist, not the absence of improvements; extras go to the findings-sink). The two sides map exactly to the two failure modes: stopping short, and spinning on "one more thing."

**Why staged in-session-first, and the bridge invariant.** "Hand over the backlog, wake up to merged code" is the top of the trust ladder; starting there is what the comfort condition forbids. So the in-session reflex is built first. But in-session must naturally grow into overnight. The guarantee is the bridge invariant: every in-session routing already produces the three artifacts the overnight rung consumes. Overnight is then the same router on a different trigger and substrate, not a new system. This is also why portability and the overnight bridge are the same seam (more rows in an adapter under the same policy).

## Later decision (v4): context health as a growth read

Trigger: routed `accumulating` units can run deep into the degraded zone (attention dilution, instruction/goal drift, stale-context anchoring, recency bias, self-reinforcing error, the per-turn re-send tax, and eventually compaction loss; compaction is one instance, not the definition). A verified roughly 29-minute build agent peaked around 470k resident tokens, never compacted on a large context window, and still landed correctly because the done-contract forced externalization to disk; that quantified the real enemy as quality/steerability decay from the low hundreds of thousands of tokens onward, not compaction.

**Decision.** Context health enters as a growth read on the Gate 2 key (`discardable` vs `accumulating`), the twin of the cost read. `accumulating` units projected past a context-health budget are right-sized into a **relay** (a journaled pipeline, or an agent chain handing off a continuation baton) split at milestone seams; the budget is a quality-knee dial, not a hard cliff, and the seam is the trigger. An off-thread continuation/handoff mechanism becomes the checkpoint, its baton serving as relay hand-off, compaction-recovery point, and packet input.

**Rejected.** (a) An external mid-run monitor on a subagent's live window: no harness exposes that from outside, so the signal must be self-imposed in the brief. (b) A hard token cliff: over-decomposition pays a lossy handoff at every seam, and large-context models tolerate the degraded zone better than the folklore threshold. (c) Resume to continue a maxed-out unit: it carries the bloat forward; relay (fresh context + baton) is the degraded-zone tool, resume is continuity-only.

## Later decision (v5): the calibration-record dependency, and agent-authored loop shapes

### A. The calibration-record dependency

Trigger: "what rung are we at" had no answer, because the trust ladder advances on calibration and nothing recorded how prior routed runs went. A fresh session, reading only the design doc's default, will misjudge the rung (it did: reported a low rung while routed runs had already happened earlier). Root cause: the live ledger is in-session and dies with the conversation; there was no durable cross-session record.

Decision: the policy depends on consuming a durable, cross-session calibration record, and names no source (see SKILL.md, The calibration record). The source is bound at the adapter edge: an interim hand-maintained log now, an automated feedback store later. Same adapter-swap seam as the next-harness slot; when the automated store lands the manual log deletes cleanly. The dependency is permanent (trust always needs evidence); the manual scaffolding is temporary. Rejected: keeping the record only in the in-session live ledger (loses it at conversation end); asking the human to recall (unreliable, a daytime session was once misremembered as an overnight run and corrected only by re-reading the transcript).

### B. Agent-authored loop shapes (the compose rung)

Trigger: a worked example of an agent authoring a bespoke state-machine orchestrator (spawn an implementation thread, spawn a review thread, loop to approval, gate a merge, trigger the next piece). This authored dynamism was the router's original intent that an early version under-served: a resolution ladder that only ever *selects* an existing primitive never crosses to *authoring* one.

The reframe: the harness already had the composition substrate (a general orchestration language for detached shapes; an attended coordinator for shapes that must pause). The gap was framing, not capability. The fix: the axis grid is reframed as a **cache of common compositions**, and resolution gains the ability to **compose** a bespoke shape on a double miss. Composition splits along the existing policy/mechanism seam (policy clause in SKILL.md, substrate binding at the adapter edge) and is a property of the mechanism slot, not the router, so "the router is NOT a workflow" stays literally true. The depth is the five-invariant composition contract (primitives-first; vetted leaves, novel wiring; irreversible steps gated-never-embedded; stop-to-ask selects the substrate; same done-contract and packet), not the trivial "let the agent write a workflow" capability.

Staging: enabled now (it is a reframe, not new machinery) but gated, irreversible steps pause or pass the merge gate, and launching a composed shape is conservative on the autonomy tiers. Fully-autonomous composed shapes that merge unattended are the top-rung form, deferred until in-session trust is earned. Rejected: an "open vocabulary, composition-first" reframe of all of Gate 2 (over-applies, pays composition cost on every routine unit when most are a clean grid row); a separate composer companion file (a rung plus a contract clause, not a body of machinery, fails the deletion test); permissive autonomy now (takes on the worked example's blast radius before trust is earned).

## Later decision (v6): de-prescription, the two-clocks form factor, and the trigger map

Trigger: the skill had become too prescriptive (an axis grid used as a lookup, a rung ladder with compose as a last resort), which risks forcing the agent into lanes when the whole point of these primitives is to let it build the fitting shape. Deeper context: the skill exists because over a long stretch of use the agent never *volunteered* any of these mechanisms (subagents, fan-outs, dynamic compositions, loops, goals, higher effort); it was built to close that disposition gap, not to cage the how.

**Holistic re-evaluation.** Every aspect was sorted by "does this restrict the agent, or enable/guide it?": disposition (route by default, only HITL stays, compose the fitting shape) is the engine that fixes the volunteering and is kept; judgment aids (Gate 1, posture, cost/growth reads, keeping-the-lead) guide without dictating and are kept; tool knowledge (capability profiles) enables composition and is kept; the safety/return contract (done-contract, the review packet, irreversible-gated, trust ladder, calibration record) constrains the *result* not the *method* and is kept; only **mechanism prescription** (the grid-as-lookup, the ladder-as-procedure, compose-as-last-resort) is the cage, and it is cut to priors. The prescriptive *tone* had bled into the rest; the fix is to reframe, not gut.

**Two clocks (the form-factor decision).** The disposition must fire *every session* to make the agent volunteer, and a skill cannot self-fire, so the disposition's weight lives in the always-on standing instruction (strengthened to name the under-used built-ins and carry the fan-out gate). The reference, contract, illustrations, and priors are only needed *once routing is chosen*, so they stay in the skill, loaded on demand. Rejected "collapse everything into the always-on path": inline reference bloats every session, thinning it loses the volunteering. The hybrid is forced by the two-clocks fact, not a compromise.

**The fan-out gate (keystone, well-evidenced).** Practitioner research converges on this: parallelism's ceiling is human verification, not generation, and the strongest solo practitioners deliberately under-use parallel fan-out. So fan out only when units are independent AND each self-evaluates (test, rubric, bot); skip it when they share mutable state or all funnel back to one reviewer. Sizing: a few focused workers beat a scattered swarm; cap validation at one. This is the counterweight that keeps the action-by-default disposition from becoming "spawn everything."

**The full trigger map (signal -> reach for -> note).** Lives here so the skill stays compact; the skill carries only a handful of illustrations. These are priors seeded from research, not a track record; the real priors accrue via the calibration record.

- Tangential dig that would flood context -> single subagent, return a summary (hygiene lane).
- Learn a codebase before doing it for real -> scout / throwaway probe, discard the code.
- React to output as it streams -> a stream-reactor, not a re-polling loop.
- Long build / dev server / test-watch that blocks -> background run.
- Verifiable done-state, unknown effort, walk away -> a goal-loop + maker/checker split + iteration cap.
- Recurring poll, no natural end -> a loop on a cadence.
- Runs overnight / independent of session / on an event -> a scheduled or cloud run (the survives-session-end substrate).
- One-off task that is long/parallel/structured/adversarial -> an agent-authored (dynamic) composition.
- Same multi-agent shape run repeatedly -> a saved/named composition.
- Many small steps each wanting a clean context -> fan-out-and-synthesize / map-reduce.
- Output the generator cannot grade itself -> judge panel + adversarial verify (about one reviewer per three to four builders).
- Taste or qualitative pick among candidates -> tournament (pairwise judging).
- Unknown work volume ("until no new findings") -> loop-until-dry.
- Parallel work that would collide -> coordinator/heartbeat (isolation per worker, periodic check-ins).
- Agents stuck repeating -> guardrails: iteration cap, forced reflection, kill-and-reassign.
- Sustained stretch where you would pitch a fan-out repeatedly -> raise the aggression dial (bounded).
- Complex reasoning or failing at the current level -> bump reasoning effort.
- Cheap vs expensive subtasks -> classify-and-act model routing.
- Review bot returns many findings -> self-repair loop until no feedback.
- Repeated corrections across sessions -> distill into the standing instructions.
- "Every time X, do Y" automatic enforcement -> a hook.

**Rejected.** Stripping to a bare vibe (no priors): the disposition decays without something concrete to fire on, so it would snap back to inline work. Keeping the structure and only softening wording: the lanes still read as lanes. A "priors = shapes that have worked for us" framing: dishonest without a track record; the priors are seeded from research, labeled as such, and the *real* ones accrue via the calibration record.

## Open questions (resolved when the skill was first built)

- **Live-ledger concrete form** -> resolved: bound on the adapter edge (a harness task list for several in-flight units, a lightweight in-context list for one or two); the policy core only says "maintain a live ledger."
- **Mechanism-selection edge cases** -> resolved by the hybrid axis grid: a unit both verbose and parallel routes parallel (breadth wins), and the cost read downgrades a `parallel / in-session` unit from a full fan-out to a few agents. Novel units that fit no row drop to the capability profiles, then to composition.

## Composes with (do not duplicate)

- [SKILL.md](SKILL.md): the portable policy core (two gates, posture, push-to-define, done-contract, return model, bridge invariant). Names no harness.
- The per-harness adapter file (MECHANISMS-claude.md for Claude Code, MECHANISMS-codex.md for Codex): the only place concrete tools are named. A new harness adds its own.
- [REVIEW-PACKET.md](REVIEW-PACKET.md): the return half (the packet contract, the discipline injected into briefs, the merge gate, the trust ladder).
