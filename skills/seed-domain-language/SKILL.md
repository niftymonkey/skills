---
name: seed-domain-language
description: Seed a project's DOMAIN-LANGUAGE.md glossary from the vocabulary already in its codebase.
disable-model-invocation: true
---

<what-to-do>

Bootstrap a `DOMAIN-LANGUAGE.md` glossary for this repository by mining the domain
vocabulary already present in it, then refining it with the user. A one-time
(occasionally re-run) pass. Companion to `explore-idea`, which maintains the glossary
incrementally once it exists; this skill only creates the initial one.

If a `DOMAIN-LANGUAGE.md` / `DOMAIN-LANGUAGE-MAP.md` already exists, don't silently
overwrite it. Tell the user it exists, and ask whether to re-seed from scratch
(replacing it) or leave maintenance to `explore-idea`.

Run it as a sequence of stages, with the user in the loop at three gates.

## 1. Harvest

Determine the repo type first, then gather candidate vocabulary from the
highest-signal sources for that type (see Scan strategy). Collect raw candidates;
don't refine yet.

## 2. Context gate: confirm the partitioning

Judge whether the repo is single-context (one cohesive domain language) or
multi-context (distinct bounded contexts, e.g. a monorepo with clearly separate
packages). Surface your read and let the user confirm or correct it *before*
distilling:

> "This repo reads as single-context: one `DOMAIN-LANGUAGE.md` at the root."

or

> "This repo looks like N bounded contexts: X, Y, Z. Each gets its own
> `DOMAIN-LANGUAGE.md`, indexed by a root `DOMAIN-LANGUAGE-MAP.md`. Confirm or correct."

Never auto-decide multi-context; it reshapes everything downstream.

## 3. Distill

Turn raw candidates into an opinionated glossary:

- Collapse synonyms: when several words name one concept, pick the canonical term
  and list the rest as terms to avoid.
- Flag ambiguity: when one word is used for different concepts, call it out.
- Drop generic programming terms: only domain concepts belong; a term meaningful
  to a domain expert who doesn't know the code, not general engineering vocabulary.
- Cluster the survivors: by bounded context if multi-context, else by domain area.

Follow [DOMAIN-LANGUAGE-FORMAT.md](DOMAIN-LANGUAGE-FORMAT.md)
for what a good entry looks like.

## 4. Cluster gates: review per cluster

Present distilled terms **one cluster at a time**, not all at once, not term-by-term.
Per cluster, the user accepts, edits, or rejects entries. A wrong framing caught in
an early cluster saves correcting it everywhere downstream.

## 5. Final gate: confirm the whole draft

Assemble the full glossary and show it whole before writing anything to disk.

## 6. Write

On confirmation, write per
[DOMAIN-LANGUAGE-FORMAT.md](DOMAIN-LANGUAGE-FORMAT.md):

- Single-context → `DOMAIN-LANGUAGE.md` at the repo root.
- Multi-context → one `DOMAIN-LANGUAGE.md` per context plus a root `DOMAIN-LANGUAGE-MAP.md`.

Tell the user the glossary is live and that `explore-idea` maintains it from here.

</what-to-do>

<supporting-info>

## Scan strategy

Rank sources by how much domain signal they carry, and adapt to repo type.

**Code repositories**. Vocabulary lives in names:

- High signal: type / class / interface names, exported symbols, directory and
  module names, doc headings (`README`, `docs/`).
- Medium: function and method names (verbs and some nouns; noisier).
- Low: comments. Skip: local variables and parameters.

**Prose / documentation repositories** (knowledge base, skills or config repo).
Vocabulary lives in the writing:

- High signal: document headings, recurring **bolded** and `backticked` terms,
  defined-term patterns ("X is a…", "we call this…"), directory and file names.
- Skip: prose connective tissue.

If a repo mixes both, harvest from both and let the docs disambiguate the code.

## Relationship to explore-idea

`seed-domain-language` and `explore-idea` are independent skills sharing one
artifact and one contract:

- The contract is [DOMAIN-LANGUAGE-FORMAT.md](DOMAIN-LANGUAGE-FORMAT.md).
- This skill *creates* the initial `DOMAIN-LANGUAGE.md`; `explore-idea` *maintains*
  it incrementally.
- They never invoke each other; they integrate only through the file on disk.

`explore-idea`'s startup offer points users here when a repo has no glossary yet;
this skill points users back to `explore-idea` for ongoing maintenance.

</supporting-info>
