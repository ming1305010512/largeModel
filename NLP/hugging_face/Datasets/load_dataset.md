 

[TOC]

Hugging Face Datasets** 里的加载入口：`from datasets import load_dataset`。它的定位是：**把 Hub 或本地的数据文件（csv/json/parquet/text…）加载成 `Dataset / DatasetDict`（或流式的 `IterableDataset`）**，并自动做下载、解压、解析、缓存（Arrow）等工作。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

------

## 1) `load_dataset` 会返回什么？

- **`split` 传了**：返回一个 **`Dataset`**（或 `streaming=True` 时返回 `IterableDataset`）([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))
- **`split=None`（默认）**：返回一个 **`DatasetDict`**（包含 train/validation/test 等所有 split）([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))
- **`streaming=True`**：返回 **`IterableDataset / IterableDatasetDict`**，按迭代“边读边用”，不做完整下载和本地 Arrow 缓存；但它不适合随机访问（比如 `ds[123]` 这类）。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

------

## 2) 你最常用的参数（按重要程度）

下面是文档里 `load_dataset(...)` 的核心参数（我按“你实际会经常用到”的角度解释）。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

### A. `path`（必填）

**数据从哪来**：

- Hub 上的数据集：`"username/dataset_name"`
- 本地目录：`"./data_dir"`
- 或者指定通用 builder：`"csv" / "json" / "parquet" / "text" ...` 再配 `data_files` ([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

### B. `name`（可选）

**配置名 / 子数据集**（比如 GLUE 的 `sst2`）。
示例：`load_dataset("nyu-mll/glue", "sst2", split="train")` ([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

### C. `data_files`（很常用）

**指定加载哪些文件**，也可以**手动映射到 split**：

```python
data_files = {"train": "train.csv", "test": "test.csv"}
ds = load_dataset("namespace/your_dataset_name", data_files=data_files)
```

([Hugging Face](https://huggingface.co/docs/datasets/v2.21.0/en/loading))

> 小细节：`data_files` / `data_dir` 允许相对路径，它会以“数据集加载的基路径”为参照来解析。([Hugging Face](https://huggingface.co/docs/datasets/v2.21.0/en/loading))

### D. `data_dir`（常用）

当一个仓库/目录里文件很多，你只想加载某个子目录：

```python
ds = load_dataset("allenai/c4", data_dir="en")
```

([Hugging Face](https://huggingface.co/docs/datasets/v2.21.0/en/loading))

### E. `split`（非常常用）

**指定加载哪个 split**，并且支持“TFDS 风格的组合/切片语法”。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

典型玩法（都很常见）：

```python
# 1) 单个 split
train_ds = load_dataset("bookcorpus", split="train")

# 2) 同时取多个 split（返回两个 Dataset）
train_ds, test_ds = load_dataset("bookcorpus", split=["train", "test"])

# 3) 拼接 split（合成一个 Dataset）
train_test = load_dataset("bookcorpus", split="train+test")

# 4) 切片：取第 10~20 条
sub = load_dataset("bookcorpus", split="train[10:20]")

# 5) 切片：取前 10%
sub_pct = load_dataset("bookcorpus", split="train[:10%]")
```

([Hugging Face](https://huggingface.co/docs/datasets/v1.11.0/splits.html))

> 经验：如果你只想快速调通流程，`split="train[:1%]"` 这种特别省时间。

### F. `cache_dir`（很实用）

指定缓存目录（默认在 `~/.cache/huggingface/datasets`）。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))
当你磁盘分区/服务器目录有要求时必用。

### G. `streaming`（大数据必备）

```python
ds = load_dataset("bigcode/the-stack", split="train", streaming=True)
```

流式不会把全量数据下载/处理成 Arrow 缓存，而是迭代时再读。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))
但它的限制也很关键：**不适合随机访问**，更多用于训练这种“顺序迭代”的任务。([Hugging Face](https://huggingface.co/docs/datasets/en/stream))

### H. `revision`（可复现/固定版本）

指定 Hub 数据集的分支/Tag/commit：

```python
ds = load_dataset("lhoestq/custom_squad", revision="main")
```

([Hugging Face](https://huggingface.co/docs/datasets/v2.21.0/en/loading))

### I. `token`（私有数据集 / 需鉴权）

访问私有数据集或受限文件用。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

### J. `features`（你在做 ClassLabel、Sequence 等时会用到）

当你希望**加载时就指定 schema/类型**（比如把某列当作 `ClassLabel`），就用它。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

------

## 3) 不那么常用，但你迟早会遇到的参数

- `download_mode`：控制下载/复用策略（默认复用已有缓存）。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))
- `verification_mode`：控制校验强度（checksum/size/splits 等）。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))
- `keep_in_memory`：是否把数据拷进内存（更快但吃 RAM）。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))
- `num_proc`：下载/生成数据时用多进程。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))
- `storage_options`：实验性，给底层文件系统 backend 传参数（例如某些云存储场景）。([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

------

## 4) `trust_remote_code` 要不要管？

如果某些 Hub 数据集依赖“dataset script（需要执行仓库里的 Python 代码）”，通常会涉及 `trust_remote_code`：**只有在你信任且看过代码时才建议开**，因为它会在你本地执行对方仓库的代码。([Hugging Face](https://huggingface.co/docs/datasets/v2.18.0/en/package_reference/loading_methods))

------

## 5) 你可以直接抄的三段“常用模板”

### 模板 1：从 Hub 加载 + 只取一小段调试

```python
from datasets import load_dataset
ds = load_dataset("nyu-mll/glue", "sst2", split="train[:1%]")
```

([Hugging Face](https://huggingface.co/docs/datasets/en/package_reference/loading_methods))

### 模板 2：加载本地 CSV，并手动划分 train/test

```python
from datasets import load_dataset
data_files = {"train": "train.csv", "test": "test.csv"}
ds = load_dataset("csv", data_files=data_files)
```

([Hugging Face](https://huggingface.co/docs/datasets/v2.21.0/en/loading))

### 模板 3：大数据集用 streaming

```python
from datasets import load_dataset
ds = load_dataset("allenai/c4", data_dir="en", split="train", streaming=True)
```

([Hugging Face](https://huggingface.co/docs/datasets/v2.21.0/en/loading))