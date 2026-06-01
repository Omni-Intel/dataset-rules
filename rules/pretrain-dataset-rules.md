# Pretraining Dataset Rules

## 1. Purpose

This document defines the extension rules for pretraining datasets.

Pretraining datasets MUST reuse the core dataset contract:

```text
dataset.lance/
metadata.toml
versions.toml
description.toml
```

The extension exists because pretraining datasets commonly differ from downstream
datasets in four ways:

- they combine multiple source datasets;
- they are usually self-supervised and may not have task labels;
- they may contain multiple physical Lance tables for efficient batching;
- they may include row-aligned feature views derived from the signal table.

These differences MUST be declared in `metadata.toml`. `description.toml` MUST
NOT be required for parsing, validation, restoration, sampling, or training.

## 2. Relationship to Downstream Rules

Pretraining datasets MUST remain compatible with the base Lance and metadata
rules unless this document explicitly defines an extension.

Required base columns remain required for the logical signal table. For
partitioned pretraining datasets, every released signal table MUST contain
these physical columns:

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

For single-table pretraining datasets, `sample_id` SHOULD remain a globally
unique stable string, matching the downstream convention.

For partitioned pretraining datasets, `sample_id` MAY be table-local when:

```toml
[pretrain]
sample_id_scope = "table_local"
sample_key_columns = ["table_id", "sample_id"]
```

In `table_local` mode, the globally unique sample key is the compound key
`(table_id, sample_id)`. `table_id` MUST match the
`[[pretrain.tables]].table_id` of the physical signal table, and `sample_id`
MUST be unique within that table. A dataset MAY materialize `sample_uid` as a
globally unique string derived from `table_id` and `sample_id`, for example
`"tuh_train:000000001"`, but validators MUST treat `(table_id, sample_id)` as
the authoritative key when `sample_id_scope = "table_local"`.

If a source table contains an original local row identifier, it SHOULD be stored
as `source_sample_id` or `local_sample_id`, not used as the pretraining sample
key unless it also satisfies the declared sample key contract.

Pretraining datasets SHOULD set:

```toml
[dataset]
task_type = "pretraining"
```

Supervised `[label]` metadata is OPTIONAL for pretraining datasets. If labels are
included for auxiliary or evaluation use, they MUST be declared as optional and
MUST NOT be required to read the pretraining samples.

## 3. Layout

### 3.1 Preferred single-table layout

When practical, a pretraining dataset SHOULD expose one canonical signal table:

```text
<dataset_id>/
  dataset.lance/
  metadata.toml
  versions.toml
  description.toml
  derivatives/
    features/
      <feature_view_id>.lance/
    qc/
```

`dataset.lance/` remains the canonical signal table. Derived feature tables live
under `derivatives/` and MUST be declared in `metadata.toml`.

### 3.2 Partitioned layout

Pretraining datasets MAY use multiple physical Lance tables when a single table
is impractical for scale, channel profile separation, source separation, or
batching efficiency.

Partitioned layouts MUST declare every physical table in `[[pretrain.tables]]`.
The declared tables form one logical dataset. Validators and readers MUST treat
the union of declared signal tables as the dataset surface.

Partitioned layouts MUST set:

```toml
[storage]
table_name = "__partitioned__"
lance_path = "__partitioned__"
```

The sentinel value `__partitioned__` means that no single physical
`dataset.lance/` is the canonical signal table. Readers and validators MUST
resolve the logical signal table from all `[[pretrain.tables]]` entries with
`role = "signal"`.

Example:

```text
<dataset_id>/
  metadata.toml
  versions.toml
  tables/
    signal/
      tuh_train.lance/
      tuh_val.lance/
      openneuro_slot064_train.lance/
    features/
      npd_v1/
        tuh_train.lance/
        openneuro_slot064_train.lance/
```

Any generated registry or index file is an implementation cache. It MUST NOT be
required to parse, validate, restore, sample, or train from the dataset unless
it is explicitly declared as a `[[pretrain.tables]]` entry with `role = "index"`.

## 4. Required Pretraining Metadata

When `dataset.task_type = "pretraining"`, `metadata.toml` MUST include:

```toml
[pretrain]
```

Required `[pretrain]` fields:

| Field | Type | Required | Description |
|---|---|---:|---|
| `pretrain_type` | string | Yes | `self_supervised`, `supervised`, `multi_objective`, or `contrastive` |
| `sample_id_scope` | string | Yes | `global` or `table_local` |
| `sample_key_columns` | array[string] | Conditional | Required when `sample_id_scope = "table_local"`; SHOULD be `["table_id", "sample_id"]` |
| `source_dataset_column` | string | Yes | Column containing the source dataset ID |
| `recording_id_column` | string | Yes | Column identifying the source recording |
| `preprocess_version` | string | Yes | Default or primary preprocessing pipeline version. Row-level `preprocess_version` is the ground truth. |
| `split_policy` | string | Yes | Split isolation policy |

