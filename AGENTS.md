# Instructions for Codex or Other Coding Agents

This repository defines a dataset specification, not application code.

## Goals

- Keep the specification precise, minimal, and machine-parseable.
- Prefer normative language: MUST, SHOULD, MAY, MUST NOT.
- Keep `metadata.toml` as the source of truth for machine-readable details.
- Keep `description.toml` optional and human-facing.
- Keep `dataset.lance/` as the default main data table.

## Editing rules

When editing the spec:

1. Update the relevant document under `rules/`.
2. Update examples under `examples/` when TOML structure changes.
3. Update `validation/validation-checklist.md` when required fields change.
4. Update `CHANGELOG.md` for all meaningful changes.
5. Avoid breaking existing examples unless intentionally changing the spec.

## Style

- Use concise technical language.
- Use tables for field requirements.
- Use TOML code blocks for examples.
- Do not introduce multiple Lance tables as the default unless the spec explicitly changes.
- Do not move schema details from `metadata.toml` into `description.toml`.

