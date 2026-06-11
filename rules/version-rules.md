# Version Rules

## 1. Purpose

`versions.toml` maps Lance internal table versions to human-readable semantic changes.

Lance versions describe table states. `versions.toml` explains why a table state exists and what changed.

## 2. Required top-level fields

```toml
spec_version = "0.1.0"
dataset_id = "ds-example"
table = "dataset.lance"
```

## 3. Current version section

`versions.toml` SHOULD contain:

```toml
[current]
lance_version = 1
dataset_version = "2026.05.20"
description = "Current validated release."
```

## 4. Version entries

Each meaningful Lance version SHOULD be described with `[[versions]]`.

Required fields:

| Field | Type | Required | Description |
|---|---|---:|---|
| `lance_version` | int | Yes | Lance internal table version |
| `dataset_version` | string | Yes | Semantic dataset version |
| `created_at` | datetime/string | Yes | Change time |
| `author` | string | Yes | Author or system |
| `change_type` | string | Yes | Controlled change type |
| `compatible_with_previous` | bool | Yes | Whether old readers can still use this version |
| `description` | string | Yes | Human-readable change description |
| `affected_rows` | int | Yes | Number of affected rows |
| `affected_columns` | list<string> | Yes | Affected columns |

## 5. Recommended `change_type` values

```text
initial_import
append_samples
delete_samples
update_data
label_fix
qc_update
split_update
schema_update
metadata_update
preprocessing_update
feature_update
migration
release
rollback
```

## 6. Example

```toml
spec_version = "0.1.0"
dataset_id = "ds-hmc-eeg"
table = "dataset.lance"

[current]
lance_version = 3
dataset_version = "2026.05.20"
description = "Current validated release."

[[versions]]
lance_version = 1
dataset_version = "2026.05.18"
created_at = "2026-05-18T12:00:00+08:00"
author = "system"
change_type = "initial_import"
compatible_with_previous = false
description = "Initial conversion from raw files to Lance format."
affected_rows = 111890
affected_columns = ["sample_id", "subject_id", "data", "label"]

[[versions]]
lance_version = 2
dataset_version = "2026.05.19"
created_at = "2026-05-19T09:30:00+08:00"
author = "system"
change_type = "schema_update"
compatible_with_previous = true
description = "Added split column."
affected_rows = 111890
affected_columns = ["split"]

[[versions]]
lance_version = 3
dataset_version = "2026.05.20"
created_at = "2026-05-20T11:00:00+08:00"
author = "system"
change_type = "schema_update"
compatible_with_previous = true
description = "Added original_shape and valid_length for reversible padding."
affected_rows = 111890
affected_columns = ["original_shape", "valid_length"]
```
