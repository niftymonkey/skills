# niftymonkey/skills

Custom agent skills, promoted from incubation in [niftymonkey/claude](https://github.com/niftymonkey/claude) once they've passed a portability review.

## Install

These skills work with any agent supported by the [`skills` CLI](https://github.com/vercel-labs/skills): Claude Code, Codex, Cursor, and ~50 more.

```bash
# Browse what's available before installing
npx skills@latest add niftymonkey/skills --list

# Install everything
npx skills@latest add niftymonkey/skills

# Install a single skill
npx skills@latest add niftymonkey/skills --skill <name>
```

By default the CLI installs at project scope (`./<agent>/skills/`). Use `-g` for global (`~/<agent>/skills/`).

## Skills

- [**architect-deep**](./skills/architect-deep/SKILL.md): Sketch a new module, feature, or project's architecture with deep-module thinking before any code is written, so shallow modules don't get baked in at design time. Runs standalone or composes with PRD and planning workflows. Includes MIT companion files adapted from `improve-codebase-architecture`. ([history](./history/architect-deep.md))
- [**continue**](./skills/continue/SKILL.md): Capture full session state to a `continue.md` file so a brand-new conversation can resume exactly where this one ended. Useful when the context window is filling up, before stopping mid-task, or any time you want a clean hand-off to a fresh session. ([history](./history/continue.md))
- [**explore-idea**](./skills/explore-idea/SKILL.md): Stress-test a plan, design, or idea through a relentless one-question-at-a-time interview, and optionally maintain a `DOMAIN-LANGUAGE.md` glossary and ADRs inline as decisions crystallize. Includes MIT companion files adapted from `grill-with-docs`. ([history](./history/explore-idea.md))
- [**externalize**](./skills/externalize/SKILL.md): Continuously write hard-won, expensive, non-reproducible context to your work-thread handoff file the instant it's earned, so it can't die in a full context window or get flattened by compaction. The continuous-write capture sibling of work-router; feeds on-demand curate-side handoff skills like continue. ([history](./history/externalize.md))
- [**fork**](./skills/fork/SKILL.md): Branch a newly-surfaced task off the current conversation into a single-use handoff document, so a separate agent can run it independently while this session stays focused. The lateral counterpart to continue (which resumes the same task). MIT, adapted from Pocock's `handoff`. ([history](./history/fork.md))
- [**pr-feedback**](./skills/pr-feedback/SKILL.md): Address code review comments on a GitHub PR systematically. Fixes code, pushes changes, and responds inline to every comment with either the fix or specific reasoning for dismissing. Uses `gh` CLI. ([history](./history/pr-feedback.md))
- [**seed-domain-language**](./skills/seed-domain-language/SKILL.md): Seed a project's `DOMAIN-LANGUAGE.md` glossary by mining the domain vocabulary already in its codebase, then refining it with you across context, per-cluster, and final review gates. A one-time bootstrap; companion to explore-idea, which maintains the glossary from there. Includes an MIT companion file adapted from `grill-with-docs`. ([history](./history/seed-domain-language.md))
- [**work-router**](./skills/work-router/SKILL.md): Decide, for each unit of work, whether it stays in the main thread or routes off-thread, and by which mechanism, so only human-in-the-loop work stays resident while the harness's parallelism does the rest. A portable policy core plus per-harness mechanism adapters for Claude Code and Codex. ([history](./history/work-router.md))
- [**write-prd**](./skills/write-prd/SKILL.md): Create a PRD through a relentless interview, codebase exploration, and deliberate module design, saved as a local markdown file with optional GitHub-issue submission. Works for new projects, features, or refactors. Composes with explore-idea (optional input) and architect-deep (optional module-design step), both in this repo, and runs standalone without them. ([history](./history/write-prd.md))

## How a skill arrives here

Each skill starts in [niftymonkey/claude](https://github.com/niftymonkey/claude) at status `incubating`. When it's matured and proven useful, it goes through a `/promote-skill` review: automated checks for personal paths, credentials, and domain references, plus a collaborative pass on implicit assumptions a stranger wouldn't share. Only skills that pass land here.

The full design history for every skill is preserved in [`history/`](./history/). It's deliberately kept outside the `skills/` tree so `npx skills add` ships only runtime content; installs stay lean while the archival record stays public.

## License

MIT. See [LICENSE](./LICENSE).

Some skills include content adapted from [mattpocock/skills](https://github.com/mattpocock/skills) (also MIT). Adapted files carry a visible attribution line at the top of the file.
