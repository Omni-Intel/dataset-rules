# Pretrain Profile Rules

This document defines the **pretrain profile**: a sibling profile of the base specification for
**self-supervised EEG pretraining** corpora (for example OpenNeuro datasets and the TUH foundation corpus).

The pretrain profile reuses the base layout, Lance, metadata, version, and description rules. This
document only states where the pretrain profile **extends or overrides** the base rules. Where this
document is silent, the base rules apply.

A dataset that follows this profile MUST set, at the top of `metadata.toml`:

```toml
spec_version = "0.1.0"
profile = "pretrain"
```

A dataset without `profile`, or with `profile = "base"`, follows the base (supervised) rules unchanged.

## 1. Purpose and scope

- The pretrain profile applies to EEG datasets used for self-supervised pretraining. Data is **unlabeled**.
- One dataset MUST be stored as one directory with one main `dataset.lance/` table.
- Train and validation splits MUST be encoded by a `split` column, not by separate tables.

## 2. Layout extensions

Base: `dataset-layout-rules.md`.

```text
<dataset_id>/
  dataset.lance/                    # required, main signal table (one row = one window/segment)
  metadata.toml                     # required
  versions.toml                     # recommended
  description.toml                  # optional
  derivatives/
    electrode_vocab.json            # recommended, shared electrode vocab (id <-> name)
    montage_vocab.json              # optional, montage vocab (id <-> name)
    features/
      npd.lance/                    # recommended, row-aligned NPD feature table
    qc/                             # optional
    reports/                        # optional
```

### MUST / SHOULD / MAY

- A pretrain dataset MUST contain `dataset.lance/` and `metadata.toml`.
- If NPD features are provided, `derivatives/features/npd.lance/` MUST exist and MUST be declared in `[features.npd]`.
- The referenced electrode vocab SHOULD be stored at `derivatives/electrode_vocab.json` and declared via `[eeg].electrode_vocab_path`.

### `dataset_id` conventions

- OpenNeuro: MUST use the accession, e.g. `ds003690`, `ds005089`. Each OpenNeuro accession is its own dataset and its own table.
- TUH foundation: use `tuh-eeg-foundation` (single dataset, single table).

### One table per dataset (override)

The base spec permits merging strictly aligned data. The pretrain profile **requires one table per dataset**.
Merging multiple datasets into channel-count buckets (for example `eeg_openneuro_64ch`) MUST NOT be used as
the pretrain storage form.

## 3. Main table extensions

Base: `lance-rules.md`. Required base columns are unchanged:
`sample_id, subject_id, modality, data, shape, original_shape, valid_length, qc_pass`.

The pretrain profile **promotes the following EEG columns from optional to required**:

| Column | Type | Required | Description |
|---|---|---:|---|
| `electrode_ids` | fixed_size_list\<uint16>[C] | Yes | Shared-vocab electrode id per slot; `0`=pad, `1`=UNK_EEG |
| `channel_names` | list\<string> (length C) | Yes | Canonical channel name per slot, aligned with `electrode_ids` |
| `channel_mask` | fixed_size_list\<bool>[C] | Yes | `true` = valid (non-padding) slot |
| `bad_channel_mask` | fixed_size_list\<bool>[C] | Yes | `true` = bad channel, meaningful only within valid slots |

- `electrode_ids` and `channel_names` describe the same fact (which electrode occupies slot `i`) and MUST
  have equal length C and identical order.
- `label` and `label_name` columns MUST NOT be present.
- The signal column MUST be named `data`. A legacy `signals` column MUST be renamed to `data` before release.

Recommended columns (SHOULD when applicable):
`split, session_id, run_id, source_path, start_time, channel_counts, montage_id, channel_status,
shard_name, global_idx, dataset_key`.

## 4. Channel and electrode model

### Shared electrode vocab

- All pretrain datasets MUST reference one shared electrode vocab: the same physical electrode maps to the
  same integer id in every dataset (for example `F7=17` and `F3=19` in both TUH and OpenNeuro).
