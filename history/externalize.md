---
skill: externalize
created: 2026-06-21
current-version: v2
status: published
---

# externalize: History

## Problem

Hard-won context (research with sources, architectural framing, dead ends with their reasons, decisions with rationale, costly code understanding) is earned continuously during work but only ever lives in the context window until something deliberately writes it down. The model never volunteers to externalize it as it goes. So when the window fills, that context is either lost outright or flattened by the harness's auto-compaction into a generic gist that drops exactly the specific, hard-won parts that were worth keeping.

The workaround was the existing on-demand handoff skills (`continue` for code, `carryover` for personal), fired manually at a boundary: end of session, or when context got tight. Two problems with that. First, firing at a boundary is exactly when the window has already lost the earliest and richest context, so the capture is incomplete by construction. Second, the user stopped reliably firing them: a manual command at session end is easy to skip, and once skipped the context is gone. There was no standing reflex that captured value at the moment it was earned, so context level and compaction kept mattering when they should not.

## How it works (brief)

externalize is a standing main-loop disposition, the continuous-write capture sibling of `work-router` (work-router routes WORK off-thread by default; externalize externalizes CONTEXT to disk by default). The instant a unit of expensive, non-reproducible context is earned, it gets appended to the live work-thread handoff file (`continue.md` for code, `carryover.md` for personal, or whatever handoff the thread keeps). The trigger is the moment of earning, not session end and not a context-window threshold. Because the value is already on disk while fresh, context level and compaction stop being failure modes.

externalize owns the capture-quality discipline that `continue` and `carryover` previously each carried (the shared "spine": capture faithfully, fight recency, keep dead ends with their reasons, note compacted-summary provenance, redact secrets). That spine migrated up into this skill because faithful capture is externalize's whole thesis; the two curate-side siblings now reference externalize for it instead of each holding a copy. externalize is the WRITE side only: it never curates. Pruning (continue) and preserving (carryover) stay where they were.

See `SKILL.md` for the disposition, `skills/continue/SKILL.md` and `skills/carryover/SKILL.md` for the two curate-side siblings it feeds.

## Iteration log

### 2026-06-21, v1 (initial)

Split the lifecycle of the work-thread handoff file into a continuous WRITE side and an on-demand CURATE side. externalize is the new WRITE side; the existing `continue` and `carryover` skills become purely the CURATE side.

The design move that motivated the split: `continue` and `carryover` felt like duplicates because they shared a "spine", the capture-quality discipline (the window will lose or lossily summarize hard-won context, so capture faithfully, fight recency, keep dead ends with their reasons, note when content is drawn from a compacted summary, redact secrets). That spine is exactly externalize's thesis, so it migrated up here. `continue` and `carryover` then shed the spine and reference externalize for it, keeping only their divergent curation cores (continue PRUNEs toward durable external homes; carryover PRESERVEs because personal context has no external home). Net: both existing skills got lighter, the capture principle has one home, no duplication, and the two curation skills stay separate because their cores are inverses and must not be fused.

The clean trigger map this produces: "don't lose expensive context I just earned" -> externalize (automatic); "survive an involuntary compaction" -> externalize keeps the file current, with a separate git-state backstop hook as the floor; "deliberately start a clean code session" -> continue (on-demand); same for a personal thread -> carryover (on-demand).

In scope for v1: the disposition itself, the firing rule (earned, not boundary or threshold), the what-to-write / what-to-skip-because-it-has-a-durable-home distinction, the target-file selection (domain-general across continue/carryover/active handoff), the migrated capture-quality spine, and the explicit WRITE-side-only boundary.

Deliberately out of scope for v1: any curation behavior (owned by continue/carryover); a hard enforcement mechanism (rich capture needs the model, not a hook, so this stays a best-effort disposition, with the existing git-state backstop hook as the only mechanical floor); and the always-on reflex line in `CLAUDE.md` / `AGENTS.md` that would actually fire this every session (drafted but not installed, because `CLAUDE.md` is mid cold-start experiment as of 2026-06-21 and its placement is the user's call).

### 2026-06-23, v2 (promoted to publishable repo)

Promoted from the niftymonkey/claude repo to the public niftymonkey/skills repo. A clean promotion: externalize is a harness-agnostic writing disposition with no per-harness mechanism file, so no adapter or design-doc bundling was needed (unlike its sibling work-router). Promoted alongside work-router, second in the wave so its `work-router` cross-reference resolves against the now-published skill.

Portability remediation during promotion (via `/promote-skill`): softened the `carryover` references in the WRITE-side-only boundary section to an optional-companion framing ("a personal-handoff curator, such as a `carryover` companion skill, if installed"), since carryover stays incubating this wave and externalize functions without it (externalize is the WRITE side; carryover is one of the curate-side siblings). The `continue` and `work-router` references were left as-is (both published). This `HISTORY.md` was scrubbed for the public page (neutralized the author name).

Source-of-truth moved from the niftymonkey/claude repo to the niftymonkey/skills repo (skills/externalize/ + history/externalize.md).

## Design uncertainties

- It is a best-effort disposition, not a hard guarantee. Faithful capture needs the model's judgment about what was hard-won, which a hook cannot supply, so nothing mechanically forces externalization to happen at the right moment. Whether the disposition actually fires at the right times, unprompted, is unobserved; it is the same volunteering bet as work-router's, and unproven.
- Reliability is the open risk: a year of weak or absent disposition means the model historically did not volunteer this. Whether a strong standing reflex changes that, rather than the capture being skipped under task focus, is exactly what live use will show.
- Cold-start tension: `CLAUDE.md` currently has advisory sections (including the old work-router reflex) commented out for a 5-day cold-start experiment begun 2026-06-21. Adding the externalize reflex line collides with that experiment, so where and when it lands is the user's call, not assumed here.
- The earned-not-threshold trigger assumes the model can recognize the moment context is "earned." Where that boundary actually sits (and whether trivial findings get over-captured, bloating the handoff, or genuine ones get missed) needs calibration from real use.
- The split assumes continuous capture (externalize) plus on-demand curation (continue/carryover) is cleaner than the old combined skills. Whether the handoff produced by externalize is actually well-formed enough for continue/carryover to curate without re-capturing is unverified until the three are used together.

## Files

- `SKILL.md`: the continuous-write capture disposition (firing rule, what to write vs skip, target-file selection, the migrated capture-quality spine, the WRITE-side-only boundary). The capture sibling of work-router.
- `HISTORY.md`: this file.
