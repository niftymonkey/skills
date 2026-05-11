# niftymonkey/skills

Custom agent skills, promoted from incubation in [niftymonkey/claude](https://github.com/niftymonkey/claude) once they've passed a portability review.

## Install

These skills work with any agent supported by the [`skills` CLI](https://github.com/vercel-labs/skills) — Claude Code, Codex, Cursor, and ~50 more.

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

- [**continue**](./skills/continue/SKILL.md) — Capture full session state to a `continue.md` file so a brand-new conversation can resume exactly where this one ended. Useful when the context window is filling up, before stopping mid-task, or any time you want a clean hand-off to a fresh session. ([history](./history/continue.md))
- [**pr-feedback**](./skills/pr-feedback/SKILL.md) — Address code review comments on a GitHub PR systematically. Fixes code, pushes changes, and responds inline to every comment with either the fix or specific reasoning for dismissing. Uses `gh` CLI. ([history](./history/pr-feedback.md))

## How a skill arrives here

Each skill starts in [niftymonkey/claude](https://github.com/niftymonkey/claude) at status `incubating`. When it's matured and proven useful, it goes through a `/promote-skill` review — automated checks for personal paths, credentials, and domain references, plus a collaborative pass on implicit assumptions a stranger wouldn't share. Only skills that pass land here.

The full design history for every skill is preserved in [`history/`](./history/). It's deliberately kept outside the `skills/` tree so `npx skills add` ships only runtime content — installs stay lean while the archival record stays public.

## License

MIT — see [LICENSE](./LICENSE).

Some skills include content adapted from [mattpocock/skills](https://github.com/mattpocock/skills) (also MIT). Adapted files carry a visible attribution line at the top of the file.
