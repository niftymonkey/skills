# Companion files

Shared content referenced by one or more skills. Each skill that needs a companion has a symlink pointing here (e.g., a hypothetical `skills/<name>/SOMETHING.md → ../../companion-files/SOMETHING.md`). Edit a file here once and every skill referencing it picks up the change.

When `npx skills add` installs a skill in symlink mode, the symlink chain resolves through to the canonical content here. In `--copy` mode, the file is duplicated into each installing skill's directory — that's a fine outcome on the installer side; the authoring-side single-source-of-truth is what matters.

## Index

*(no companions currently — `continue`'s `TEMPLATE.md` was the first member and was dropped in `continue` v4 when the skill moved to a no-template, trust-the-model handoff shape. Future shared content from subsequent skill promotions will land here.)*
