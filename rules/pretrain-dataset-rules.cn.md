# 预训练数据集规则

## 1. 目的

本文定义预训练数据集的扩展规则。

预训练数据集 MUST 复用基础数据集契约：

```text
dataset.lance/
metadata.toml
versions.toml
description.toml
```

需要扩展的原因是，预训练数据集通常与下游数据集有四点不同：

- 会合并多个 source dataset；
- 通常是自监督或多任务预训练，不一定有任务标签；
- 为了规模、通道 profile 或 batch 效率，可能包含多个物理 Lance 表；
- 可能包含与信号表逐行对齐的派生特征视图。

这些差异 MUST 在 `metadata.toml` 中声明。`description.toml` MUST NOT 成为解析、校验、恢复、采样或训练的必需文件。

## 2. 与下游规则的关系

除非本文显式定义扩展，预训练数据集 MUST 兼容基础 Lance 和 metadata 规则。

逻辑信号表仍然 MUST 包含基础必需列。对于分区预训练数据集，每个发布的
signal 表 MUST 包含这些物理列：

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

对于单表预训练数据集，`sample_id` SHOULD 继续是全局唯一、稳定的字符串，
与下游规范保持一致。

对于分区预训练数据集，当如下声明存在时，`sample_id` MAY 只在表内唯一：

```toml
[pretrain]
sample_id_scope = "table_local"
sample_key_columns = ["table_id", "sample_id"]
```

在 `table_local` 模式下，全局唯一样本键是复合键
`(table_id, sample_id)`。`table_id` MUST 与该物理 signal 表对应的
`[[pretrain.tables]].table_id` 一致，`sample_id` MUST 在该表内唯一。数据集
MAY 物化一个全局唯一字符串 `sample_uid`，例如
`"tuh_train:000000001"`，但当 `sample_id_scope = "table_local"` 时，
validator MUST 将 `(table_id, sample_id)` 视为权威样本键。

如果源表里存在原始局部行 ID，SHOULD 存储为 `source_sample_id` 或
`local_sample_id`；除非它也满足声明的样本键契约，否则不得作为预训练样本键。

预训练数据集 SHOULD 设置：

```toml
[dataset]
task_type = "pretraining"
```

预训练数据集中的监督 `[label]` 元数据是 OPTIONAL。如果为了辅助训练或评估包含标签，标签 MUST 被声明为可选，并且读取预训练样本时 MUST NOT 依赖标签存在。

## 3. 布局

### 3.1 推荐单主表布局

可行时，预训练数据集 SHOULD 暴露一个 canonical signal table：

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

`dataset.lance/` 仍然是 canonical signal table。派生特征表放在 `derivatives/` 下，并且 MUST 在 `metadata.toml` 中声明。

### 3.2 分区布局

当单表因为规模、通道 profile、source 分离或 batch 效率而不现实，预训练数据集 MAY 使用多个物理 Lance 表。

分区布局 MUST 在 `[[pretrain.tables]]` 中声明每个物理表。所有声明的表共同组成一个逻辑数据集。validator 和 reader MUST 将所有声明的 signal 表的并集视为数据集表面。

分区布局 MUST 设置：

```toml
[storage]
table_name = "__partitioned__"
lance_path = "__partitioned__"
```

`__partitioned__` 是一个哨兵值，表示不存在单一物理 `dataset.lance/`
作为 canonical signal table。reader 和 validator MUST 从所有
`[[pretrain.tables]]` 中 `role = "signal"` 的条目解析逻辑信号表。

示例：

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

任何生成的 registry 或 index 文件都只是实现缓存。除非它被显式声明为
`[[pretrain.tables]]` 且 `role = "index"`，否则 MUST NOT 成为解析、校验、恢复、采样或训练的必需文件。

## 4. 必需预训练元数据

当 `dataset.task_type = "pretraining"` 时，`metadata.toml` MUST 包含：

```toml
[pretrain]
```

`[pretrain]` 必需字段：

| Field | Type | Required | Description |
|---|---|---:|---|
| `pretrain_type` | string | Yes | `self_supervised`、`supervised`、`multi_objective` 或 `contrastive` |
| `sample_id_scope` | string | Yes | `global` 或 `table_local` |
| `sample_key_columns` | array[string] | Conditional | 当 `sample_id_scope = "table_local"` 时必需；SHOULD 为 `["table_id", "sample_id"]` |
| `source_dataset_column` | string | Yes | 存储 source dataset ID 的列 |
| `recording_id_column` | string | Yes | 标识源 recording 的列 |
| `preprocess_version` | string | Yes | 默认或主要预处理 pipeline 版本；行级 `preprocess_version` 是 ground truth |
| `split_policy` | string | Yes | split 隔离策略 |