- Reserved ids: `0 = PAD` (padding slot), `1 = UNK_EEG` (EEG-like channel not in the vocab).
- Relation: `electrode_ids[i] = electrode_vocab[ canonicalize(channel_names[i]) ]`.
- The canonicalization policy SHOULD be recorded in `description.toml` or `[preprocessing]`.

### `channel_layout`

The base values `dataset_level`, `per_subject`, and `per_sample` are reused. Regardless of layout, the
per-row `electrode_ids` and `channel_names` are always present and authoritative under this profile.

| Value | Meaning |
|---|---|
| `dataset_level` | Every row uses the slot order declared in `[[eeg.channels]]`; `n_channels` equals the entry count. |
| `per_subject` | Slot mapping may differ by subject; rows of the same subject SHOULD agree. |
| `per_sample` | Slot mapping may differ by sample; `electrode_ids`/`channel_names` are authoritative. |

### Masks

- `channel_mask` (valid vs padding): padding slots SHOULD be appended at the end (slots
  `n_valid_channels … C-1`) with `electrode_id = 0`, `channel_names = ""`, `channel_mask = false`.
- `bad_channel_mask` (good vs bad): flags artifact/interpolated channels inside valid slots; it does NOT change shape.
- Both masks MUST align with the channel axis (length C).

## 5. Padding and restoration

Base: `lance-rules.md` §4–§5.

- `data` MUST be the (C, T) array flattened row-major into `fixed_size_list<float32>` of length `C·T`.
- `shape = [C, T]`; `original_shape = [n_valid_channels, T]`. Time length T is fixed within a dataset, so
  `[data].time_padding = false`.
- `valid_length = n_valid_channels · T`, explicitly declared (it is NOT `C·T`).
- Restoration uses `channel_mask` to select valid channels:

```python
padded = data.reshape(shape)                                   # (C, T)
original = padded[[i for i, m in enumerate(channel_mask) if m], :]   # (n_valid_channels, T)
```

`[data]` adds these required fields under this profile:

```toml
channel_padding = "slot_mask"
channel_mask_column = "channel_mask"
time_padding = false
```

## 6. metadata.toml extensions

Base: `metadata-rules.md`.

- `[label]` MUST NOT appear. `dataset.task_type` MUST be `"pretrain"`.
- `[pretrain]` MUST appear (it replaces `[label]`).
- `[features.npd]` MUST appear with `present = true` if NPD features are provided. It MAY appear with `present = false` when NPD features are not provided.
- `[preprocessing]` SHOULD appear.

### `[pretrain]`

| Field | Type | Required | Description |
|---|---|---:|---|
| `is_labeled` | bool | Yes | MUST be `false` |
| `recommended_pretext` | list\<string> | MAY | e.g. `["masked_reconstruction", "jepa", "cwt_time_freq"]` |

### `[preprocessing]` (SHOULD)

| Field | Type | Description |
|---|---|---|
| `target_sfreq` | float | Resampled rate, e.g. `200.0` |
| `window_seconds` | float | Window length in seconds |
| `segment_samples` | int | Samples per window (`= target_sfreq * window_seconds`) |
| `bandpass_low` / `bandpass_high` | float | Bandpass edges |
| `notch_hz` | float | Notch frequency |
| `unit_before_normalization` | string | e.g. `"uV"` |
| `normalization` | string | e.g. `"clip_divide_200"` |
| `normalization_detail` | string | Human-readable normalization formula |
| `channel_canonicalization` | string | Canonicalization policy summary |

### `[eeg]` additions

In addition to the base required EEG fields, the pretrain profile requires:

| Field | Type | Required | Description |
|---|---|---:|---|
| `electrode_ids_column` | string | Yes | `"electrode_ids"` |
| `channel_names_column` | string | Yes | `"channel_names"` |
| `channel_mask_column` | string | Yes | `"channel_mask"` |
| `bad_channel_mask_column` | string | Yes | `"bad_channel_mask"` |
| `electrode_vocab_name` | string | SHOULD | Shared vocab name |
| `electrode_vocab_version` | string | SHOULD | Vocab version |
| `electrode_vocab_path` | string | SHOULD | `"derivatives/electrode_vocab.json"` |
| `pad_electrode_id` | int | SHOULD | `0` |
| `unk_electrode_id` | int | SHOULD | `1` |

