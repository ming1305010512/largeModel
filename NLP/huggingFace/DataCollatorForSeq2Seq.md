[TOC]

`DataCollatorForSeq2Seq` 是 Hugging Face `transformers` 里专门给 **Seq2Seq（BART/T5/MBART…）** 训练用的“**动态拼 batch 的打包工**”。它做的事非常关键：**把一堆样本（每个样本长短不一）在 batch 维度上对齐（padding），并把 labels 也对齐成训练能用的形状**，同时还能（可选）帮你准备 decoder 侧的一些东西。

你可以把它理解为：

> 你的 `Dataset.map(tokenize)` 只是把每条样本变成 token；
> `DataCollatorForSeq2Seq` 负责把“多条样本”合成一个 batch，并把 pad / label pad / decoder 相关细节处理好。

------

## 1）它解决的核心问题：动态 padding + label padding

### 为什么需要它？

Seq2Seq里每条样本长度不一样，比如：

- input_ids: `[101, 23, 45]`、`[101, 99]`
- labels: `[11, 12]`、`[13]`

PyTorch 一个 batch 里必须是同 shape 的 tensor，所以要 padding 成：

- input_ids → `[batch, max_src_len]`
- attention_mask → 同上
- labels → `[batch, max_tgt_len]`

而且**labels 的 padding 不能用普通 pad_token_id**，一般要用 `-100`，这样 loss 计算时会忽略 padding 位置（CrossEntropyLoss 会 ignore_index=-100）。

✅ `DataCollatorForSeq2Seq` 会自动：

- 对 input 侧做 padding
- 对 label 侧做 padding，并把 pad 部分改成 `label_pad_token_id`（默认 -100）

------

## 2）它一般会输出哪些键？

典型输出（训练时）会包含：

- `input_ids`：`[B, S]`
- `attention_mask`：`[B, S]`
- `labels`：`[B, T]`（pad部分是 -100）
- **有时还会有** `decoder_input_ids`（看设置/模型/Trainer流程）

你在 `Trainer` 里不用手写这些，collator 会把 batch 拼好。

------

## 3）最常用的初始化方式（你现在的 BART 对联训练就用这个）

```python
from transformers import DataCollatorForSeq2Seq

data_collator = DataCollatorForSeq2Seq(
    tokenizer=tokenizer,
    model=model,       # 建议传：能更好处理 decoder_input_ids / label 相关逻辑
    padding=True       # 动态 padding（每个 batch pad 到本 batch 的最长）
)
```

这就是 Hugging Face 官方最推荐的写法之一。

------

## 4）重要参数逐个讲（你一定会用到）

### 4.1 tokenizer（必填）

它用 tokenizer 的 `pad` 方法来完成动态 padding。

### 4.2 model（强烈建议传）

传了 `model` 的好处：

- 某些模型需要 `prepare_decoder_input_ids_from_labels`
- 可以更正确地生成 `decoder_input_ids`
- 兼容 label smoothing、不同架构的小差异

> 经验：**能传就传**，少踩坑。

### 4.3 padding（默认 True / “longest”）

- `True` / `"longest"`：pad 到 batch 内最长（省显存，最常用）
- `"max_length"`：pad 到 `max_length`（固定长度，利于某些编译/加速）
- `False`：不 pad（基本没法组 batch，不建议）

### 4.4 max_length / pad_to_multiple_of

- `max_length`：当 `padding="max_length"` 时使用
- `pad_to_multiple_of`：把长度 pad 到某个倍数（例如 8/16）
  - 常用于 GPU Tensor Core / fp16/bf16 加速

例子：

```python
DataCollatorForSeq2Seq(
    tokenizer=tokenizer,
    model=model,
    padding=True,
    pad_to_multiple_of=8
)
```

### 4.5 label_pad_token_id（默认 -100）

非常关键：

- labels pad 的地方用 -100
- loss 会忽略它

如果你把它改成 pad_token_id，会导致模型在 padding 位置也算 loss，训练会变怪。

------

## 5）它跟 `tokenizer(..., padding=True)` 有啥区别？

很多人会问：我在 `map` 时就 `padding=True` 不就行了？

区别很大：

### ✅ 推荐：在 collator 里动态 padding

- `map` 时只做 `truncation=True`，不要 pad
- collator 按 batch 动态 pad
- 更省内存、更快

### ❌ 不推荐：在 map 时 pad 到固定长度

- 所有样本都 pad 成同样长度，数据集变得更大
- I/O 更慢、占磁盘更大、训练更浪费显存

所以你的 tokenize 通常写成这样更好：

```python
tokenizer(texts, truncation=True, max_length=64)  # 不 padding
```

------

## 6）Seq2Seq 的“labels/decoder_input_ids”到底怎么走？

对 BART/T5 来说：

- 你提供 `labels`
- 模型内部会做 “shift right”（把 labels 右移）得到 decoder 输入（教师强制）
- pad 部分用 -100 会被忽略

**DataCollatorForSeq2Seq** 主要保证 `labels` pad 好、shape 对齐；是否显式提供 `decoder_input_ids`，取决于模型/Trainer流程。

你可以认为：

> 训练最关键的是 `labels` pad 成 -100；decoder_input_ids 这块通常交给模型/Trainer即可。

------

## 7）你做对联任务时的最佳实践（直接照抄）

### tokenize map（不 padding）

```python
def tokenize_fn(batch):
    model_inputs = tokenizer(batch["src"], truncation=True, max_length=64)
    with tokenizer.as_target_tokenizer():
        labels = tokenizer(batch["tgt"], truncation=True, max_length=64)
    model_inputs["labels"] = labels["input_ids"]
    return model_inputs
```

### collator

```python
data_collator = DataCollatorForSeq2Seq(
    tokenizer=tokenizer,
    model=model,
    padding=True,
    pad_to_multiple_of=8  # 可选
)
```

------

## 8）常见坑 & 排查清单

### 坑 1：labels 没有 -100

现象：loss 很怪、收敛慢、输出总喜欢生成 pad 相关 token。
原因：labels padding 没被替换成 -100。
解决：用 `DataCollatorForSeq2Seq`，或手动把 pad 替换为 -100。

### 坑 2：你在 map 里固定 padding 导致显存爆

现象：batch_size 一大就 OOM。
解决：map 不 padding，让 collator 动态 padding。

### 坑 3：predict_with_generate=True 时评估报 shape/类型问题

通常是因为 `labels` 里 -100 decode 前没替换回 pad id。
解决：compute_metrics 里 decode labels 前做：

```python
labels = np.where(labels == -100, tokenizer.pad_token_id, labels)
```

