# Companion files

Shared content referenced by one or more skills. Each skill that needs a companion has a symlink pointing here (e.g., a hypothetical `skills/<name>/SOMETHING.md → ../../companion-files/SOMETHING.md`). Edit a file here once and every skill referencing it picks up the change.

When `npx skills add` installs a skill in symlink mode, the symlink chain resolves through to the canonical content here. In `--copy` mode, the file is duplicated into each installing skill's directory. That is a fine outcome on the installer side; the authoring-side single-source-of-truth is what matters.

## Index

| File | Used by |
|---|---|
| `DOMAIN-LANGUAGE-FORMAT.md` | `explore-idea` (format of the `DOMAIN-LANGUAGE.md` glossary it maintains). Adapted from Pocock's `grill-with-docs`, MIT. Shared with `seed-domain-language` once that skill is promoted. |
