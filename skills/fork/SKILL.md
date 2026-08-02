---
name: fork
description: Branch a newly-surfaced task off the current conversation into a handoff document for a separate agent to pick up.
argument-hint: "What will the next session be used for?"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

> Adapted from [mattpocock/skills](https://github.com/mattpocock/skills), MIT License ([LICENSE](https://github.com/mattpocock/skills/blob/main/LICENSE)). Source: [skills/productivity/handoff](https://github.com/mattpocock/skills/blob/main/skills/productivity/handoff/SKILL.md).

Write a handoff document that branches a single task off the current conversation, so a separate agent can run it independently while this conversation stays focused on its own work. Save it to `./fork-<slug>.md` in the working directory, where `<slug>` is a short kebab-case label for the task. The file is personal session state: if `fork-*.md` is not already gitignored, add it to `.git/info/exclude`.

Carry forward everything the forked task needs, and nothing else. The filter is relevance to the fork's stated purpose, not recency in the conversation; what passes travels verbatim, not summarized. Make sure the document carries each of these that exists:

- **The task.** What the forked session must do, the realization that surfaced it, and anything about it still unresolved.
- **Why it surfaced.** The connection in the current work that made this task necessary. This is the single most perishable thing in the document: the current session knows it, a cold agent cannot reconstruct it.
- **Hard-won understanding.** Research findings, with sources and exact specifics; and the connections the conversation worked out between modules, systems, or pieces of context. These are the costliest things for a cold agent to rederive, so they go in full, never as a pointer to "we figured something out".
- **What's already settled.** Decisions and constraints the forked task must respect, and approaches already ruled out, with the reason.
- **Orientation.** Where to start in the repo: the relevant files, modules, and docs.

Include a "suggested skills" section in the document, which suggests skills that the agent should invoke.

Do not duplicate content already captured in other artifacts (PRDs, plans, ADRs, issues, commits, diffs). Reference them by path or URL instead.

Redact any sensitive information, such as API keys, passwords, or personally identifiable information.

If the user passed arguments, treat them as a description of what the next session will focus on and tailor the doc accordingly.

The document is single-use: delete it once the forked agent has picked up the task.
