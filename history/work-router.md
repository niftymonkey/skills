---
skill: work-router
created: 2026-06-14
current-version: v7
status: published
---

# work-router: History

## Problem

The user runs a multi-threaded harness in single-threaded mode. Almost everything happens in the main driver conversation, bloating context with work that did not need to be there, while the harness's parallelism (subagents, workflows, background tasks, scheduled runs) goes mostly unused. There was no standing reflex deciding, per unit of work, what should leave the main thread. The goal: only HITL work stays in the main conversation, everything else is routed off-thread by whatever mechanism fits, and only ever work that can be reviewed with confidence is delegated.

The gap the existing artifacts left: `orchestration-decisions.md` already decides delegate-vs-keep, but with the opposite default ("orchestration is the exception, never silently escalate"), since absorbed into this skill and deleted in v3; an existing AFK-review method owned the return-path discipline once you had already delegated (since folded into this skill as the `REVIEW-PACKET.md` companion); neither made delegation the reflex, and neither selected the mechanism. work-router is the posture inversion plus the mechanism-selector that sits between them.

## How it works (brief)

Two-tier skill. The portable policy core (`SKILL.md`, names no harness) runs as a main-loop disposition on every assignment: Gate 1 keep-vs-route (does this need the user?), Gate 2 mechanism-by-shape (for routed work). The Claude binding (`MECHANISMS.md`, the adapter edge) is the only place concrete tools are named, so a new harness swaps that one file. A three-tier autonomy posture (silent / announce-and-go / propose-and-wait) defaults to action; presence is neutral to routing; push-to-define drives a fuzzy unit to delegatability rather than parking it; a two-sided done-contract (floor + ceiling + budget cap) is set before any agent starts; results return as a review packet (the companion `REVIEW-PACKET.md`) tracked in a live ledger of in-flight units. Every routing produces the three artifacts the overnight rung will consume (the bridge invariant), so in-session grows into overnight without a rewrite.

See `SKILL.md` for the policy, `MECHANISMS-claude.md` / `MECHANISMS-codex.md` for the per-harness bindings, `REVIEW-PACKET.md` for the return half, and `DESIGN.md` for the full reasoning and the eleven resolved decisions.

## Iteration log

### 2026-06-14, v1 (initial)

Built Module 1 of the Work Router from the design notes, which forked from a prior case study of real delegate-vs-keep calls. The decision phase (eleven resolved decisions) was already complete; this slice built the skill.

