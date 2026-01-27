[TOC]

**训练时 labels 里确实有 padding**，而且一般会把 padding 位置设成 **-100** 来“忽略 loss”。
但**评估 ROUGE**这件事，关键区别在于：你是在评估**生成的完整摘要字符串**，还是评估**teacher-forcing 下的 token 预测**。

------

## 1)  ROUGE 评估（用 `generate()`）——不需要手动忽略 padding

如果评估脚本是：

- `pred = model.generate(...)` 得到生成序列
- `decode(pred)` 得到字符串
- 和 `decode(label)` 的参考摘要字符串算 ROUGE

那么：

✅ **不需要你处理“预测值 padding”**
因为 `generate()` 输出的是“已生成的序列”，不是固定长度的 `[B, T]` logits；它不会像训练 batch 那样为了对齐而 pad 一堆 token 给你（即使 batch 内部做了 padding，`decode(..., skip_special_tokens=True)` 也会把 `<pad>` 等特殊符号去掉）。

✅ **参考 summary 也不需要手动忽略 padding**
因为你的参考是来自 CSV 的真实 `summary` 字符串，本来就没有 padding。

> 所以：**用生成式评估（generate + ROUGE）不需要你自己忽略 padding**。

------

## 2) 什么时候才需要“忽略 padding”？

### A) 训练时算 loss（必须）

这是你说的那种情况：

- labels 为了 batch 对齐会 padding
- padding 位置必须设为 **-100**，CrossEntropyLoss 才会忽略

这一步通常由 `DataCollatorForSeq2Seq` 自动帮你做。

### B) 如果你用的是“teacher forcing 预测 + token 级评估”

比如你不用 generate，而是：

- forward 得到 `logits: [B, T, V]`
- `argmax` 得到 `pred_ids: [B, T]`
- 然后你想和 `labels: [B, T]` 做 token accuracy / token-F1

那你就必须忽略：

- labels == -100 的位置（或者 pad token 的位置）

否则你会把“padding token 预测对不对”也算进去，指标会失真。

------

## 3) 你看到的“predictions 也有 padding”，到底是哪种 predictions？

有两种常见情况：

### 情况 1：`Seq2SeqTrainer` 在评估时 `predict_with_generate=True`

- Trainer 会调用 `generate()`，生成序列
- 它为了把 batch 里的生成序列拼成一个矩阵，可能会 pad 到同长度
- 但你 decode 时用 `skip_special_tokens=True`，pad 会被跳过
  ✅ 无需额外处理

### 情况 2：你拿的是 `logits.argmax(-1)` 的 token 序列

- 这就是 teacher forcing 的“逐位置预测”
- 这里的序列长度固定 = labels 的长度（包含 padding）
  ✅ **必须根据 labels 的 mask 忽略 padding**

------

## 4) 你现在的 ROUGE 评估脚本该怎么写才“万无一失”？

如果你用的是 generate（我给你的 evaluate_sum.py 就是这种），你只要确保两点：

1. decode 时跳过特殊 token：

```python
tok.batch_decode(out_ids, skip_special_tokens=True)
```

1. 参考摘要直接用 CSV 的 `summary` 字符串（不要用 padded labels 去 decode）
   ✅ 这已经天然避免 padding 问题

------

## 5) 如果你确实想“用 labels decode 做参考”（比如从 dataset 里取 labels）

那你就要先把 `-100` 替换回 `pad_token_id` 再 decode（否则 decode 会报错或产生奇怪 token）：

```python
labels = np.where(labels != -100, labels, tokenizer.pad_token_id)
ref_texts = tokenizer.batch_decode(labels, skip_special_tokens=True)
```

这不是为了“ROUGE 忽略 padding”，而是为了**能正常 decode**。

------

### 最简结论

- ✅ **ROUGE（generate 输出字符串）**：不用手动忽略 padding
- ✅ **token 级评估（argmax logits vs labels）**：必须忽略 labels 的 padding（-100）