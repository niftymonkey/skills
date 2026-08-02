---
name: explore-idea
description: Interview the user one question at a time through every decision an idea implies, taking it from a raw just-thought-of-it notion to a written artifact: Problem/Opportunity, Target Users, Core Requirements, Key Decisions, Constraints, Open Questions, the outline of a PRD. Use whenever an idea should survive the conversation as a document, including when the user says "grill me" about one. Use grilling instead when you only need to think something through out loud and nothing needs writing down.
disable-model-invocation: true
---

<what-to-do>

## Before the interview: offer to ground the session in domain language

When the session is rooted in a code repository, check for a `DOMAIN-LANGUAGE.md` (or a `DOMAIN-LANGUAGE-MAP.md` at the root, for multi-context repos).

**If one exists:** read it. It is the agreed glossary for this project: use its terms, and challenge any user wording that conflicts with it.

**If none exists:** before grilling, surface this offer once:

> This project has no `DOMAIN-LANGUAGE.md`. A shared domain glossary is worth the small upfront cost: we use fewer words to mean the same things, my replies and thinking stay concise, and generated code is easier to navigate because its names match the language we agreed on. Three ways to go:
>
> 1. **Seed it now**: run the `seed-domain-language` skill, a companion that mines an initial glossary from the existing codebase for your review. Then we grill on a populated base. (Best for established, code-heavy repos.)
> 2. **Build it as we go**: we start grilling and the glossary grows as terms get resolved.
> 3. **Skip it**: no glossary this session; you can do this later.

If the user picks (3), drop the offer for this session and don't re-offer.

## The interview

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one.

- **Ask one question at a time.** Wait for the answer before continuing. Don't dump multi-question paragraphs.
- **Lead with your recommended answer for each question.** State your default and the reasoning, then ask whether to take it. Blank-slate questions are slower and weaker.
- **If a question can be answered by exploring the codebase, explore the codebase instead.** Don't ask the user what the code already shows.
- **Cross-reference assertions against the code.** When the user says how something works, check. If the code disagrees, surface the contradiction immediately: *"You said X, but the code does Y. Which is right?"*
- **Sharpen fuzzy language.** When a term is overloaded or vague, propose a precise canonical name. *"You said 'account': Customer or User? Those are different things."*
- **Stress-test with concrete scenarios.** When relationships between concepts come up, invent edge cases that force precision on the boundaries. *"What happens if the Order is partially fulfilled and the Customer cancels?"*
- **During endgame or dream-mapping, don't editorialize implementation complexity.** When the user is intentionally describing their ideal end state, capture each ambitious choice without "that's a lot of work" / "significant lift" / "near-frontier" caveats; defer feasibility to a separate phasing pass. Still surface a genuine logical contradiction between requirements (that is different from complexity, and the user needs to know).

</what-to-do>

<supporting-info>

## Domain language (DOMAIN-LANGUAGE.md / ADRs)

This skill can maintain a living domain glossary (`DOMAIN-LANGUAGE.md`) and architecture decision records (`docs/adr/`) inline as decisions crystallize. Format definitions: [DOMAIN-LANGUAGE-FORMAT.md](DOMAIN-LANGUAGE-FORMAT.md), [ADR-FORMAT.md](ADR-FORMAT.md).

`DOMAIN-LANGUAGE.md` is a glossary and nothing else. Keep it totally devoid of implementation details: it is not a spec, not a scratch pad, not a home for design decisions. Include only terms meaningful to someone who knows the domain but not the code. Implementation-shaping decisions belong in ADRs; the session's outcome belongs in the end-of-session artifact.

### File structure

Most repos have a single context, one `DOMAIN-LANGUAGE.md` at the root:

```
/
├── DOMAIN-LANGUAGE.md
├── docs/
│   └── adr/
│       ├── 0001-event-sourced-orders.md
│       └── 0002-postgres-for-write-model.md
└── src/
```

If a `DOMAIN-LANGUAGE-MAP.md` exists at the root, the repo has multiple contexts; the map points to where each lives:

```
/
├── DOMAIN-LANGUAGE-MAP.md
├── docs/
│   └── adr/                          ← system-wide decisions
├── src/
│   ├── ordering/
│   │   ├── DOMAIN-LANGUAGE.md
│   │   └── docs/adr/                 ← context-specific decisions
│   └── billing/
│       ├── DOMAIN-LANGUAGE.md
│       └── docs/adr/
```

### When to read

- If `DOMAIN-LANGUAGE-MAP.md` exists at the repo root, read it to find the relevant context (multi-context repo).
- If `DOMAIN-LANGUAGE.md` exists (root or context-scoped), read it. Challenge any user term that conflicts with the existing glossary: *"Your DOMAIN-LANGUAGE.md defines 'cancellation' as X, but you seem to mean Y. Which is it?"*
- If `docs/adr/` exists, read recent ADRs in the area being explored. Don't re-litigate accepted decisions; if a candidate contradicts one, mark it clearly.

### When to write: conservative mode

Do not lazily create `DOMAIN-LANGUAGE.md` or `docs/adr/` without consent.

- **The glossary** is normally settled by the startup offer above. If the user chose "seed it now" or "build it as we go," it is opted-in, so maintain it inline as terms resolve, without re-asking. If they chose "skip," don't re-offer this session. Only when the startup offer didn't run (e.g. a session not rooted in a repo) fall back to asking once, the first time terms get resolved: *"This conversation produced N resolved terms. Start a DOMAIN-LANGUAGE.md?"*
- **ADRs** are always offered mid-session, never up front, because an ADR-worthy decision can't be predicted before grilling. The first time one arises in a project with no `docs/adr/`, ask once: *"This decision is hard to reverse and would surprise a future reader without context. Record it as ADR-0001?"* If yes, create per [ADR-FORMAT.md](ADR-FORMAT.md). If no, drop and don't re-offer. Once opted in, record inline without re-asking.

### ADR offer gate

Only consider offering an ADR when all three are true:

1. **Hard to reverse**: the cost of changing your mind later is meaningful.
2. **Surprising without context**: a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off**: there were genuine alternatives and you picked one for specific reasons.

If any are missing, don't offer. See [ADR-FORMAT.md](ADR-FORMAT.md) for what qualifies.

## End-of-session artifact

When the interview reaches natural completion, generate a structured markdown summary covering: Problem/Opportunity, Target Users, Core Requirements (must-have vs nice-to-have), Key Decisions Made, Constraints/Boundaries, and Open Questions. Skip or rename sections that don't apply.

Show me the draft for review before saving. Default save location is `./docs/` in the current repo; ask whether that's where I want it, or somewhere else. If the chosen directory doesn't exist, confirm before creating it.

</supporting-info>
