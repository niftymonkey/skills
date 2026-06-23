# Work Router: the review packet (companion)

How a routed unit comes back so it can be reviewed without having been watched. This is the return half of the policy in [SKILL.md](SKILL.md): work-router's two gates decide what to route and how; this file defines what comes back and the discipline that keeps the run honest. Loaded on demand when a routed unit produces a return (the breadth and long-unattended lanes); the hygiene lane folds inline and needs no packet.

## Why this shape

In an interactive session you are not really reviewing at the end. You are sampled continuously as a steering function: the agent surfaces a recommendation, you redirect, dozens of small corrections, so by the time you see the diff you already hold a model of what was built. Routed and unattended work strips that out and hands you a finished artifact with no map, which is why accepting blind feels rational and is the failure mode. The fix is not a better diff; it is to **reconstruct the steering surface**: surface the handful of forks the agent took that you would have redirected, not just the result. That is why the packet leads with the decision log and the overlay, not the diff.

The worked example that taught this shape: a self-reviewed agent merge that shipped a dry-run bug, accepted blind because no one reconstructed the forks the agent had taken. The trust ladder is set out in depth at the end of this file.

## The discipline injected into every brief

State the done-contract (defined in [SKILL.md](SKILL.md)) into each agent's brief as three standing instructions. They prevent the failure modes that make routed runs spin, drift, or ship blind:

1. **Closed definition-of-done.** Acceptance criteria are a finite checklist; done = criteria met AND the verification gate green. "Nothing left to improve" is never the bar; that bar is unreachable and is what makes a run spin and burn budget.
2. **Findings-sink.** "If you discover anything outside this unit's scope, do NOT fix it and do NOT drop it. Append it to a findings file or open a triage-labeled issue, then continue." Decouples honesty from scope creep.
3. **Budget / iteration + context cap.** A turn or token ceiling so a run that cannot converge stops and reports "blocked here, state is X" rather than grinding silently. For `accumulating` units this also carries the context-health budget (a checkpoint threshold and soft ceiling on resident window): at a milestone seam past the checkpoint the agent externalizes progress via `continue` and either finishes or returns "budget reached, baton written" for relay (see SKILL.md Right-sizing).

## The packet (the output contract)

Always this shape, regardless of how many units. Read the overlay first; drill into a triplet only where the overlay or a spot-check points.

### One conversation-level overlay (read first)

- **Work split + injected guardrails:** how the work was divided and the standing instructions each agent got.
- **Post-return decisions:** anything decided AFTER the agents returned. These belong to no single agent, and the most consequential call often lives here.
- **Cross-item coherence:** do the results cohere or conflict, is anything fine per-unit but a problem combined. Integration problems live here; no single agent can see them.

### One triplet per unit

- **Decision log:** every fork the agent took that a watching human might have redirected, each with the road not taken and how reversible it is. The highest-value part.
- **Tech/Outcome recap:** the standard recap (Technical bullets paired with Outcome bullets), so the result does not feel disconnected from how interactive sessions end.
- **Spot-check list:** the two or three places actually worth human eyes, so review is bounded rather than all-or-nothing.

## Gate before merge (do not merge blind)

- Re-run the review gate on the EXACT final commits, not an earlier state. The concrete gate is bound at the adapter edge (`coderabbit:code-review` on Claude Code; `/review` or `@codex review` on Codex).
- A pre-merge gate hook can backstop a forgotten re-run: it blocks an agent `gh pr merge` until the gate has run on the final commits. The concrete hook is bound at the adapter edge.
- Present the packet and let the human approve having actually read the overlay and spot-checks. An authorized merge is not a reviewed merge; the packet is what makes the reading cheap enough to actually happen.

## Emitting the packet by mechanism

- **Workflow:** each unit is a pipeline item whose stage emits its triplet (use a schema so fields come back structured, not prose). A final synthesis stage reads all triplets and emits the overlay (the one step that genuinely needs all results at once, for cross-item coherence). `pipeline()` by default.
- **A few Agent calls:** inline them and write the overlay yourself.
- **A single background run:** the unit's triplet is the return; write the (small) overlay yourself.
- **A relay (chain or pipeline):** the final leg emits the unit's triplet, reading the baton trail for the decision log; write the (small) overlay yourself.

## The trust ladder (where you are determines the next safe step)

Do not jump to full autonomy; reaching for the top rung is why it feels two steps beyond comfort. The rungs:

1. Sleep on one hand-vetted unit.
2. Review the result and CALIBRATE: which mechanism bit (drift, lost findings, scope creep, cost). This reading is worth more than more planning.
3. Templated batch: encode the discipline above, run two or three hand-picked units.
4. Let the agent pick the next unit itself.
5. Schedule a recurring low-risk class of work. This is the `survives-session-end` row at the adapter edge, the overnight rung.