推荐字段：

```toml
objective_family = "masked_prediction"
feature_view_ids = ["npd_v1"]
sampling_policy = "balanced_by_source"
sample_uid_column = "sample_uid"
```

当 `sample_id_scope = "table_local"` 时，`[schema].primary_key` MUST 写为：

```toml
[schema]
primary_key = ["table_id", "sample_id"]
```

这是对下游基础约定 `primary_key = "sample_id"` 的预训练扩展。单表预训练数据集
SHOULD 继续使用基础约定，除非明确需要 table-local key。

## 5. 必需预训练列

除基础列外，每个发布的 signal 表 MUST 包含以下物理列：

| Column | Type | Required | Description |
|---|---|---:|---|
| `source_dataset_id` | string | Yes | 源数据集 ID，例如 `tuh`、`ds004395` |
| `recording_id` | string | Yes | 源数据集内部稳定 recording ID |
| `source_path` | string | Yes | 源文件相对路径或源 URI |
| `start_time` | float64 | Yes | 窗口起始时间，单位秒 |
| `duration` | float64 | Yes | 窗口长度，单位秒 |
| `split` | string | Yes | 预训练 split |
| `preprocess_version` | string | Yes | 行级预处理版本；当它与 `[pretrain].preprocess_version` 不同时，以行级值为准 |
| `channel_profile` | string | EEG 必需 | 通道 profile ID |

推荐列：

| Column | Type | Description |
|---|---|---|
| `table_id` | string | 物理表 ID；分区数据集推荐物理存储，且当 `sample_id_scope = "table_local"` 时 MUST 作为 reader 可用字段 |
| `sample_uid` | string | 可选物化全局唯一样本键，由 `table_id` 和 `sample_id` 派生 |
| `subject_uid` | string | 跨 source 作用域稳定 subject ID |
| `recording_uid` | string | 跨 source 作用域稳定 recording ID |
| `source_sample_id` | string/int64 | source-local sample ID |
| `sample_weight` | float32 | 可选采样权重 |
| `license` | string | 源数据许可或访问条款 |
| `access_level` | string | `open`、`registered`、`restricted`、`internal` 或 `unknown` |

当 `split_policy = "subject_disjoint"` 时，split 隔离 MUST 使用
`subject_uid`；如果不存在 `subject_uid`，则使用复合键
`(source_dataset_id, subject_id)`。

当 `split_policy = "recording_disjoint"` 时，split 隔离 MUST 使用
`recording_uid`；如果不存在 `recording_uid`，则使用复合键
`(source_dataset_id, recording_id)`。

## 6. Source Dataset 元数据

每个 source dataset MUST 被声明：

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

必需字段：

| Field | Type | Required | Description |
|---|---|---:|---|
| `source_dataset_id` | string | Yes | Lance 行中使用的 source ID |
| `source_family` | string | Yes | source 家族，例如 `tuh`、`openneuro` |
| `version` | string | Yes | source snapshot 或 release 版本 |
| `license` | string | Yes | license 或访问条款 |
| `access_level` | string | Yes | 访问类别 |
| `n_samples` | int | Yes | 该 source 发布的样本数 |

同一个预训练数据集内，`source_dataset_id` 值 MUST 唯一。

`access_level` MUST 是以下值之一：

```text
open
registered
restricted
internal
unknown
```

`source_family` SHOULD 使用稳定的小写标识符，例如 `tuh`、`openneuro`、
`sleep-edf` 或 `internal`。新增 source family MAY 不修改规范即可引入。

## 7. 物理表元数据

分区数据集 MUST 声明每个表：

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

必需字段：

| Field | Type | Required | Description |
|---|---|---:|---|
| `table_id` | string | Yes | 稳定表 ID；同一个预训练数据集内 MUST 唯一 |
| `role` | string | Yes | `signal`、`feature`、`index` 或 `manifest` |
| `lance_path` | string | Yes | 相对或绝对 Lance 路径 |
| `n_samples` | int | Yes | 行数 |

推荐字段：

```toml
split = "train"
source_family = "openneuro"
source_dataset_id = "ds004395"
channel_profile = "openneuro_slot136"
view_id = "npd_v1"
aligned_to = "openneuro_slot136_train"
alignment = "sample_id"
```

`alignment` MUST 是 `sample_id`、`sample_key`、`sample_uid` 或
`row_index`。

如果 `alignment = "sample_id"`，feature 表 MUST 包含与其对齐 signal 表相同的
`sample_id` 集合。在 `table_local` 模式下，只有当 `aligned_to` 明确指向一个
signal 表时，这种对齐才没有歧义。

