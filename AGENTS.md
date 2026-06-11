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

## Pretrain profile

A sibling profile exists for self-supervised EEG pretraining (`rules/pretrain-rules.md`, `rules/pretrain-rules.cn.md`). When editing it, preserve these invariants:

- A pretrain dataset sets `profile = "pretrain"` and `dataset.task_type = "pretrain"`.
- No `[label]` section and no `label` columns; `[pretrain]` replaces `[label]`.
- One table per dataset; do not reintroduce channel-count bucket tables. Train/val use the `split` column, not separate tables.
- `electrode_ids` and `channel_names` are both required and aligned, over a shared electrode vocab (ids `0`=pad, `1`=UNK_EEG).
- Channel padding is slot-mask based; `valid_length` is declared as `n_valid_channels * T`.
- NPD features live in `derivatives/features/npd.lance`, row-aligned by `sample_id`.
- Keep the pretrain profile additive: do not rewrite the base rule documents to assume pretrain semantics.

