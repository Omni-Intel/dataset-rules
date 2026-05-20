# Validation Checklist

Use this checklist before accepting a dataset into the platform.

## File existence

- [ ] `dataset.lance/` exists.
- [ ] `metadata.toml` exists.
- [ ] `versions.toml` exists or the dataset is explicitly marked as not version-tracked.
- [ ] `description.toml` is optional.

## Metadata checks

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

## Storage section

- [ ] `storage.table_name` exists.
- [ ] `storage.lance_path` exists.
- [ ] `storage.path_type` is `relative` or `absolute`.
- [ ] `storage.lance_path` resolves to `dataset.lance/` or another declared Lance path.

## Lance schema checks

- [ ] All Lance columns are declared in `[[schema.columns]]`.
- [ ] All declared required columns exist in Lance.
- [ ] Types in `metadata.toml` match Lance schema.
- [ ] `sample_id` is unique.
- [ ] `qc_pass` is boolean.
- [ ] `shape`, `original_shape`, and `valid_length` exist.

## Data restoration checks

- [ ] `data` can be reshaped to `shape` for sampled rows.
- [ ] `original_shape` can be used to crop or slice the padded array.
- [ ] `valid_length` matches the product of `original_shape`, unless explicitly declared otherwise.
- [ ] `axis_order` length matches shape rank.

## Label checks

- [ ] Supervised datasets contain `[label]`.
- [ ] Classification datasets contain `[[label.classes]]`.
- [ ] `label` values are valid class IDs for classification tasks.
- [ ] `label_name` matches `label.classes`, if present.

## EEG checks

- [ ] `[eeg]` exists when `dataset.modality = "eeg"`.
- [ ] `eeg.n_channels` equals the number of `[[eeg.channels]]` entries.
- [ ] Channel indexes are contiguous and start from 0.
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

