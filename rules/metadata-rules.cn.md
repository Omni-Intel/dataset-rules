# 元数据规则

## 1. 目的

`metadata.toml` 是必需的机器可读元数据文件。它描述如何解析、校验、恢复并解释 `dataset.lance/`。

它不只是数据集简介。人类可读摘要属于 `description.toml`。

## 2. 必需顶层章节

`metadata.toml` MUST 包含：

```toml
spec_version = "0.1.0"

[dataset]
[storage]
[schema]
[[schema.columns]]
[data]
[quality_control]
```

它还 MUST 根据 `dataset.modality` 包含一个模态特定章节：

```toml
[eeg]   # modality = "eeg" 时必需
[mri]   # modality = "mri" 时必需
[fmri]  # modality = "fmri" 时必需
```

如果数据集是监督学习数据集，它 MUST 包含：

```toml
[label]
```

## 3. `[dataset]` 必需字段

| Field | Type | Required | Description |
|---|---|---:|---|
| `dataset_id` | string | Yes | 唯一数据集 ID |
| `name` | string | Yes | 数据集名称 |
| `modality` | string | Yes | `eeg`、`mri`、`fmri` 或已声明模态 |
| `version` | string | Yes | 语义化数据集版本 |
| `created_at` | datetime/string | Yes | 创建时间 |
| `updated_at` | datetime/string | Yes | 最近更新时间 |
| `sample_unit` | string | Yes | `window`、`epoch`、`image`、`volume`、`run` 等 |
| `n_samples` | int | Yes | 样本数量 |

推荐字段：

```toml
description = "..."
task_type = "classification"
n_subjects = 100
```

## 4. `[storage]` 必需字段

| Field | Type | Required | Description |
|---|---|---:|---|
| `table_name` | string | Yes | 默认：`dataset.lance` |
| `lance_path` | string | Yes | 相对或绝对 Lance 路径 |
| `path_type` | string | Yes | `relative` 或 `absolute` |

推荐字段：

```toml
root_path = "."
backend = "local"  # local, s3, tos, oss, cos, etc.
uri = ""
```

## 5. `[schema]` 规则

`[schema]` MUST 声明：

```toml
primary_key = "sample_id"
data_column = "data"
```

如果是监督学习数据集：

```toml
label_column = "label"
```

`dataset.lance/` 中的每一列 MUST 通过 `[[schema.columns]]` 描述。

每个 `[[schema.columns]]` 条目 MUST 包含：

| Field | Type | Required | Description |
|---|---|---:|---|
| `name` | string | Yes | 列名 |
| `type` | string | Yes | Lance/Arrow 逻辑类型 |
| `required` | bool | Yes | 是否为规范或数据集要求的必需列 |
| `nullable` | bool | Yes | 是否允许 null 值 |
| `description` | string | Yes | 人类可读字段含义 |

推荐可选字段：

```toml
allowed_values = ["train", "val", "test"]
unit = "uV"
shape = ["n_channels", "n_times"]
```

## 6. 必需基础列

`metadata.toml` MUST 包含以下列的 schema 定义：

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

它 SHOULD 包含以下列的定义：

```text
session_id
run_id
label
label_name
split
```

## 7. `[data]` 必需字段

| Field | Type | Required | Description |
|---|---|---:|---|
| `array_column` | string | Yes | 数据列名称 |
| `dtype` | string | Yes | 数组数据类型 |
| `axis_order` | list<string> | Yes | 恢复后的轴顺序 |
| `padding` | bool | Yes | 是否使用 padding |
| `padding_value` | number | Yes | Padding 填充值 |
| `shape_column` | string | Yes | Padded shape 列 |
| `original_shape_column` | string | Yes | Original shape 列 |
| `valid_length_column` | string | Yes | 有效展平长度列 |

## 8. `[label]` 规则

如果是监督学习数据集，`[label]` MUST 声明：

```toml
label_column = "label"
task_type = "classification"  # classification, regression, multilabel
```

对于分类任务，还 MUST 声明：

```toml
num_classes = 5
```

并声明每个类别：

```toml
[[label.classes]]
id = 0
name = "class_name"
description = "..."
```

对于回归任务，SHOULD 声明：

```toml
target_name = "age"
target_unit = "year"
```

## 9. `[quality_control]` 必需字段

```toml
[quality_control]
qc_column = "qc_pass"
```

推荐字段：

```toml
qc_pass_values = [true, false]
qc_method = "..."
```

## 10. EEG 必需元数据

当 `dataset.modality = "eeg"` 时，`[eeg]` MUST 包含：

| Field | Type | Required | Description |
|---|---|---:|---|
| `n_channels` | int | Yes | 通道数量 |
| `sampling_rate` | float | Yes | 采样率 |
| `sampling_rate_unit` | string | Yes | 通常为 `Hz` |
| `unit` | string | Yes | 信号单位，例如 `uV` |
| `reference` | string | Yes | 参考设置 |
| `montage` | string | Yes | 电极 montage |
| `channel_axis` | int | Yes | 恢复后的通道轴 |
| `time_axis` | int | Yes | 恢复后的时间轴 |

它 MUST 为每个通道包含 `[[eeg.channels]]` 条目。

每个通道 MUST 包含：

| Field | Type | Required | Description |
|---|---|---:|---|
| `index` | int | Yes | 数据中的通道顺序 |
| `name` | string | Yes | 通道名称 |
| `type` | string | Yes | `eeg`、`eog`、`emg`、`ecg`、`stim`、`misc`、`ref` |
| `unit` | string | Yes | 通道单位 |

推荐通道字段：

```toml
status = "good"  # good, bad, unknown
x = 0.0
y = 0.0
z = 0.0
```

## 11. MRI 必需元数据

当 `dataset.modality = "mri"` 时，`[mri]` MUST 包含：

| Field | Type | Required | Description |
|---|---|---:|---|
| `image_type` | string | Yes | `T1w`、`T2w`、`FLAIR`、`DWI` 等 |
| `field_strength` | float | Yes | 磁场强度 |
| `field_strength_unit` | string | Yes | 通常为 `T` |
| `manufacturer` | string | Yes | 扫描仪厂商或 `unknown` |
| `scanner_model` | string | Yes | 扫描仪型号或 `unknown` |
| `voxel_size` | list<float> | Yes | 体素大小 |
| `voxel_size_unit` | string | Yes | 通常为 `mm` |
| `orientation` | string | Yes | `RAS`、`LAS` 等 |
| `space` | string | Yes | `native`、`MNI152` 等 |
| `spatial_axes` | list<string> | Yes | 通常为 `["x", "y", "z"]` |
| `affine_column` | string | Yes | 包含展平 affine 的 Lance 列，或空字符串 |

推荐字段：

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

## 12. fMRI 必需元数据

当 `dataset.modality = "fmri"` 时，`[fmri]` MUST 包含：

| Field | Type | Required | Description |
|---|---|---:|---|
| `bold_type` | string | Yes | 通常为 `BOLD` |
| `tr` | float | Yes | Repetition time |
| `tr_unit` | string | Yes | 通常为 `s` |
| `space` | string | Yes | `native`、`MNI152` 等 |
| `voxel_size` | list<float> | Yes | 体素大小 |
| `voxel_size_unit` | string | Yes | 通常为 `mm` |
| `spatial_axes` | list<string> | Yes | 空间轴 |
| `time_axis` | int | Yes | 时间轴索引 |
| `slice_timing_corrected` | bool | Yes | 是否已进行 slice timing correction |
| `motion_corrected` | bool | Yes | 是否已进行 motion correction |
| `normalized` | bool | Yes | 是否已归一化到模板空间 |
