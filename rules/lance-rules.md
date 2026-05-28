# Lance Table Rules

## 1. Purpose

`dataset.lance/` is the main sample-level data table. Each row represents one sample.

A sample may be:

- an EEG window or epoch;
- an MRI image;
- an fMRI volume, window, or run;
- a feature vector or derived sample, if declared in `metadata.toml`.

## 2. Required columns

`dataset.lance/` MUST contain:

| Column | Type | Required | Description |
|---|---|---:|---|
| `sample_id` | string | Yes | Unique sample identifier |
| `subject_id` | string | Yes | Subject identifier |
| `modality` | string | Yes | `eeg`, `mri`, `fmri`, or another declared modality |
| `data` | fixed_size_list<float32> or list<float32> | Yes | Flattened padded data |
| `shape` | list<int32> | Yes | Padded shape |
| `original_shape` | list<int32> | Yes | Original shape before padding |
| `valid_length` | int64 | Yes | Number of valid flattened values before padding |
| `qc_pass` | bool | Yes | Whether the sample passed QC |

## 3. Recommended columns

`dataset.lance/` SHOULD contain when applicable:

| Column | Type | Description |
|---|---|---|
| `session_id` | string | Session identifier |
| `run_id` | string | Run identifier |
| `label` | int32 / float32 / string | Primary label |
| `label_name` | string | Human-readable label |
| `split` | string | `train`, `val`, `test`, or another declared split |
| `source_path` | string | Relative source path or URI |
| `created_at` | timestamp/string | Row creation time |
| `updated_at` | timestamp/string | Row update time |

## 4. Data storage rule

All multidimensional data MUST be stored as a flattened array in the `data` column.

The table MUST also store:

```text
shape           # padded shape
original_shape  # shape before padding
valid_length    # flattened valid length before padding
```

Restore procedure:

```python
padded = data.reshape(shape)
original = crop_or_slice(padded, original_shape)
```

## 5. Padding rule

- Data SHOULD be padded to a fixed shape within a dataset.
- Padding value MUST be declared in `metadata.toml`.
- Axis order MUST be declared in `metadata.toml`.
- `shape` MUST describe the padded array shape.
- `original_shape` MUST describe the unpadded array shape.
- `valid_length` MUST equal the product of `original_shape`, unless explicitly declared otherwise.

## 6. Column naming rule

Default names:

```text
data
shape
original_shape
valid_length
label
label_name
qc_pass
split
```

If non-default names are used, they MUST be declared in `metadata.toml`.

## 7. Modality-specific columns

### EEG optional columns

```text
start_time: float64
duration: float64
start_sample: int64
channel_names: list<string>
channel_status: list<string>
channel_mask: list<bool>
```

For variable EEG channel layouts, `metadata.toml` MUST declare which channel columns to read. When present:

- `channel_names` MUST contain channel names in the order used by the restored data channel axis.
- `channel_status` MUST align with `channel_names`.
- `channel_mask` MUST align with the declared dataset-level channel universe when one exists; otherwise it MUST align with `channel_names`.

For `channel_layout = "per_subject"`, subject-level channel metadata SHOULD be repeated on every sample row for that subject.

### MRI optional columns

```text
affine: fixed_size_list<float32>[16]
voxel_size: list<float32>
space: string
image_type: string
```

### fMRI optional columns

```text
affine: fixed_size_list<float32>[16]
tr: float32
start_volume: int32
n_volumes: int32
space: string
```

## 8. Indexing recommendation

Indexes SHOULD be considered for:

```text
sample_id
subject_id
session_id
run_id
modality
label
qc_pass
split
```
