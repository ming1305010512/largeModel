[TOC]

`tokenized.word_ids()` 是 **Fast tokenizer**（通常是 `AutoTokenizer(..., use_fast=True)`）里用来做“**子词(token) ↔ 原始单词(word)** 对齐”的方法。

它返回：**每个 token 属于原始输入的第几个 word**（按空格/分词后的“词”编号），常用于 NER / 词级标注任务把 word-level label 对齐到 subword token。

------

## 1）它返回什么？

- 返回一个长度 = `input_ids` 长度的列表
- 每个位置是：
  - `None`：特殊符号或 padding（如 `[CLS] [SEP] [PAD]`）
  - `0,1,2,...`：表示该 token 来自原始输入的第 `i` 个 word

> 这个 word 的定义不是“中文一个字”，而是 **你传入 tokenizer 的“词单位”**：

- 如果你传的是 `is_split_into_words=True` 的 list（例如 `["我","喜欢","北京"]`），那 word 就是这些元素。
- 如果你传的是普通字符串 `"I love NLP"`，word 通常按空格分（英文场景最常见）。

------

## 2）典型例子（英文）

```python
from transformers import AutoTokenizer
tok = AutoTokenizer.from_pretrained("bert-base-uncased", use_fast=True)

enc = tok("New York", return_offsets_mapping=True)
print(tok.convert_ids_to_tokens(enc["input_ids"]))
print(enc.word_ids())
```

可能输出类似：

- tokens: `['[CLS]', 'new', 'york', '[SEP]']`
- word_ids: `[None, 0, 1, None]`

如果是 `"playing"` 这种会被拆成子词：

- tokens: `['[CLS]', 'play', '##ing', '[SEP]']`
- word_ids: `[None, 0, 0, None]`
  说明 `play` 和 `##ing` 都属于第 0 个 word。

------

## 3）中文场景要注意

中文通常不会按空格分词，所以：

### A. 你直接传字符串（不 split）

```python
enc = tok("我喜欢北京")
enc.word_ids()
```

这时 **word_ids 的意义往往不如英文直观**（因为“word”并不是你想象的词级单位）。

### B. 你先把词切好（推荐用于 NER/词级任务）

```python
words = ["我", "喜欢", "北京"]
enc = tok(words, is_split_into_words=True)
enc.word_ids()
```

这时 `0/1/2` 就严格对应你给的 `words` 下标，最适合做 label 对齐。

------

## 4）它常用来做什么？（NER/序列标注对齐）

比如你有 word-level 标签 `labels = [B-PER, O, B-LOC]`，tokenizer 可能把某个 word 拆成多个 subword。你需要把标签扩展到每个 subword：

常用规则：

- 同一个 word 的 **第一个 token** 用原标签
- 后续 subword 用 `-100`（忽略）或把 `B-` 改成 `I-`

核心就靠 `word_ids()` 判断“这个 token 属于哪个 word”。

------

## 5）补充：为什么有时没有 `.word_ids()`？

- 只有 **Fast tokenizer** 才有（Rust 实现）
- Slow tokenizer（Python 实现）可能没有，或行为不同
  所以一般要 `use_fast=True`。

