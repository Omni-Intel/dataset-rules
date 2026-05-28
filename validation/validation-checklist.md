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

- [ ] `dataset.lance/` exists.
- [ ] `dataset.lance/` can be opened as a Lance dataset.
- [ ] `dataset.lance/` contains at least one row.
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
- [ ] Lance row count equals `dataset.n_samples`.

## Storage section

- [ ] `storage.table_name` exists.
- [ ] `storage.lance_path` exists.
- [ ] `storage.path_type` is `relative` or `absolute`.
- [ ] `storage.lance_path` resolves to `dataset.lance/` or another declared Lance path.

## Lance schema checks

- [ ] Every required base column exists: `sample_id`, `subject_id`, `modality`, `data`, `shape`, `original_shape`, `valid_length`, and `qc_pass`.
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