Recommended `[pretrain]` fields:

```toml
objective_family = "masked_prediction"
feature_view_ids = ["npd_v1"]
sampling_policy = "balanced_by_source"
sample_uid_column = "sample_uid"
```

When `sample_id_scope = "table_local"`, `[schema].primary_key` MUST be:

```toml
[schema]
primary_key = ["table_id", "sample_id"]
```

This is a pretraining extension of the base downstream convention
`primary_key = "sample_id"`. Single-table pretraining datasets SHOULD keep the
base convention unless they intentionally use table-local keys.

## 5. Required Pretraining Columns

In addition to the base columns, every released signal table MUST contain the
following physical columns:

| Column | Type | Required | Description |
|---|---|---:|---|
| `source_dataset_id` | string | Yes | Source dataset identifier, e.g. `tuh`, `ds004395` |
| `recording_id` | string | Yes | Stable recording identifier within the source dataset |
| `source_path` | string | Yes | Relative source file path or source URI |
| `start_time` | float64 | Yes | Window start time in seconds |
| `duration` | float64 | Yes | Window duration in seconds |
| `split` | string | Yes | Pretraining split |
| `preprocess_version` | string | Yes | Row preprocessing version. This value is authoritative when it differs from `[pretrain].preprocess_version`. |
| `channel_profile` | string | Yes for EEG | Channel profile identifier |

Recommended columns:

| Column | Type | Description |
|---|---|---|
| `table_id` | string | Physical table identifier; recommended for partitioned datasets and required as an available reader field when `sample_id_scope = "table_local"` |
| `sample_uid` | string | Optional materialized globally unique sample key derived from `table_id` and `sample_id` |
| `subject_uid` | string | Stable subject identifier scoped across sources |
| `recording_uid` | string | Stable recording identifier scoped across sources |
| `source_sample_id` | string/int64 | Source-local sample ID |
| `sample_weight` | float32 | Optional sampling weight |
| `license` | string | Source license or access term |
| `access_level` | string | `open`, `registered`, `restricted`, `internal`, or `unknown` |

If a dataset uses `split_policy = "subject_disjoint"`, split isolation MUST be
verified using `subject_uid` when present, otherwise using the compound key
`(source_dataset_id, subject_id)`.

If a dataset uses `split_policy = "recording_disjoint"`, split isolation MUST be
verified using `recording_uid` when present, otherwise using the compound key
`(source_dataset_id, recording_id)`.

## 6. Source Dataset Metadata

Each source dataset MUST be declared:

```toml
[[pretrain.sources]]
source_dataset_id = "ds004395"
source_family = "openneuro"
version = "snapshot-2026.05.29"
license = "unknown"
access_level = "open"
n_samples = 185315
n_subjects = 30
```

Required fields:

| Field | Type | Required | Description |
|---|---|---:|---|
| `source_dataset_id` | string | Yes | Source ID used in Lance rows |
| `source_family` | string | Yes | Source family, e.g. `tuh`, `openneuro` |
| `version` | string | Yes | Source snapshot or release version |
| `license` | string | Yes | License or access term |
| `access_level` | string | Yes | Access category |
| `n_samples` | int | Yes | Number of released samples from this source |

`source_dataset_id` values MUST be unique inside one pretraining dataset.

`access_level` MUST be one of:

```text
open
registered
restricted
internal
unknown
```

`source_family` SHOULD use a stable lowercase identifier such as `tuh`,
`openneuro`, `sleep-edf`, or `internal`. New source families MAY be introduced
without changing the specification.

## 7. Physical Table Metadata

Partitioned datasets MUST declare every table:

```toml
[[pretrain.tables]]
table_id = "openneuro_slot136_train"
role = "signal"
lance_path = "tables/signal/openneuro_slot136_train.lance"
split = "train"
source_family = "openneuro"
channel_profile = "openneuro_slot136"
n_samples = 185315
```

Required fields:

| Field | Type | Required | Description |
|---|---|---:|---|
| `table_id` | string | Yes | Stable table identifier. Values MUST be unique inside one pretraining dataset. |
| `role` | string | Yes | `signal`, `feature`, `index`, or `manifest` |
| `lance_path` | string | Yes | Relative or absolute Lance path |
| `n_samples` | int | Yes | Row count |

Recommended fields:

```toml
split = "train"
source_family = "openneuro"
source_dataset_id = "ds004395"
channel_profile = "openneuro_slot136"
view_id = "npd_v1"
aligned_to = "openneuro_slot136_train"
alignment = "sample_id"
```

