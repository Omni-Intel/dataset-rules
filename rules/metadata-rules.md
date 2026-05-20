# Metadata Rules

## 1. Purpose

`metadata.toml` is the required machine-readable metadata file. It describes how to parse, validate, restore, and interpret `dataset.lance/`.

It is not merely a dataset introduction. Human-readable summaries belong in `description.toml`.

## 2. Required top-level sections

`metadata.toml` MUST contain:

```toml
spec_version = "0.1.0"

[dataset]
[storage]
[schema]
[[schema.columns]]
[data]
[quality_control]
```

It MUST also contain one modality-specific section according to `dataset.modality`:

```toml
[eeg]   # required when modality = "eeg"
[mri]   # required when modality = "mri"
[fmri]  # required when modality = "fmri"
```

If the dataset is supervised, it MUST contain:

```toml
[label]
```

## 3. `[dataset]` required fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `dataset_id` | string | Yes | Unique dataset ID |
| `name` | string | Yes | Dataset name |
| `modality` | string | Yes | `eeg`, `mri`, `fmri`, or declared modality |
| `version` | string | Yes | Semantic dataset version |
| `created_at` | datetime/string | Yes | Creation time |
| `updated_at` | datetime/string | Yes | Last update time |
| `sample_unit` | string | Yes | `window`, `epoch`, `image`, `volume`, `run`, etc. |
| `n_samples` | int | Yes | Number of samples |

Recommended fields:

```toml
description = "..."
task_type = "classification"
n_subjects = 100
```

## 4. `[storage]` required fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `table_name` | string | Yes | Default: `dataset.lance` |
| `lance_path` | string | Yes | Relative or absolute Lance path |
| `path_type` | string | Yes | `relative` or `absolute` |

Recommended fields:

```toml
root_path = "."
backend = "local"  # local, s3, tos, oss, cos, etc.
uri = ""
```

## 5. `[schema]` rules

`[schema]` MUST declare:

```toml
primary_key = "sample_id"
data_column = "data"
```

If supervised:

```toml
label_column = "label"
```

Each column in `dataset.lance/` MUST be described by `[[schema.columns]]`.

Each `[[schema.columns]]` entry MUST contain:

| Field | Type | Required | Description |
|---|---|---:|---|
| `name` | string | Yes | Column name |
| `type` | string | Yes | Lance/Arrow logical type |
| `required` | bool | Yes | Required by the spec or dataset |
| `nullable` | bool | Yes | Whether null values are allowed |
| `description` | string | Yes | Human-readable field meaning |

Recommended optional fields:

```toml
allowed_values = ["train", "val", "test"]
unit = "uV"
shape = ["n_channels", "n_times"]
```

## 6. Required base columns

`metadata.toml` MUST include schema definitions for:

```text
sample_id
subject_id
modality
data
shape
original_shape
valid_length
qc_pass
```

It SHOULD include definitions for:

```text
session_id
run_id
label
label_name
split
```

## 7. `[data]` required fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `array_column` | string | Yes | Data column name |
| `dtype` | string | Yes | Array data type |
| `axis_order` | list<string> | Yes | Axis order after restoration |
| `padding` | bool | Yes | Whether padding is used |
| `padding_value` | number | Yes | Padding fill value |
| `shape_column` | string | Yes | Padded shape column |
| `original_shape_column` | string | Yes | Original shape column |
| `valid_length_column` | string | Yes | Valid flattened length column |

## 8. `[label]` rules

If supervised, `[label]` MUST declare:

```toml
label_column = "label"
task_type = "classification"  # classification, regression, multilabel
```

For classification, it MUST also declare:

```toml
num_classes = 5
```

and each class:

```toml
[[label.classes]]
id = 0
name = "class_name"
description = "..."
```

For regression, it SHOULD declare:

```toml
target_name = "age"
target_unit = "year"
```

## 9. `[quality_control]` required fields

```toml
[quality_control]
qc_column = "qc_pass"
```

Recommended:

```toml
qc_pass_values = [true, false]
qc_method = "..."
```

## 10. EEG required metadata

When `dataset.modality = "eeg"`, `[eeg]` MUST contain:

| Field | Type | Required | Description |
|---|---|---:|---|
| `n_channels` | int | Yes | Number of channels |
| `sampling_rate` | float | Yes | Sampling rate |
| `sampling_rate_unit` | string | Yes | Usually `Hz` |
| `unit` | string | Yes | Signal unit, e.g. `uV` |
| `reference` | string | Yes | Reference setting |
| `montage` | string | Yes | Electrode montage |
| `channel_axis` | int | Yes | Channel axis after restoration |
| `time_axis` | int | Yes | Time axis after restoration |

It MUST include `[[eeg.channels]]` entries for every channel.

Each channel MUST contain:

| Field | Type | Required | Description |
|---|---|---:|---|
| `index` | int | Yes | Channel order in data |
| `name` | string | Yes | Channel name |
| `type` | string | Yes | `eeg`, `eog`, `emg`, `ecg`, `stim`, `misc`, `ref` |
| `unit` | string | Yes | Channel unit |

Recommended channel fields:

```toml
status = "good"  # good, bad, unknown
x = 0.0
y = 0.0
z = 0.0
```

## 11. MRI required metadata

When `dataset.modality = "mri"`, `[mri]` MUST contain:

| Field | Type | Required | Description |
|---|---|---:|---|
| `image_type` | string | Yes | `T1w`, `T2w`, `FLAIR`, `DWI`, etc. |
| `field_strength` | float | Yes | Magnetic field strength |
| `field_strength_unit` | string | Yes | Usually `T` |
| `manufacturer` | string | Yes | Scanner manufacturer or `unknown` |
| `scanner_model` | string | Yes | Scanner model or `unknown` |
| `voxel_size` | list<float> | Yes | Voxel size |
| `voxel_size_unit` | string | Yes | Usually `mm` |
| `orientation` | string | Yes | `RAS`, `LAS`, etc. |
| `space` | string | Yes | `native`, `MNI152`, etc. |
| `spatial_axes` | list<string> | Yes | Usually `["x", "y", "z"]` |
| `affine_column` | string | Yes | Lance column containing flattened affine, or empty string |

Recommended fields:

```toml
sequence_name = "MPRAGE"
repetition_time = 2300.0
echo_time = 2.98
inversion_time = 900.0
flip_angle = 9.0
slice_thickness = 1.0
spacing_between_slices = 1.0
phase_encoding_direction = "unknown"
skull_stripped = false
bias_corrected = false
normalized = false
```

## 12. fMRI required metadata

When `dataset.modality = "fmri"`, `[fmri]` MUST contain:

| Field | Type | Required | Description |
|---|---|---:|---|
| `bold_type` | string | Yes | Usually `BOLD` |
| `tr` | float | Yes | Repetition time |
| `tr_unit` | string | Yes | Usually `s` |
| `space` | string | Yes | `native`, `MNI152`, etc. |
| `voxel_size` | list<float> | Yes | Voxel size |
| `voxel_size_unit` | string | Yes | Usually `mm` |
| `spatial_axes` | list<string> | Yes | Spatial axes |
| `time_axis` | int | Yes | Time axis index |
| `slice_timing_corrected` | bool | Yes | Whether slice timing correction was applied |
| `motion_corrected` | bool | Yes | Whether motion correction was applied |
| `normalized` | bool | Yes | Whether normalized to a template space |

