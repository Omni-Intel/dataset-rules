# Dataset Rules

`dataset-rules` defines a lightweight neuroimaging dataset specification for storing sample-level data in Lance and machine-readable metadata in TOML.

The specification is designed for EEG, MRI, fMRI, and future multimodal datasets. The current default is a single-main-table layout:

```text
<dataset_id>/
  dataset.lance/          # required, main Lance table
  metadata.toml           # required, machine-readable metadata and schema
  versions.toml           # recommended, Lance version changelog
  description.toml        # optional, human-readable overview and processing notes
```

## Core principles

1. `dataset.lance` stores sample-level data, labels, IDs, shapes, QC fields, and splits.
2. `metadata.toml` stores structured details required to parse, restore, validate, and train on the Lance table.
3. `versions.toml` maps Lance internal versions to human-readable semantic changes.
4. `description.toml` is optional and only describes dataset overview, collection background, processing summary, intended use, limitations, and citations.
5. All array-like neuroimaging data should be padded to a fixed shape before being stored in Lance. The original shape must be preserved for restoration.

## Required files

| File | Required | Purpose |
|---|---:|---|
| `dataset.lance/` | Yes | Main sample-level Lance table |
| `metadata.toml` | Yes | Machine-readable schema, modality metadata, labels, padding, QC, split rules |
| `versions.toml` | Recommended | Lance version changelog |
| `description.toml` | No | Human-readable dataset overview and processing notes |

## Profiles

The specification has two profiles that share the same layout, Lance, metadata, version, and description rules:

- **base** (default): supervised / downstream datasets. A `[label]` section is required when the dataset is supervised. This is the profile assumed by the core rule documents.
- **pretrain**: self-supervised EEG pretraining corpora. Datasets are unlabeled, stored one table per dataset (OpenNeuro ships many datasets), carry both `electrode_ids` and `channel_names`, use slot-mask channel padding, and standardize NPD features under `derivatives/features/npd.lance`. A dataset selects this profile with `profile = "pretrain"` at the top of `metadata.toml`. See [pretrain-rules.md](rules/pretrain-rules.md).

## Rules navigation

| Topic | English | Chinese |
|---|---|---|
| Dataset layout | [dataset-layout-rules.md](rules/dataset-layout-rules.md) | [dataset-layout-rules.cn.md](rules/dataset-layout-rules.cn.md) |
| Lance table | [lance-rules.md](rules/lance-rules.md) | [lance-rules.cn.md](rules/lance-rules.cn.md) |
| Metadata | [metadata-rules.md](rules/metadata-rules.md) | [metadata-rules.cn.md](rules/metadata-rules.cn.md) |
| Versioning | [version-rules.md](rules/version-rules.md) | [version-rules.cn.md](rules/version-rules.cn.md) |
| Description | [description-rules.md](rules/description-rules.md) | [description-rules.cn.md](rules/description-rules.cn.md) |
| Pretrain profile | [pretrain-rules.md](rules/pretrain-rules.md) | [pretrain-rules.cn.md](rules/pretrain-rules.cn.md) |
| Validation checklist | [validation-checklist.md](validation/validation-checklist.md) | - |

## Recommended reading order

1. [Dataset layout rules](rules/dataset-layout-rules.md) / [中文](rules/dataset-layout-rules.cn.md)
2. [Lance table rules](rules/lance-rules.md) / [中文](rules/lance-rules.cn.md)
3. [Metadata rules](rules/metadata-rules.md) / [中文](rules/metadata-rules.cn.md)
4. [Version rules](rules/version-rules.md) / [中文](rules/version-rules.cn.md)
5. [Description rules](rules/description-rules.md) / [中文](rules/description-rules.cn.md)
6. [Pretrain profile rules](rules/pretrain-rules.md) / [中文](rules/pretrain-rules.cn.md)
7. [Validation checklist](validation/validation-checklist.md)

## Current default schema

The main Lance table should contain at least:

```text
sample_id: string
subject_id: string
modality: string
data: fixed_size_list<float32> or list<float32>
shape: list<int32>
original_shape: list<int32>
valid_length: int64
qc_pass: bool
```

It should also contain, when applicable:

```text
session_id: string
run_id: string
label: int32 / float32 / string
label_name: string
split: string
```

## Modality-specific metadata

For EEG datasets, `metadata.toml` must describe channel layout, channel types, channel order rules, channel count, sampling rate, unit, reference, and montage. Variable channel layouts must declare the Lance columns used to read per-subject or per-sample channel names and status.

For MRI datasets, `metadata.toml` must describe image type, magnetic field strength, scanner information, voxel size, orientation, space, spatial axes, and affine handling.

For fMRI datasets, `metadata.toml` must describe BOLD type, TR, voxel size, space, spatial axes, time axis, and core preprocessing status.

## Examples

Example TOML files are provided under:

```text
examples/eeg/
examples/mri/
examples/fmri/
examples/eeg-pretrain/   # pretrain profile (unlabeled EEG, one table per dataset, NPD features)
```

## Versioning

There are three version concepts:

| Version | Location | Meaning |
|---|---|---|
| `spec_version` | TOML files | Version of this specification |
| `dataset.version` | `metadata.toml` | Semantic dataset release version |
| `lance_version` | `versions.toml` | Lance internal table version |

## How to evolve this repository

When changing this specification:

1. Update the relevant rule file under `rules/`.
2. Update examples if the expected TOML structure changes.
3. Update `validation/validation-checklist.md` if validation behavior changes.
4. Update `CHANGELOG.md` with a concise description.
5. Keep backward compatibility unless the change is explicitly marked as breaking.
