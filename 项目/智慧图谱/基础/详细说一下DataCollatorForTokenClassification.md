[TOC]

`DataCollatorForTokenClassification` 是 Hugging Face Transformers 里**专门给 NER/分词标注**这类“token-level 任务”用的 batch 组装器（collator）。

它干的事很聚焦：**把一个 batch 里不同长度的样本动态 padding 到同一长度，并且把 labels 也按同样方式 padding**，避免你手写一堆对齐代码。

------

## 1）它解决的核心问题

NER 数据在 `tokenizer` 后，每条样本长度不一样：

- `input_ids` 长度不同
- `attention_mask` 长度不同
- `labels`（token 标签序列）长度也不同

DataLoader 在组 batch 时必须把它们对齐成矩阵（`[batch, seq_len]`），否则会报错。

`DataCollatorForTokenClassification` 做的就是：

- 对 `input_ids / attention_mask / token_type_ids ...` 进行 padding（通常调用 `tokenizer.pad`）
- **对 `labels` 也进行 padding**，padding 用一个特定值（默认 `-100`）
  - `-100` 的意义：PyTorch 的 `CrossEntropyLoss(ignore_index=-100)` 会**忽略这些位置的损失**，不会影响训练

------

## 2）它和 `DataCollatorWithPadding` 有什么区别？

- `DataCollatorWithPadding`：只保证输入字段 padding（input_ids 等），**不管 labels**（或只能处理 classification/regression 那种单个 label）
- `DataCollatorForTokenClassification`：在 `WithPadding` 的基础上，**额外把 token-level labels 也 padding 并对齐**

所以做 NER / POS / chunking，基本就用它。

------

## 3）它的典型输入输出长什么样

### 输入（每条样本）

```python
{
  "input_ids": [101, 2259, 3268, 102],
  "attention_mask": [1, 1, 1, 1],
  "labels": [0, 5, 0, -100]
}
```

注意：labels 往往已经是 token 级（你通过 `word_ids()` 对齐过了），并且特殊 token 通常设为 `-100`。

### 输出（一个 batch）

collator 会返回类似：

- `input_ids`: shape `[B, max_len]`
- `attention_mask`: `[B, max_len]`
- `labels`: `[B, max_len]`（padding 位是 -100）

------

## 4）它有哪些关键参数（你最常会用到的）

在 Transformers 中常见构造方式：

```python
from transformers import DataCollatorForTokenClassification

data_collator = DataCollatorForTokenClassification(
    tokenizer=tokenizer,
    padding=True,          # 或 'longest' / 'max_length'
    max_length=None,       # padding 到固定长度时用
    pad_to_multiple_of=8,  # 可选：为了 Tensor Core/加速（比如 fp16）
    label_pad_token_id=-100,
    return_tensors="pt"
)
```

### 参数解释

- `tokenizer`：用它来做 padding（决定 pad_token_id、padding_side 等）
- `padding`：
  - `True` / `'longest'`：pad 到 batch 里最长
  - `'max_length'`：pad 到 `max_length`
  - `False`：不 pad（一般训练不行）
- `pad_to_multiple_of`：把长度 pad 到某个倍数（8/16），有时对 GPU 更友好
- `label_pad_token_id`：
  - 默认 `-100`，**强烈建议保留**
  - 除非你改了 loss 的 ignore_index，否则别动
- `return_tensors`：默认一般是 `"pt"`（PyTorch）

------

## 5）它“不会”帮你做什么（这点很重要）

很多人以为它能自动把词级标签扩展到 subword —— **它做不到**。

它只做 **padding & batch 打包**：

- ✅ 统一长度、补齐 labels
- ✅ 保证张量化
- ❌ 不负责你 NER 的 label alignment（word->subword 对齐）
- ❌ 不负责 BIO/BIOES 的合法性修正
- ❌ 不负责把 `labels` 变成 `label_ids`

所以你仍然需要在 `dataset.map(...)` 的时候自己做对齐逻辑（通常用 `is_split_into_words=True` + `word_ids()`）。

------

## 6）最常见的坑与排查

### 坑 1：你的 labels 长度和 input_ids 不一致

原因：对齐逻辑写错了或忘了给特殊 token 设 `-100`
表现：DataLoader/collator 报维度错误，或训练时报 shape mismatch。

### 坑 2：label_pad_token_id 改了但 loss 还是 ignore -100

表现：padding 的 label 被当成真实类别训练 → 指标很怪
建议：不要改；或者同步改 loss 的 ignore_index。

### 坑 3：你已经 `padding='max_length'` 在 tokenizer 里做了静态 padding

你又用 collator 动态 padding —— 通常没必要，会浪费长度
建议：训练集一般让 collator 动态 pad；评估/导出时再固定长度。

------

## 7）一个最小可用的训练搭配（常见范式）

```python
tokenized_ds = raw_ds.map(tokenize_and_align_labels, batched=True)

data_collator = DataCollatorForTokenClassification(tokenizer)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_ds["train"],
    eval_dataset=tokenized_ds["validation"],
    tokenizer=tokenizer,
    data_collator=data_collator,
)
```

