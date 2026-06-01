# Changelog

## Unreleased

- Adds pretraining dataset extension rules with multi-source metadata, partitioned table declarations, feature views, channel profiles, split policy, and sampling policy.
- Adds EEG pretraining metadata, version, and description examples plus a metadata template.
- Tightens pretraining partitioned storage, physical-column requirements, split policy enums, feature alignment, and row-level version semantics after external CLI review.
- Adds table-local pretraining sample keys: partitioned datasets may use `(table_id, sample_id)` as the authoritative global sample key, with optional materialized `sample_uid`.
- Adds Chinese translations for all rule documents under `rules/`.
- Adds README navigation links for English and Chinese rule documents.
- Adds blocking local validation requirements before object storage upload.
- Consolidates hard required validation checks into the existing checklist sections.
- Adds EEG channel layout rules for dataset-level, per-subject, and per-sample channel metadata.
- Adds Lance column conventions for EEG `channel_names`, `channel_status`, and `channel_mask`.

## 0.1.0

Initial specification draft.

- Defines default dataset layout.
- Defines `dataset.lance/` main table rules.
- Defines required `metadata.toml` sections.
- Defines optional `description.toml` purpose.
- Defines recommended `versions.toml` structure.
- Adds EEG, MRI, and fMRI metadata requirements.
- Adds validation checklist.