An architect-deep "design the interface twice" pass ran inline on the mechanism table (kept in the main thread by the router's own Gate 1, not delegated). Three interface shapes were compared (flat shape table / orthogonal axes / capability matcher) on depth, locality, and execution cost; the user chose the hybrid: an axis-grid lookup keyed by breadth x lifespan x substrate plus a cost read for the common case, with per-mechanism capability profiles as the edge-case fallback and the new-harness contract. Key architectural finding: the policy/mechanism seam is real today, not speculative, because in-session and survives-session-end are already two adapter-sets across the substrate axis (the bridge invariant restated as a seam), which clears architect-deep's "two adapters = real seam" bar without waiting on a second harness.

Name resolved from provisional "Work Router" to `work-router`. The three folded-in open questions were resolved: ledger form bound on the adapter edge (harness task list or in-context list), and the mechanism-selection edge cases handled by the hybrid table (breadth wins when a unit is both verbose and parallel; the cost read downgrades parallel from a Workflow to a few agents).

Deliberately out of scope for v1: Module 2 (the always-on reflex, a memory rule plus a CLAUDE.md line pointing at this skill, the thing that actually fires it), Reconciliation B (a light cross-reference edit to `orchestration-decisions.md`), Module 3 (an enforcement hook, deferred per the deletion test), and the overnight/cloud rung (present as one dormant row in `MECHANISMS.md`).

### 2026-06-14, v2 (folded the return-review method in as a companion)

Demoted the standalone `afk-review` skill into work-router as the `REVIEW-PACKET.md` companion. Rationale: afk-review had exactly one real consumer (the router), was incomplete on its own, and was never going to be invoked or published alone, so a separate skill was just indirection rather than a real seam. Folding it in gives one source of truth for the return/review machinery, loaded on demand, with no drift between two copies.

Deduplicated on the way in: the fan-out-vs-inline decision (already work-router's two gates) and the done-contract substance (already in `SKILL.md`) were not copied; the companion keeps only the unique core (the packet output contract, the brief-injected discipline, the merge gate, the workflow mapping, the trust ladder). The afk-review rationale was preserved before the skill directory was removed, per the lean-companion / deep-why-external choice. References in `SKILL.md` and `MECHANISMS.md` were repointed from the afk-review skill to the companion; the merge-gate hook and the global CLAUDE.md needed no changes (zero references to update).

### 2026-06-15, v3 (activated the reflex; absorbed orchestration-decisions)

Built Module 2, the always-on reflex. Replaced the old `## Claude Code: Proactively suggest workflows / higher effort` section in the global `CLAUDE.md` (the previous "orchestration is the exception, never silently escalate" posture this skill inverts) with a `## Work-router reflex` section. It is a thin trigger, not a restatement of the policy: it flips the default to route-off-thread, names the three firing moments, and says "load the `work-router` skill and follow it." A first draft that restated the gates, posture, and push-to-define inline was rejected as duplication that would let the line and the skill drift. Depth stays in the skill, since a skill cannot self-fire each turn and a global memory rule would be project-scoped and dormant.

Reconciliation B started as a light cross-reference edit and was then taken all the way to a fold-in. `orchestration-decisions.md` was absorbed into the skill and deleted, on the same deletion-test logic that demoted afk-review in v2: once the old CLAUDE.md section was replaced, the skill was its only agent-facing consumer, so a separate referenced file was just a losable dependency. What moved: the cost magnitudes (agents ~4x, multi-agent ~15x, scope-small-first) and the effort/ultracode aggression dial into `MECHANISMS.md`; the keep-the-lead disciplines (pin the north-star objective, scout-then-pilot-then-fan-out, sequential phases with the user in the loop, separate research from execution) into a new `SKILL.md` section. The workflow-quality checks were dropped, not moved, as redundant with the `Workflow` tool's own description.

Keep-the-lead was the load-bearing reason the fold had to happen rather than a plain delete. Those disciplines had no published backup (the two published explainers, the Quick Card and the Field Guide, cover only the workflow/ultracode subset), so deleting without folding would have lost the one piece the user most wanted preserved. Folding promotes it from a referenced file into the policy core. Both edits (the CLAUDE.md reflex and the config deletion) are uncommitted in the `claude` repo, awaiting the user's go.

### 2026-06-15, v4 (context-health dimension: the growth read and relays)

Added a context-health dimension so routed `accumulating` units do not degrade as their resident window grows, motivated by the wide failure set (attention dilution, instruction/goal drift, stale-context anchoring, recency bias, self-reinforcing error, the per-turn re-send tax, and eventually compaction loss), not compaction alone. A verified ~29-minute build agent (~470k peak resident, no compaction on its 1M window, correct outcome via forced externalization) calibrated the enemy as quality/steerability decay from ~150-200k onward.

Kept deliberately small after an architect-deep pass that resisted a context-management subsystem: a **growth read** (`discardable` / `accumulating`) on the Gate 2 key, twin of the cost read (`SKILL.md`); a **Right-sizing** block that relays an over-budget accumulating unit at milestone seams, budget as a quality-knee dial not a cliff, with a relay-not-resume rule (`SKILL.md`, `MECHANISMS.md`); the done-contract **Budget cap deepened in place** to carry the context-health budget (`SKILL.md`, `REVIEW-PACKET.md`); a **checkpoint-protocol** return lane reusing `continue` off-thread (`SKILL.md`); a **relay capability profile** plus a growth-lever note (`MECHANISMS.md`). The decision runs in one place (Gate 2); other mentions record its output. Deep rationale and rejected alternatives in `DESIGN.md` ("Later decision (v4)"). Uncommitted in the `claude` repo, awaiting the user's go.

### 2026-06-19, v5 (calibration-record dependency + agent-authored loop shapes)

Two changes from one conversation (a video on designing agent loops, plus a status question that exposed a blind spot). Both detailed in `DESIGN.md` ("Later decision (v5)"). Uncommitted in the `claude` repo, awaiting the user's go.

**Calibration-record dependency.** A status question ("what rung are we at") had no answer because the trust ladder advances on calibration and nothing recorded how prior routed runs went; a fresh session reported the wrong rung from the doc's default while overnight runs had already happened days earlier. Added a policy dependency on a durable, cross-session calibration record, source-agnostic in `SKILL.md` (The calibration record) and source-bound at the adapter edge in `MECHANISMS.md` (Calibration record): an interim hand-maintained log now (a personal calibration log kept outside the repo), an automated feedback store later, same adapter-swap seam as the next-harness slot. The first rung-2 reading is logged: the routed runs to date were human-gated rung 1-3, never autonomous, with the local CodeRabbit gate clean on final commits.

**Agent-authored loop shapes (the compose rung).** Gate 2's resolution ladder gains a third rung so the router can author a bespoke control-flow shape, not only select an existing primitive. The axis grid is reframed as a cache of common compositions; composition is rung 3 on a double miss, splits along the policy/mechanism seam, and is a property of the mechanism slot (so "the router is NOT a workflow" holds). The depth is the five-invariant composition contract (primitives-first; vetted leaves, novel wiring; irreversible steps gated-never-embedded; stop-to-ask selects the substrate; same done-contract/packet). Gated per the user's call: enabled now, conservative autonomy, fully-autonomous composed shapes deferred to rung 5.

### 2026-06-19, v6 (de-prescription, the two-clocks form factor, and the trigger map)

Reshaped the skill from a decision procedure into a disposition-plus-reference after the user flagged it had become too prescriptive (an axis grid used as a lookup, a three-rung ladder with compose as a last resort), caging the *how* when the skill's actual job is to close a disposition gap (across roughly a year the agent never volunteered subagents/workflows/loops/goals/ultracode on its own). A holistic re-evaluation sorted every aspect by "restricts vs enables": only mechanism prescription was the cage and is cut to priors; disposition, judgment aids, tool knowledge, and the safety/return contract (including the review packet the user called non-negotiable) are kept, with the prescriptive tone reframed out.

Form factor settled by a two-clocks argument: the disposition must fire every session (so it lives in `CLAUDE.md`, which can self-fire; strengthened to name the under-used built-ins and carry the fan-out gate), while reference/contract/illustrations are needed only once routing is chosen (so they stay in the on-demand skill). Collapsing everything into CLAUDE.md was rejected (bloats every session or loses the volunteering).

Concrete changes: `SKILL.md` drops the three-rung ladder and compose-as-last-resort (resolution reframed to "build or pick the fitting shape; primitives-first only as laziness; grid/profiles/illustrations are priors, not lanes"; composition made first-class) and gains a **fan-out gate** keystone (parallelism's ceiling is human verification, so fan out only when units are independent and each self-evaluates; sizing roughly 3-5, validation capped at one; well-evidenced from practitioner research). `MECHANISMS.md` shifts from lookup to reference and adds six **trigger illustrations** (invent-beyond) plus the under-used built-ins (Monitor, Routines, effort/ultracode dial, self-repair). The `CLAUDE.md` reflex now names the built-ins to suggest and carries the one-line gate. The full trigger map and the rationale live in `DESIGN.md` ("Later decision (v6)"); the calibration record fills the real priors as runs accrue. Uncommitted in the `claude` repo, awaiting the user's go.

### 2026-06-23, v7 (promoted to publishable repo; Codex adapter added)

Promoted from the niftymonkey/claude repo to the public niftymonkey/skills repo. Two substantive additions made this more than a file move:

- **Codex adapter authored.** The Claude-only `MECHANISMS.md` was renamed to `MECHANISMS-claude.md` and a sibling `MECHANISMS-codex.md` was added, mapping the same `SKILL.md` policy onto Codex's real primitives (native subagents, `codex exec`, `codex cloud`, the local `/review` agent plus the GitHub `@codex review` gate, lifecycle hooks, and the `(sandbox, approval)` aggression dial). Four routing decisions with no native Codex equivalent are marked in a Gaps section rather than invented: the stream-reactor (`Monitor`), a CLI-native recurring loop, a single built-in maker-not-checker goal-loop, and the always-firing standing reflex. `SKILL.md` was made genuinely harness-neutral: it now branches by harness ("read the mechanisms file matching your harness") instead of naming the Claude file. The Codex bindings are dated (Codex CLI moves fast) and resolve the long-standing "portability proven by construction but not by a second adapter" uncertainty.

- **Design rationale bundled.** The external personal design doc was split. The portable rationale (problem/north-star, the eleven resolved decisions, the how-we-got-here reasoning and rejected alternatives, the v4/v5/v6 later-decisions) was bundled into a new `DESIGN.md` companion, scrubbed of personal paths, URLs, and project names. The living calibration log stays personal and unpublished; the calibration-record binding now points generically at "a personal calibration log kept outside the repo."

Portability remediation during promotion (via `/promote-skill`): repointed personal-path references in `SKILL.md`, `MECHANISMS-claude.md`, and `REVIEW-PACKET.md` to `DESIGN.md`; reframed a named personal pre-merge hook to the abstract pre-merge-gate-hook pattern; genericized the feedback-store binding (dropping an incubating cross-skill reference and a personal project-memory name); removed a personal-domain URL; and made the harness-neutral merge gate in `REVIEW-PACKET.md` name both harnesses' concrete gates. This `HISTORY.md` was scrubbed for the public page (neutralized the author name, removed project codenames and personal-infrastructure references).

Source-of-truth moved from the niftymonkey/claude repo to the niftymonkey/skills repo (skills/work-router/ + history/work-router.md).

## Design uncertainties

- Module 2 now fires the skill (the global CLAUDE.md reflex), but its real-world behavior is still unobserved; whether the two gates and the action-leaning default land at the right altitude, and whether voluminous-but-simple work actually leaves the main thread (and returns as a summary, not a replay), needs live use to tell.
- The mechanism table binds to Claude Code primitives as they exist on 2026-06-14 (Agent/Workflow/background Bash/`/loop`/ScheduleWakeup, with CronCreate/cloud as the later rung). Harness primitives move fast; the bindings will drift and need refresh. The Codex adapter (v7) carries the same caveat and is dated.
- The done-contract requires a machine-checkable gate for code units, with a declared human spot-check as the only escape. Whether that bar is too strict for small in-session routes (versus only overnight ones) is untested.
- The capability-profile fallback is a reasoning procedure, not a lookup; whether an agent reliably drops to it for novel units, rather than forcing them into an ill-fitting grid row, is unverified.
- Portability is now exercised by a second adapter (Codex, v7), not only proven by construction. Whether the Codex bindings hold up in real Codex use (and whether the marked gaps are the right ones) is the next thing live use will show.
- The context-health budget is a quality-knee dial (default checkpoint ~200k on a 1M window), not pegged to one failure mode; the right threshold per model and work type, and whether milestone-seam relays beat one long agent in practice, is unobserved. On 200k-window models, quality decay and compaction converge much lower and the dial must drop with them.

- The compose rung is a reasoning procedure layered on the capability-profile fallback; whether an agent reliably reaches rung 3 to author a fitting shape (rather than forcing a novel unit into an ill-fitting primitive or hand-running it) is unverified, the same open question as the profile fallback one rung up.
- The calibration record's interim form is a hand-maintained log; whether it actually gets appended after routed runs (rather than forgotten) before an automated feedback store handles capture is the live risk, the manual-discipline gap the dependency names but cannot itself enforce.

- The whole v6 bet is that a strong disposition plus a few priors makes the agent *volunteer* these mechanisms unprompted, where a year of weak or absent disposition did not. Whether it actually fires (the agent suggesting Monitor / Routines / workflows / ultracode on its own, at the right moments, without over-spawning) is unproven, and it is exactly what the calibration record is meant to measure.

## Files

- `SKILL.md`: the portable policy core (two gates, posture, push-to-define, done-contract, return model, bridge invariant). Names no harness.
- `MECHANISMS-claude.md` / `MECHANISMS-codex.md`: the per-harness adapter edges (axis grid plus capability profiles plus ledger/packet bindings; the Codex adapter marks routing decisions with no native Codex equivalent).
- `REVIEW-PACKET.md`: the return half (the review-packet output contract, the discipline injected into briefs, the merge gate, the workflow mapping, the trust ladder). Folded in from the former `afk-review` skill.
- `DESIGN.md`: the north-star design doc, the eleven resolved decisions, and what was rejected.