`alignment` MUST be one of `sample_id`, `sample_key`, `sample_uid`, or
`row_index`.

If `alignment = "sample_id"`, the feature table MUST contain the same
`sample_id` values as its aligned signal table. In `table_local` mode this is
unambiguous only when `aligned_to` names exactly one signal table.

If `alignment = "sample_key"`, the feature table MUST contain `signal_table_id`
and `sample_id`; together they reference `(table_id, sample_id)` in the aligned
signal tables. `sample_key` SHOULD be used when one feature table mixes samples
from multiple signal tables under `sample_id_scope = "table_local"`.

If `alignment = "sample_uid"`, the feature table MUST contain the materialized
`sample_uid` column declared by `[pretrain].sample_uid_column`.

If `alignment = "row_index"`, the feature table MUST contain an explicit
`row_index` column whose values reference row indexes in the declared
`aligned_to` signal table. Pure implicit row-order alignment MUST NOT be used
for released datasets.

## 8. Feature Views

Derived features SHOULD be stored as feature views rather than replacing signal
data in the canonical signal table.

Each feature view MUST be declared:

```toml
[[pretrain.feature_views]]
view_id = "npd_v1"
feature_type = "handcrafted_descriptor"
derived_from = "signal"
feature_version = "2026.05.29"
alignment = "sample_id"
required_for_training = false
```

Feature tables MUST contain:

```text
sample_id
```

when `alignment = "sample_id"`. Feature tables MUST contain `signal_table_id`
and `sample_id` when `alignment = "sample_key"`. Feature tables MUST contain
`sample_uid` when `alignment = "sample_uid"`. Feature tables MUST contain
`row_index` when `alignment = "row_index"`.

Feature tables MUST also contain `feature_version`, or declare a single feature
version in `[[pretrain.feature_views]]`. A row-level `feature_version` is
authoritative when it differs from the feature-view-level default.

## 9. EEG Channel Profiles

EEG pretraining datasets SHOULD declare channel profiles when multiple channel
universes or slot layouts are used:

```toml
[[pretrain.channel_profiles]]
profile_id = "tuh_named26"
n_channels = 26
layout = "fixed_slot"
channel_mask_column = "channel_mask"
electrode_id_column = "electrode_ids"
```

`profile_id` values MUST match the `channel_profile` column values. Validators
MUST NOT infer channel count from physical table names.

`layout` MUST be one of:

```text
fixed_slot
variable
named_only
```

`[eeg].channel_layout` describes how channel names are stored in Lance rows.
`[[pretrain.channel_profiles]].layout` describes the slot layout inside one
pretraining channel profile.

For non-EEG rows in multimodal pretraining datasets, `channel_profile` MUST be
null or an empty string. Validators MUST only require a non-empty declared
`channel_profile` for rows where `modality = "eeg"`.

## 10. Split Rules

Pretraining splits MUST be stable and reproducible.

`[pretrain]` MUST declare `split_policy`. It MUST be one of:

```text
subject_disjoint
recording_disjoint
source_dataset_disjoint
custom_declared
```

If `split_policy = "custom_declared"`, `[pretrain]` MUST also declare
`split_declaration_path`, and the referenced file MUST describe the split
isolation rule.

For EEG, windows from the same recording MUST NOT appear in multiple splits
unless the dataset explicitly declares a non-leakage exception and its intended
use.

## 11. Sampling Rules

Pretraining metadata SHOULD declare the default sampling policy:

```toml
[pretrain.sampling]
policy = "balanced_by_source"
source_weighting = "temperature"
temperature = 0.5
weight_column = "sample_weight"
```

Sampling policy metadata is required when the row distribution is not intended
to define training probability.

If `weight_column` is declared, the named column MUST exist in every signal
table used by the sampler.

## 12. Validation

Pretraining validation MUST include all base dataset validation checks plus:

- If `sample_id_scope = "global"`, `sample_id` is globally unique across all
  declared signal tables.
- If `sample_id_scope = "table_local"`, each declared signal table has a unique
  `table_id`, `sample_id` is unique within each signal table, and the compound
  key `(table_id, sample_id)` is the authoritative global sample key.
- All declared `[[pretrain.sources]]` IDs appear in rows or are explicitly marked
  unused.
- All declared `[[pretrain.tables]]` paths exist and row counts match.
- Signal table rows contain required pretraining columns.
- Feature views are aligned to signal tables by `sample_id`, `sample_key`,
  `sample_uid`, or explicit `row_index`.
- Split policy is declared and verified at the subject or recording level when
  the required columns are present.
- EEG `channel_profile` values are declared and their channel counts match the
  restored shape.
- Sampling policy is declared when source distributions are intentionally
  reweighted.
