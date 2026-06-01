# Validation Checklist

Use this checklist before accepting a dataset into the platform.

Validation MUST run on the local processing server against the final dataset directory before upload. Object storage upload MUST NOT start unless every required check passes.

If any required check fails, the dataset MUST be rejected for upload. Any change to files after validation MUST trigger a new validation run.

Unless an item is explicitly marked optional or recommended, every checklist item is required and blocking.

## Upload gate

- [ ] Validation is run against the exact dataset directory intended for upload.
- [ ] `dataset_id` and `dataset.version` in `metadata.toml` match the upload target.
- [ ] No files are modified after validation and before upload.
- [ ] All required checks in this checklist pass.
- [ ] A validation result is recorded with pass/fail status, dataset ID, dataset version, row count, and failed checks if any.

## File existence

- [ ] `dataset.lance/` exists, unless `dataset.task_type = "pretraining"` and all signal tables are declared in `[[pretrain.tables]]`.
- [ ] If `dataset.task_type = "pretraining"` and `dataset.lance/` is absent, at least one `[[pretrain.tables]]` entry with `role = "signal"` exists.
- [ ] The canonical signal table can be opened as a Lance dataset. For partitioned pretraining datasets, every declared signal table can be opened.
- [ ] The canonical signal table contains at least one row. For partitioned pretraining datasets, the union of declared signal tables contains at least one row.
- [ ] `metadata.toml` exists.
- [ ] `versions.toml` exists or the dataset is explicitly marked as not version-tracked.
- [ ] `description.toml` is optional.

## Metadata checks

- [ ] `metadata.toml` is valid TOML and can be parsed without errors.
- [ ] `spec_version` exists.
- [ ] `[dataset]` exists.
- [ ] `[storage]` exists.
- [ ] `[schema]` exists.
- [ ] At least one `[[schema.columns]]` entry exists.
- [ ] `[data]` exists.
- [ ] `[quality_control]` exists.
- [ ] Required modality-specific section exists.

## Dataset section

- [ ] `dataset.dataset_id` exists.
- [ ] `dataset.name` exists.
- [ ] `dataset.modality` exists.
- [ ] `dataset.version` exists.
- [ ] `dataset.created_at` exists.
- [ ] `dataset.updated_at` exists.
- [ ] `dataset.sample_unit` exists.
- [ ] `dataset.n_samples` exists.
- [ ] `dataset.n_samples` is positive.
- [ ] Lance row count equals `dataset.n_samples`. For partitioned pretraining datasets, the sum of declared signal table row counts equals `dataset.n_samples`.

## Storage section

- [ ] `storage.table_name` exists.
- [ ] `storage.lance_path` exists.
- [ ] `storage.path_type` is `relative` or `absolute`.
- [ ] `storage.lance_path` resolves to `dataset.lance/` or another declared Lance path. For partitioned pretraining datasets, `storage.table_name` and `storage.lance_path` MUST both be `__partitioned__`, and readers MUST resolve signal tables from `[[pretrain.tables]]`.

## Lance schema checks

- [ ] Every required base column exists in the canonical signal table. For partitioned pretraining datasets, every declared signal table contains these physical columns: `sample_id`, `subject_id`, `modality`, `data`, `shape`, `original_shape`, `valid_length`, and `qc_pass`.
- [ ] Required base columns contain no null values.
- [ ] All Lance columns are declared in `[[schema.columns]]`.
- [ ] All declared required columns exist in Lance.
- [ ] Types in `metadata.toml` match Lance schema.
- [ ] `sample_id` values are non-empty and unique.
- [ ] `subject_id` values are non-empty.
- [ ] `modality` values match `dataset.modality`.
- [ ] `qc_pass` is boolean and non-null for every row.

## Data restoration checks

- [ ] `data` values are present and non-empty for every row.
- [ ] `shape` and `original_shape` contain positive integer dimensions.
- [ ] `data` length matches the product of `shape`.
- [ ] `valid_length` is positive and does not exceed the product of `shape`.
- [ ] `data` can be reshaped to `shape` for every row.
- [ ] `original_shape` can be used to crop or slice the padded array.
- [ ] `valid_length` matches the product of `original_shape`, unless explicitly declared otherwise.
- [ ] `axis_order` length matches shape rank.

## Label checks

- [ ] Supervised datasets contain `[label]`.
- [ ] Supervised datasets contain a `label` column.
- [ ] Supervised datasets have non-null `label` values for every released row.
- [ ] Classification datasets contain `[[label.classes]]`.
- [ ] `label` values are valid class IDs for classification tasks.
- [ ] `label_name` matches `label.classes`, if present.

## Split checks

- [ ] If `split` is present, every value is declared in metadata or uses an accepted platform split value.

## EEG checks

- [ ] `[eeg]` exists when `dataset.modality = "eeg"`.
- [ ] `eeg.channel_layout` exists and is one of `dataset_level`, `per_subject`, or `per_sample`.
- [ ] If `eeg.channel_layout = "dataset_level"`, `eeg.n_channels` equals the number of `[[eeg.channels]]` entries.
- [ ] If `[[eeg.channels]]` is present, channel indexes are contiguous and start from 0.
- [ ] If `eeg.channel_layout` is `per_subject` or `per_sample`, `eeg.channel_names_column` exists and names a Lance column.
- [ ] If `eeg.channel_status_column` is declared, it names a Lance column aligned with `eeg.channel_names_column`.
- [ ] If `eeg.channel_mask_column` is declared, it names a Lance column aligned with the dataset-level channel universe or `eeg.channel_names_column`.
- [ ] For `per_subject` or `per_sample`, each `channel_names_column` value is non-empty and matches the restored channel-axis order.
- [ ] For `per_subject` or `per_sample`, `eeg.n_channels` is greater than or equal to every row's restored channel-axis length.
- [ ] `eeg.sampling_rate` is positive.
- [ ] `eeg.channel_axis` and `eeg.time_axis` are valid axes.

