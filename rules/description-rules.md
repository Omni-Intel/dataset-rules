# Description Rules

## 1. Purpose

`description.toml` is optional. It is intended for humans and platform display pages.

It SHOULD NOT be required for parsing `dataset.lance/`. Machine-readable details belong in `metadata.toml`.

## 2. Recommended sections

```toml
[overview]
[collection]
[processing_summary]
[intended_use]
[limitations]
[citation]
```

## 3. Example

```toml
[overview]
title = "HMC Sleep EEG Dataset"
summary = "A sleep staging EEG dataset converted into Lance format."
modality = "eeg"
task = "sleep_stage_classification"

[collection]
source = "HMC"
population = "Sleep study participants"
collection_period = "unknown"
institution = "unknown"

[processing_summary]
description = "Raw EEG recordings were segmented into fixed-length windows and converted to padded Lance rows."
window_sec = 30.0
normalization = "none"
label_mapping = "Sleep stage labels were mapped to integer class IDs."

[intended_use]
tasks = ["classification", "representation_learning"]
recommended_split = "Use the split column in dataset.lance."

[limitations]
notes = [
  "Channel information should be checked before cross-dataset training.",
  "Labels depend on the original annotation quality."
]

[citation]
text = "Please cite the original dataset and this converted release."
```

