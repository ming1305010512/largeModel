[TOC]

以 HuggingFace 里最常用的 **`BartForConditionalGeneration.forward()`** 为主线，把一次前向 **从输入到 loss / logits** 的完整链路讲清楚（含：shift、encoder/decoder、cross-attn、lm_head、cache）。

下面的“流程”可以当成读源码的导图。

------

## 0. 你在调用 forward 时一般传什么

常见两种模式：

### A) 训练（有 labels）

```python
out = model(
  input_ids=src_ids,
  attention_mask=src_mask,
  labels=tgt_ids
)
```

你不显式传 `decoder_input_ids`，只传 `labels`。

### B) 推理/生成（无 labels）

```python
out = model(
  input_ids=src_ids,
  attention_mask=src_mask,
  decoder_input_ids=dec_ids
)
```

------

## 1. forward 的“大框架”（一句话）

**BART forward = Encoder(src) → Decoder(tgt, cross-attn to encoder) → lm_head → (可选) loss**

------

## 2. 关键步骤 1：处理 decoder 输入（shift right）

### 2.1 如果你传了 `labels`

HF 会做一件事：**把 labels 右移一位**作为 decoder 的输入（teacher forcing）。

- `labels`: 目标序列（希望模型预测出来的 token）
- `decoder_input_ids`: 右移后的目标序列（喂给 decoder 的“已知前缀”）

右移规则（概念）：

```
decoder_input_ids = [BOS] + labels[:-1]
```

同时，`labels` 里如果有 `-100`（忽略位），loss 计算会跳过。

> 你要记住：
> **训练时 decoder 看到的是正确答案的前缀（teacher forcing），不是自己预测的前缀。**

### 2.2 如果你没传 labels，但传了 decoder_input_ids

那就直接用你给的 `decoder_input_ids`，不 shift。

------

## 3. 关键步骤 2：Encoder 前向

Encoder 输入：

- `input_ids`：形状 `[bs, src_len]`
- `attention_mask`：形状 `[bs, src_len]`（1 表示有效 token，0 表示 padding）

Encoder 做什么：

- token embedding + position embedding
- N 层 Transformer Encoder block（self-attn + FFN）
- 输出 `encoder_last_hidden_state`：形状 `[bs, src_len, d_model]`

HF 的返回通常是一个结构体/tuple：

- `encoder_outputs.last_hidden_state` 就是上面的张量

------

## 4. 关键步骤 3：Decoder 前向（含 cross-attention）

Decoder 输入：

- `decoder_input_ids`：`[bs, tgt_len]`
- `decoder_attention_mask`：`[bs, tgt_len]`（有时你不传，模型会自动根据 pad 推）
- `encoder_hidden_states`：来自 encoder 的 `[bs, src_len, d_model]`
- `encoder_attention_mask`：也就是 `attention_mask`

Decoder 每一层包含三块注意力：

### 4.1 Masked Self-Attention（自回归）

- 只能看见自己当前位置之前的 token（因果 mask）
- 保证生成时“不能偷看未来”

### 4.2 Cross-Attention（最关键）

- Query 来自 decoder 当前隐状态
- Key/Value 来自 encoder 输出
- 让 decoder 在生成时“对齐/检索”源文本信息（翻译、摘要都靠它）

### 4.3 FFN

- 两层前馈网络 + 激活 + dropout

Decoder 输出：

- `decoder_last_hidden_state`: `[bs, tgt_len, d_model]`
- （可选）`past_key_values`: 用于增量生成 cache
- （可选）attentions / hidden_states

------

## 5. 关键步骤 4：lm_head 得到 logits

BART ForConditionalGeneration 会把 decoder 输出投影到词表：

```
logits = lm_head(decoder_last_hidden_state)
```

`lm_head` 本质是一个**线性层**：

```
lm_head: Linear(d_model → vocab_size)
```

数学形式：

```
logits = decoder_hidden_state @ Wᵀ + b
```