如果 `alignment = "sample_key"`，feature 表 MUST 包含 `signal_table_id` 和
`sample_id`；二者共同引用对齐 signal 表中的 `(table_id, sample_id)`。当
`sample_id_scope = "table_local"` 且一个 feature 表混合多个 signal 表的样本时，
SHOULD 使用 `sample_key`。

如果 `alignment = "sample_uid"`，feature 表 MUST 包含由
`[pretrain].sample_uid_column` 声明的物化 `sample_uid` 列。

如果 `alignment = "row_index"`，feature 表 MUST 包含显式 `row_index` 列，且该列值引用声明的 `aligned_to` signal 表中的行索引。发布数据集 MUST NOT 使用纯隐式 row-order 对齐。

## 8. 特征视图

派生特征 SHOULD 作为 feature view 存储，而不是替代 canonical signal table 中的 signal data。

每个 feature view MUST 被声明：

```toml
[[pretrain.feature_views]]
view_id = "npd_v1"
feature_type = "handcrafted_descriptor"
derived_from = "signal"
feature_version = "2026.05.29"
alignment = "sample_id"
required_for_training = false
```

Feature 表 MUST 包含：

```text
sample_id
```

当 `alignment = "sample_id"` 时。Feature 表在 `alignment = "sample_key"` 时
MUST 包含 `signal_table_id` 和 `sample_id`；在 `alignment = "sample_uid"` 时
MUST 包含 `sample_uid`；在 `alignment = "row_index"` 时 MUST 包含 `row_index`。

Feature 表还 MUST 包含 `feature_version`，或在 `[[pretrain.feature_views]]`
中声明单一 feature version。当行级 `feature_version` 与 feature-view 级默认值不同时，以行级值为准。

## 9. EEG Channel Profiles

当 EEG 预训练数据集使用多个通道全集或 slot layout 时，SHOULD 声明 channel profiles：

```toml
[[pretrain.channel_profiles]]
profile_id = "tuh_named26"
n_channels = 26
layout = "fixed_slot"
channel_mask_column = "channel_mask"
electrode_id_column = "electrode_ids"
```

`profile_id` 值 MUST 与 `channel_profile` 列值匹配。validator MUST NOT 从物理表名推断通道数。

`layout` MUST 是以下值之一：

```text
fixed_slot
variable
named_only
```

`[eeg].channel_layout` 描述通道名称如何存储在 Lance 行中。
`[[pretrain.channel_profiles]].layout` 描述一个预训练 channel profile 内部的 slot layout。

对于多模态预训练数据集中的非 EEG 行，`channel_profile` MUST 为 null 或空字符串。validator MUST 仅对 `modality = "eeg"` 的行要求非空且已声明的 `channel_profile`。

## 10. Split 规则

预训练 split MUST 稳定、可复现。

`[pretrain]` MUST 声明 `split_policy`。它 MUST 是以下值之一：

```text
subject_disjoint
recording_disjoint
source_dataset_disjoint
custom_declared
```

如果 `split_policy = "custom_declared"`，`[pretrain]` 还 MUST 声明
`split_declaration_path`，且引用文件 MUST 描述 split 隔离规则。

对于 EEG，同一个 recording 产生的 windows MUST NOT 出现在多个 split 中，除非数据集显式声明非泄漏例外及其预期用途。

## 11. Sampling 规则

预训练 metadata SHOULD 声明默认采样策略：

```toml
[pretrain.sampling]
policy = "balanced_by_source"
source_weighting = "temperature"
temperature = 0.5
weight_column = "sample_weight"
```

当行数分布不应直接等于训练采样概率时，采样策略元数据是必需的。

如果声明了 `weight_column`，被引用列 MUST 存在于 sampler 使用的每个 signal 表中。

## 12. 校验

预训练校验 MUST 包含所有基础数据集校验，并额外检查：

- 如果 `sample_id_scope = "global"`，`sample_id` 在所有声明的 signal 表中全局唯一。
- 如果 `sample_id_scope = "table_local"`，每个声明的 signal 表有唯一
  `table_id`，`sample_id` 在各自表内唯一，且复合键 `(table_id, sample_id)`
  是权威全局样本键。
- 所有声明的 `[[pretrain.sources]]` ID 都出现在行中，或被显式标记为 unused。
- 所有声明的 `[[pretrain.tables]]` 路径存在，且行数匹配。
- Signal 表行包含必需预训练物理列。
- Feature views 按 `sample_id`、`sample_key`、`sample_uid` 或显式
  `row_index` 与 signal 表对齐。
- split policy 已声明；当相应列存在时，按 subject 或 recording 级别验证。
- EEG `channel_profile` 值均已声明，且其通道数与恢复 shape 匹配。
- 当 source 分布被有意重加权时，sampling policy 已声明。
