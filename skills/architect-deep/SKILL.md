---
name: architect-deep
description: Sketch a new module, feature, or project's architecture using deep-module thinking before any code is written. Apply proactively any time you are proposing module structure, naming new boundaries, classifying dependencies, or designing interfaces for new functionality. Runs standalone for ad-hoc design, and composes with PRD and planning workflows (populating their architecture sections) when you use them.
---

# Architect Deep

The proactive companion to Pocock's `improve-codebase-architecture`. That skill finds shallow modules in code that already exists. This skill prevents them. It applies the same lens at design time, before commitments are made.

## Vocabulary

Use the terms in [LANGUAGE.md](LANGUAGE.md) exactly. **Module**, **interface**, **implementation**, **depth**, **seam**, **adapter**, **leverage**, **locality**. Don't drift into "component," "service," "API," or "boundary." If the user uses different language, translate to the canonical terms in your output.

Key principles repeated here so they fire at design time:

- **Pre-deletion test**: *before* sketching a module, ask: "If I never built this, would complexity vanish or concentrate across N callers?" Vanish = don't build it. Concentrate = build it deep.
- **The interface is the test surface.** Design the interface as the thing tests will exercise.
- **One adapter = hypothetical seam. Two adapters = real seam.** Don't introduce a port unless production + test (or two real production variants) actually need to differ.
- **Depth is leverage at the interface, not lines-of-implementation per line-of-interface.** A small body behind a small interface can still be deep if it earns its keep across many callers.

## When to fire

Fire when a new boundary is being drawn:

- Naming a new module, package, slice, or service.
- Sketching the major modules section of a PRD (write-prd step 4).
- Locking durable architectural decisions for a plan (prd-to-plan step 3).
- Late in explore-idea, when concrete modules start surfacing.
- Any ad-hoc design conversation where the user is choosing structure.

Do **not** fire on:

- Function-level decisions inside an existing module.
- Refactoring code that already exists: that's `improve-codebase-architecture`.
- Pure phasing/sequencing decisions (prd-to-issues).

## Process

### 1. Gather input

Pull the functionality the design must support. Sources, in priority order:

1. The PRD draft if write-prd is active.
2. The explore-idea output file if one exists.
3. The conversation directly if neither exists.

If a `CONTEXT.md` exists in the project, read it for domain language. If absent, skip silently; pull domain terms from the PRD/explore-idea/conversation, or ask the user a single short question for any ambiguous term. Do not require CONTEXT.md to exist.

### 2. List candidate modules

From the gathered input, list every module the design seems to require. Name each one in the project's domain language. Avoid generic names like `Service`, `Manager`, `Handler`, `Util`.

### 3. Apply the pre-deletion test to each candidate

For each candidate, ask:

> "If this module never existed, would complexity vanish (it's a pass-through) or concentrate across N callers (it earns depth)?"

- **Vanish**: drop the candidate. Inline the work into its caller. Note the decision so a future reviewer doesn't re-suggest it.
- **Concentrate**: keep the candidate. Note *what* concentrates there. That description is the seed of the interface.

Present the surviving candidates to the user before continuing.

### 4. Classify each dependency

For each surviving module, identify its dependencies and classify each into one of the four categories from [DEEPENING.md](DEEPENING.md):

1. **In-process**: pure computation, no I/O. No port needed.
2. **Local-substitutable**: has a local stand-in (PGLite, in-memory FS). Internal seam only; no port at the external interface.
3. **Remote but owned**: your own services across a network. Port at the seam, transport adapter injected.
4. **True external**: third-party (Stripe, Twilio, etc). Port at the seam, mock adapter for tests.

The category determines whether a port belongs at the module's external interface. Categories 1 and 2 do not justify external ports. Categories 3 and 4 do.

### 5. Design the interface (twice, for non-trivial seams)

For each module whose interface is non-trivial (it sits at a category-3 or category-4 seam, or has more than ~3 entry points, or the design has obvious tension), apply the parallel sub-agent process in [INTERFACE-DESIGN.md](INTERFACE-DESIGN.md). "Design It Twice" (Ousterhout): the first idea is unlikely to be the best.

For trivial in-process modules with one obvious shape, skip step 5 and propose a single interface inline.

### 6. Present in leverage / locality terms

For each module, state:

- **Interface**: entry points, invariants, ordering, error modes, required configuration.
- **What it hides**: the concentrated complexity behind the seam.
- **Leverage**: what callers gain (one body of logic across N call sites and M tests).
- **Locality**: what maintainers gain (where change, bugs, and verification will land).
- **Dependency category** and the chosen seam strategy.
- **Test surface**: how tests cross the interface; what is *not* tested past it.

Avoid file paths and code snippets; those date fast. Stay at the module-shape level.

### 7. Output

Architect-deep does **not** write its own document. Output flows into the calling artifact:

| Caller | Lands in |
|---|---|
| `write-prd` | PRD section: **Implementation Decisions** |
| `prd-to-plan` | Plan header: **Architectural decisions** |
| `explore-idea` | Inline only (idea still too fluid for hard module commitments) |
| Ad-hoc | Inline; if the user says "save it," write to `./docs/architecture/<thing>.md` |

This avoids document sprawl. The PRD and plan stay the source of truth.

## Conflict handling

If the user's stated preference contradicts a recommendation (e.g. they want one module split that the deletion test says should stay merged), surface the trade-off in `leverage`/`locality` terms. Don't override silently. Don't comply silently either: make the cost visible.

If a project has ADRs and a candidate contradicts one, only surface it when the friction is real enough to warrant revisiting the ADR. Mark clearly: *"contradicts ADR-NNNN, but worth reopening because…"* Skip if the ADR is settled and the recommendation is marginal.

## Relationship to other skills

These are optional companions, not requirements. Architect-deep runs standalone for ad-hoc design; when you use the broader planning workflow, it slots in as follows.

- **explore-idea** (optional companion): runs first. Fluid. Architect-deep stays inline if invoked here.
- **write-prd** (optional companion): primary insertion point. Architect-deep populates Implementation Decisions in step 4.
- **prd-to-plan** (optional companion): fires architect-deep in step 3 only if Implementation Decisions in the PRD weren't produced via architect-deep.
- **prd-to-issues**: does not invoke architect-deep. Pure phasing.
- **improve-codebase-architecture** (Pocock, mattpocock/skills): the reactive sibling. Runs against existing code. Same vocabulary, same principles, opposite direction.