`[[eeg.channels]]` entries (required for `dataset_level`) carry an additional `electrode_id` field:

| Field | Type | Required | Description |
|---|---|---:|---|
| `index` | int | Yes | Slot index, contiguous from 0 |
| `electrode_id` | int | Yes | Shared-vocab id |
| `name` | string | Yes | Channel name |
| `type` | string | Yes | `eeg`/`eog`/`emg`/`ecg`/`stim`/`misc`/`ref` |
| `unit` | string | Yes | Channel unit |

### `[features.npd]`

This section MAY appear with `present = false` to explicitly declare that NPD features are absent. If
`present = true`, or if NPD features exist, all required fields below MUST be present and
`derivatives/features/npd.lance/` MUST satisfy the NPD derivative table rules.

| Field | Type | Required | Description |
|---|---|---:|---|
| `present` | bool | Yes | Whether NPD features exist |
| `table_name` / `lance_path` | string | Yes | `"derivatives/features/npd.lance"` |
| `row_alignment` | string | Yes | `"sample_id"` |
| `extractor` | string | Yes | Feature extractor name |
| `extractor_version` | string | SHOULD | Extractor version |
| `channel_feature_dim` | int | Yes | Per-channel feature dimension (e.g. 41) |
| `segment_feature_dim` | int | Yes | Segment-level feature dimension (e.g. 76) |
| `manual_feature_dim` | int | Yes | Manual feature dimension (e.g. 158) |
| `channel_feature_axis_order` | list\<string> | Yes | e.g. `["channel", "feature"]` |
| `signal_already_normalized` | bool | SHOULD | Whether features were computed on normalized signal |

## 7. NPD derivative table

`derivatives/features/npd.lance/` MUST be row-aligned to the main table by `sample_id`.

- Row count MUST equal the main table; the `sample_id` set MUST be equal; rows SHOULD keep the same order.
- Readers MUST join by `sample_id`, not by implicit row position.

| Column | Type | Required | Description |
|---|---|---:|---|
| `sample_id` | string | Yes | Join key to the main table |
| `channel_features` | fixed_size_list\<float32>[C·D_ch] | Yes | Flattened (C, D_ch) per-channel features |
| `segment_features` | fixed_size_list\<float32>[D_seg] | Yes | Segment-level features |
| `manual_features` | fixed_size_list\<float32>[D_man] | Yes | Manual features |
| `channel_mask` | fixed_size_list\<bool>[C] | Yes | Aligned with the main table |
| `electrode_ids` | fixed_size_list\<uint16>[C] | Yes | Aligned with the main table |
| `bad_channel_mask` | fixed_size_list\<bool>[C] | Yes | Aligned with the main table |
| `subject_id` / `split` | string | SHOULD | Provenance redundancy for standalone loading |

NPD table changes SHOULD be tracked in `versions.toml` with `change_type = "feature_update"`.

## 8. Versioning and description

- `versions.toml`: base rules apply; the controlled `change_type` set adds `feature_update` (for NPD).
- `description.toml`: for OpenNeuro, the `[collection]`/`[citation]` sections SHOULD carry the BIDS
  `Name`, `DatasetDOI`, `Authors`, and `License`; `intended_use.tasks` SHOULD include
  `["representation_learning", "pretraining"]`.

## 9. Validation

See the **Pretrain profile checks** section in `validation/validation-checklist.md`.

## 10. Dataset conventions (OpenNeuro / TUH)

| | OpenNeuro (one table per accession) | TUH foundation (single table) |
|---|---|---|
| `dataset_id` | `ds00XXXX` | `tuh-eeg-foundation` |
| `dataset.source` | `"openneuro"` | `"tuh"` |
| C (padded slots) | per-dataset `c_max` (e.g. ds003690 = 65) | 26 (named26 fixed slots) |
| `channel_layout` | usually `per_sample` | `dataset_level` |
| T | 2000 (200 Hz × 10 s) | 2000 |
| split | `split` column (`train`/`val`) | `split` column (`train`/`val`) |

A complete example dataset is provided under `examples/eeg-pretrain/`.
