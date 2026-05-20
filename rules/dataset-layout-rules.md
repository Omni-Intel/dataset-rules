# Dataset Layout Rules

## 1. Default dataset layout

Each dataset should be stored in a single directory named by its dataset ID:

```text
<dataset_id>/
  dataset.lance/
  metadata.toml
  versions.toml
  description.toml
```

## 2. Required and optional files

### MUST

- A dataset MUST contain `dataset.lance/`.
- A dataset MUST contain `metadata.toml`.
- `dataset.lance/` MUST be the default main Lance table name unless otherwise declared in `metadata.toml`.
- `metadata.toml` MUST be sufficient for a program to parse and restore data from `dataset.lance`.

### SHOULD

- A dataset SHOULD contain `versions.toml`.
- `versions.toml` SHOULD describe every meaningful Lance table version.
- Relative paths SHOULD be used inside TOML files when the dataset directory is self-contained.

### MAY

- A dataset MAY contain `description.toml`.
- A dataset MAY contain a `derivatives/` directory for derived features, reports, QC artifacts, or model outputs.

## 3. Extended layout

For derived data or reports:

```text
<dataset_id>/
  dataset.lance/
  metadata.toml
  versions.toml
  description.toml
  derivatives/
    features/
    reports/
    qc/
```

## 4. Single-modality rule

For current single-modality datasets, one `dataset.lance/` table is preferred. The table should store sample-level data, labels, IDs, shape metadata, QC flags, and splits.

## 5. Multimodal extension rule

For future multimodal datasets:

- If modalities are strictly aligned and always read together, a single wide `dataset.lance/` table MAY be used.
- If modalities are not strictly aligned, each modality SHOULD be stored separately and linked by a manifest or relation table.
- The current specification reserves this extension but defaults to the single-main-table design.

