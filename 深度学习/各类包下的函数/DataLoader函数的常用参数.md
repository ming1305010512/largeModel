[TOC]



------

## 1）DataLoader 是干嘛的

它把你的 `Dataset` 变成一个“**可迭代的批次流**”：

- 每次迭代给你一个 batch（如 `images, labels`）
- 负责：**打乱、分批、并行读取、拼 batch、把最后不足一批怎么处理、是否把数据提前拷到内存等**

典型用法：

```python
train_loader = DataLoader(train_dataset, batch_size=64, shuffle=True, num_workers=4)
for x, y in train_loader:
    ...
```

------

## 2）最常用参数（日常 90% 就用这几个）

### A. `dataset`

**数据集对象**，必须实现 `__len__`（可选）和 `__getitem__`（必须）。

- map-style：`Dataset`（索引式）最常见
- iterable-style：`IterableDataset`（流式）也支持，但有些参数行为不同（后面说）

------

### B. `batch_size`

每个 batch 里样本数。

- `batch_size=None` 表示不自动组 batch（一般少用）
- 训练：大一点速度快但显存吃紧；验证：可更大一些

------

### C. `shuffle`

是否在每个 epoch 开始时**打乱样本顺序**（只对 map-style dataset 生效）。

- 训练集常用 `True`
- 验证/测试集一般 `False`

> 注意：如果你用了 `sampler`，通常就不要再用 `shuffle=True`（两者冲突，二选一）。

------

### D. `drop_last`

如果最后剩下不足一个 batch 的样本，是否丢弃。

- `drop_last=True`：保证每个 batch 形状完全一致
  适合：BN（BatchNorm）更稳定、分布式训练（DDP）对齐步数、某些需要固定 batch 的代码
- `drop_last=False`：保留最后小 batch
  适合：验证/测试不想浪费数据

------

### E. `num_workers`

**并行读数据的进程数**（不是线程），用来加速 I/O 和预处理。

- `0`：主进程读（最稳，但可能慢）
- `>0`：多进程并行读（通常更快）

经验：

- Windows 下多进程更容易踩坑（必须加 `if __name__ == "__main__":`）
- 太大不一定快：会有进程切换、磁盘瓶颈、CPU瓶颈

------

### F. `pin_memory`

是否把 batch 放到 **页锁定内存**（pinned memory），以便更快拷贝到 GPU。

- 用 GPU 训练时常设 `pin_memory=True`
- CPU 训练时意义不大

通常搭配：

```python
for x, y in loader:
    x = x.to(device, non_blocking=True)
```

------

## 3）采样相关（控制“取哪些样本、以什么顺序取”）

### A. `sampler`

自定义采样器（比如类别不均衡时加权采样）。
常见：`WeightedRandomSampler`

> 一旦用了 `sampler`，一般不要 `shuffle=True`。

------

### B. `batch_sampler`

比 `sampler` 更高级：它直接产出“一个 batch 的索引列表”。
用了 `batch_sampler` 后，`batch_size/shuffle/sampler/drop_last` 通常由它接管。

------

## 4）拼 batch 相关（遇到 “stack expects each tensor…” 的核心点）

### A. `collate_fn`

默认的 `collate_fn` 会把一个 batch 的样本用 `torch.stack` 拼起来。
如果你的样本长度不一致（如文本序列、不同尺寸图片），就会报类似：

> stack expects each tensor to be equal size ...

解决方式：自定义 `collate_fn`（比如 padding、或者返回 list 不 stack）。

例：对变长序列做 padding（示意）：

```python
from torch.nn.utils.rnn import pad_sequence

def collate_fn(batch):
    xs, ys = zip(*batch)
    xs = [torch.tensor(x) for x in xs]
    xs_pad = pad_sequence(xs, batch_first=True, padding_value=0)
    ys = torch.tensor(ys)
    return xs_pad, ys

loader = DataLoader(ds, batch_size=32, collate_fn=collate_fn)
```

------

## 5）性能/稳定性进阶参数（加速与“少踩坑”）

### A. `persistent_workers`

`True`：每个 epoch 不重建 worker 进程（减少开销）

- 只有当 `num_workers > 0` 才有意义
- 训练多 epoch 时更稳更快

### B. `prefetch_factor`

每个 worker 预取多少个 batch（默认通常是 2）。

- 如果 CPU 预处理重、GPU 很快，适当调大可能更顺滑
- 太大可能占用更多内存

### C. `timeout`

worker 获取数据超时时间（秒）。

- 数据读取偶发卡死时可用来“快速暴露问题”
- 但根因通常是 Dataset/IO/多进程问题

### D. `worker_init_fn`

初始化每个 worker 用（比如设置随机种子、打开不同的文件句柄）。

- 用于保证增强随机性可控、或者避免多进程共享资源冲突

### E. `generator`

控制 shuffle 的随机性（可复现训练）。

```python
g = torch.Generator()
g.manual_seed(42)
loader = DataLoader(ds, shuffle=True, generator=g)
```

------

## 6）几个典型配置模板（直接抄就能用）

### 训练集（GPU）

```python
train_loader = DataLoader(
    train_ds,
    batch_size=64,
    shuffle=True,
    num_workers=4,
    pin_memory=True,
    drop_last=True,
    persistent_workers=True,
)
```

### 验证集

```python
val_loader = DataLoader(
    val_ds,
    batch_size=128,
    shuffle=False,
    num_workers=4,
    pin_memory=True,
    drop_last=False
)
```

### 类别不均衡（加权采样）

```python
from torch.utils.data import WeightedRandomSampler
sampler = WeightedRandomSampler(weights, num_samples=len(weights), replacement=True)

train_loader = DataLoader(train_ds, batch_size=64, sampler=sampler, num_workers=4)
```

### 变长序列/不等尺寸（自定义 collate）

```python
train_loader = DataLoader(train_ds, batch_size=32, collate_fn=my_collate_fn)
```

------

## 7）最常见坑

1. **`stack expects each tensor to be equal size`**
   → 99% 是默认 collate 在 stack，样本 shape 不一致
   → 自定义 `collate_fn` 做 padding/resize/返回 list
2. **Windows 多进程卡死/报错**
   → 训练入口必须放进 `if __name__ == "__main__":`
3. **`shuffle=True` + `sampler` 冲突**
   → 二选一
4. **`drop_last=True` 导致验证集少算一部分样本**
   → 验证/测试一般 `drop_last=False`
5. **GPU 拷贝慢**
   → `pin_memory=True` + `.to(device, non_blocking=True)` + 合理 `num_workers`