- `W`: `[vocab_size, d_model]`
- `b`: `[vocab_size]`

👉 **把“语义空间”投影回“词表空间”**

形状：

- `logits`: `[bs, tgt_len, vocab_size]`

注意：BART 通常还会加一个 `final_logits_bias`（一个词表大小的 bias）：

```
logits = logits + final_logits_bias
```

------

## 6. 关键步骤 5：loss（只在你传 labels 时）

loss 计算方式：

- 把 `logits` 展平成 `[bs*tgt_len, vocab]`
- 把 `labels` 展平成 `[bs*tgt_len]`
- 用 CrossEntropyLoss(ignore_index=-100)

也就是：

```
loss = CE(logits.view(-1, V), labels.view(-1))
```

输出里会包含：

- `loss`（标量）
- `logits`
- 以及 encoder/decoder 的各种输出

------

## 7. cache / use_cache / past_key_values（生成时为什么快）

在生成时（特别是 `generate()` 内部），会设置 `use_cache=True`：

- decoder 每一步生成只新增 1 个 token
- 如果不 cache：每一步都要对整个历史序列做 attention，代价越来越大
- 有 cache：保存每一层的 K/V（past_key_values），下一步只算新 token 的 Q 和新 K/V

所以 forward 会支持：

- 输入 `past_key_values`
- 输出更新后的 `past_key_values`

这也是为什么生成时 `tgt_len` 每次增 1，但速度还能接受。

------

## 8. 一份“接近源码”的伪代码（你读源码会非常对得上）

```python
def forward(
  input_ids, attention_mask,
  decoder_input_ids=None, decoder_attention_mask=None,
  labels=None,
  encoder_outputs=None,
  past_key_values=None,
  use_cache=None,
  output_attentions=None,
  output_hidden_states=None,
  return_dict=True
):

  # 1) 准备 decoder_input_ids
  if labels is not None and decoder_input_ids is None:
      decoder_input_ids = shift_right(labels)  # [BOS] + labels[:-1]

  # 2) encoder（如果没给 encoder_outputs）
  if encoder_outputs is None:
      encoder_outputs = encoder(
        input_ids=input_ids,
        attention_mask=attention_mask,
        ...
      )
  enc = encoder_outputs.last_hidden_state  # [bs, src_len, d_model]

  # 3) decoder
  decoder_outputs = decoder(
    input_ids=decoder_input_ids,
    attention_mask=decoder_attention_mask,
    encoder_hidden_states=enc,
    encoder_attention_mask=attention_mask,
    past_key_values=past_key_values,
    use_cache=use_cache,
    ...
  )
  dec = decoder_outputs.last_hidden_state  # [bs, tgt_len, d_model]

  # 4) lm_head -> logits
  logits = lm_head(dec) + final_logits_bias  # [bs, tgt_len, vocab]

  # 5) loss（可选）
  loss = None
  if labels is not None:
      loss = CE(logits.view(-1, V), labels.view(-1), ignore_index=-100)

  # 6) 返回
  return Seq2SeqLMOutput(
    loss=loss,
    logits=logits,
    past_key_values=decoder_outputs.past_key_values,
    decoder_hidden_states=...,
    decoder_attentions=...,
    cross_attentions=...,
    encoder_last_hidden_state=enc,
    encoder_hidden_states=...,
    encoder_attentions=...
  )
```

------

## 9. 你最容易卡的 3 个点（我提前帮你点出来）

1. **labels vs decoder_input_ids**

- `labels` 是“要预测的目标”
- `decoder_input_ids` 是“喂给 decoder 的输入”
- 训练时会自动 shift：decoder 输入 = labels 右移

1. **cross-attention 的来源**

- decoder 的 cross-attn 的 K/V 来自 encoder 输出
- attention_mask 会影响 encoder 哪些 token 可被关注（padding 不该被关注）

1. **生成为什么要 past_key_values**

- 否则每一步都重算历史，复杂度爆炸
- cache 让生成从 “O(T²)” 更接近 “逐步增量”