## Pretraining checks

- [ ] `[pretrain]` exists when `dataset.task_type = "pretraining"`.
- [ ] `pretrain.pretrain_type` exists and is one of `self_supervised`, `supervised`, `multi_objective`, or `contrastive`.
- [ ] `pretrain.sample_id_scope` exists and is one of `global` or `table_local`.
- [ ] If `pretrain.sample_id_scope = "table_local"`, `pretrain.sample_key_columns` exists and is `["table_id", "sample_id"]`, unless an explicitly documented equivalent key is declared.
- [ ] If `pretrain.sample_id_scope = "table_local"`, `schema.primary_key` is `["table_id", "sample_id"]`.
- [ ] `pretrain.source_dataset_column` names an existing physical signal-table column.
- [ ] `pretrain.recording_id_column` names an existing physical signal-table column.
- [ ] `pretrain.preprocess_version` exists.
- [ ] `pretrain.split_policy` exists.
- [ ] `pretrain.split_policy` is one of `subject_disjoint`, `recording_disjoint`, `source_dataset_disjoint`, or `custom_declared`.
- [ ] If `pretrain.split_policy = "custom_declared"`, `pretrain.split_declaration_path` exists and resolves to a split policy file.
- [ ] At least one `[[pretrain.sources]]` entry exists.
- [ ] Each `[[pretrain.sources]]` entry has a unique `source_dataset_id`.
- [ ] Each source declares `source_family`, `version`, `license`, `access_level`, and `n_samples`.
- [ ] Each source `access_level` is one of `open`, `registered`, `restricted`, `internal`, or `unknown`.
- [ ] If `[[pretrain.tables]]` is present, every table has `table_id`, `role`, `lance_path`, and `n_samples`.
- [ ] Every declared `[[pretrain.tables]].table_id` is unique inside the dataset.
- [ ] Every declared pretraining table path resolves and row count matches `n_samples`.
- [ ] If `pretrain.sample_id_scope = "global"`, `sample_id` values are globally unique across all declared signal tables.
- [ ] If `pretrain.sample_id_scope = "table_local"`, `sample_id` values are unique within each declared signal table, and the global sample key is `(table_id, sample_id)`.
- [ ] If a physical `table_id` column exists in a signal table, every value matches the enclosing `[[pretrain.tables]].table_id`.
- [ ] If `pretrain.sample_uid_column` is declared, the named column is either present and globally unique or explicitly documented as a derived reader field.
- [ ] Signal table rows contain physical columns for `source_dataset_id`, `recording_id`, `source_path`, `start_time`, `duration`, `split`, and `preprocess_version`.
- [ ] EEG pretraining signal rows contain a physical `channel_profile` column.
- [ ] Every `channel_profile` value is declared in `[[pretrain.channel_profiles]]` when channel profiles are used.
- [ ] Every declared `[[pretrain.channel_profiles]]` layout is one of `fixed_slot`, `variable`, or `named_only`.
- [ ] Feature tables declare `view_id`, `aligned_to`, and `alignment`, and are aligned to signal tables by `sample_id`, `sample_key`, `sample_uid`, or explicit `row_index`.
- [ ] Feature tables using `alignment = "sample_id"` contain `sample_id`.
- [ ] Feature tables using `alignment = "sample_key"` contain `signal_table_id` and `sample_id`.
- [ ] Feature tables using `alignment = "sample_uid"` contain the declared `sample_uid` column.
- [ ] Feature tables using `alignment = "row_index"` contain `row_index`.
- [ ] Under `sample_id_scope = "table_local"`, feature tables using `alignment = "sample_id"` declare exactly one `aligned_to` signal table.
- [ ] Sampling policy is declared when source distributions are intentionally reweighted.
- [ ] If `[pretrain.sampling].weight_column` is declared, the named column exists in every signal table used by the sampler.

## MRI checks

- [ ] `[mri]` exists when `dataset.modality = "mri"`.
- [ ] `mri.voxel_size` contains 3 positive values.
- [ ] `mri.spatial_axes` contains 3 axes.
- [ ] `mri.field_strength` is positive or explicitly unknown by policy.
- [ ] If `mri.affine_column` is not empty, the column exists in Lance.

## fMRI checks

- [ ] `[fmri]` exists when `dataset.modality = "fmri"`.
- [ ] `fmri.tr` is positive.
- [ ] `fmri.voxel_size` contains 3 positive values.
- [ ] `fmri.time_axis` is valid.
- [ ] Preprocessing status fields exist.

## Pre-upload object storage checks

- [ ] The upload payload contains the validated `dataset.lance/`, `metadata.toml`, and `versions.toml` unless version tracking is explicitly disabled.
- [ ] Relative paths in TOML files resolve inside the dataset directory.
- [ ] Absolute local paths are not used in uploaded metadata unless explicitly allowed by platform policy.
- [ ] Object storage destination is derived from `dataset_id` and `dataset.version`.
- [ ] Upload is blocked when validation status is not `pass`.
