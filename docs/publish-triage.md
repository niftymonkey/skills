# Skill publish triage

Triage of the regularly-used skills from [niftymonkey/regimen#2](https://github.com/niftymonkey/regimen/issues/2) ("Finish and publish the skill library") into two buckets: skills sourced from someone else's repo (install from upstream, do not republish) and skills authored locally (candidates for `promote-skill` into this repo).

## How provenance was determined

Authoritative, not by recollection:

- Locally-authored skills live in `niftymonkey/claude/skills/<name>/` as real content (most carry a `HISTORY.md` documenting their genesis).
- External skills are installed by the `skills` CLI into `~/.agents/skills/<name>/` and appear in the incubation dir only as a symlink pointing back at that install.
- `~/.agents/.skill-lock.json` records the exact upstream repo and path for each installed skill.

## Usage data caveat

Invocation counts come from regimen#2's prioritization comment: roughly 16 days of this machine's Claude Code transcripts. They are Claude-Code-only and approximate. The Feedback evidence layer logged the Skill tool 132 times but dropped the skill name at capture, so it could not rank by identity (a separate ticket covers that fix). Treat the ordering as directional.

## Section A: External (install from a separate repo)

Not yours to publish. The README should document these as "install from upstream" and point at their source repos, installed via `npx skills@latest add`. Ordered by usage.

| Skill | Uses | Upstream repo | Upstream path |
|---|---|---|---|
| tdd | 33 | `mattpocock/skills` | `skills/engineering/tdd` |
| prd-to-plan | 3 | `mattpocock/skills` | `prd-to-plan` |
| setup-pre-commit | 2 | `mattpocock/skills` | `setup-pre-commit` |
| diagnose | 2 | `mattpocock/skills` | `skills/engineering/diagnose` |
| brainstorming | 2 | `obra/superpowers` | `skills/brainstorming` |
| prd-to-issues | 1 | `mattpocock/skills` | `prd-to-issues` |
| writing-clearly-and-concisely | 1 | `softaworks/agent-toolkit` | `skills/writing-clearly-and-concisely` |

Three upstream sources in total: `mattpocock/skills` (five skills), `obra/superpowers` (one), `softaworks/agent-toolkit` (one).

## Section B: Yours (candidates for `promote-skill`)

The real targets for the publish work. Each should be run through `promote-skill` to move it from the `niftymonkey/claude` incubation repo into this published repo, with a portability pass for the noted dependencies. Ordered by usage.

| Skill | Uses | Note |
|---|---|---|
| architect-deep | 32 | Adapts Pocock MIT companion files (`LANGUAGE.md`, `DEEPENING.md`, `INTERFACE-DESIGN.md`); attribution already inlined. |
| md-niftymonkey | 7 | Hard dependency on `md.niftymonkey.dev` (your service). Publishable, but ties adopters to that host. |
| consult-radar | 7 | Hardcoded `git@github.com:niftymonkey/tool-radar.git`. Needs a portability pass before publishing. |
| kickoff | 2 | Fully original. (Initially suspected external; it is not.) |
| write-prd | 1 | Fully original. |
| explore-idea | 1 | Adapts Pocock's `grill-with-docs` format specs (ADR / domain-language); attribution already inlined. |
| seed-domain-language | 1 | Fully original. |
| feedback-evidence | 1 | Special case: a Codex copy already ships inside `regimen-feedback`. Depends on the `feedback` CLI. Not yet in this repo. |

## Already published in this repo

`continue` (11 uses) and `pr-feedback` (1 use) are already in `niftymonkey/skills` and are reinstalled back through the `skills` CLI (dogfooding the published repo).

## Summary

Of the 15 regularly-used skills not yet published: 8 are yours (publish candidates), 7 are external (install from upstream).
