# Companion files

Shared content referenced by one or more skills. Each skill that needs a companion has a symlink pointing here (e.g., `skills/continue/TEMPLATE.md → ../../companion-files/TEMPLATE.md`). Edit a file here once and every skill referencing it picks up the change.

When `npx skills add` installs a skill in symlink mode, the symlink chain resolves through to the canonical content here. In `--copy` mode, the file is duplicated into each installing skill's directory — that's a fine outcome on the installer side; the authoring-side single-source-of-truth is what matters.

## Index

| File | Used by |
|---|---|
| `TEMPLATE.md` | `continue` — the markdown skeleton written into `continue.md` when the skill is invoked |
