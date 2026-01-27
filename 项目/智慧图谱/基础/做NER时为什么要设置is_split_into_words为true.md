[TOC]

在做 NER（尤其是**token-level 标注**：每个“词/字”一个标签）时把 `is_split_into_words=True`，核心目的就一个：

**告诉 tokenizer：你给我的输入已经是“按词（或字）切好的列表”，不要再按空格/规则把它当成一整句去重新切词；并且要保留“每个 subword 属于原来哪个词”的映射（word_ids）。**

这样你才能把 **原始的词级标签** 正确对齐到 **tokenizer 切出来的 subword tokens** 上。

------

## 1）NER 的最大坑：标签和 subword 对不齐

NER 数据常见格式：

- tokens（词序列）：`["John", "lives", "in", "New", "York"]`
- labels（词级标签）：`["B-PER", "O", "O", "B-LOC", "I-LOC"]`

但 tokenizer（BERT 的 WordPiece / BPE）会把某些词切成多个 subword：

- `"York"` 可能变成 `["Yo", "##rk"]`
- `"Washington"` 可能变成 `["Wash", "##ing", "##ton"]`

这时 **一个词的一个标签** 要怎么分配给多个 subword？
你必须知道 “每个 subword 来自原始第几个词”，才能对齐。

------

## 2）`is_split_into_words=True` 帮你做了两件关键事

### A. 按“词列表”而不是“字符串”来分词

如果你传入的是 list：`["New", "York"]`

- `is_split_into_words=True`：tokenizer 认为这是**已经切好的词**，它会对每个词单独做 subword 切分
- 不开这个参数：tokenizer 可能把它当成“字符序列/句子”，在某些 tokenizer 上会出现不符合你预期的处理（尤其是你本来就有 tokens 列表时）

### B. 生成 `word_ids()` 映射，让你能做 label alignment

开启后你可以拿到：

- `word_ids = encoding.word_ids()`

它会返回一个列表，长度等于 tokenized 后的 token 数，每个位置表示该 token 属于原始第几个词；特殊符号位置是 `None`：

例子（示意）：

原始 tokens：`["New", "York"]`
tokenizer 输出 tokens：`[CLS], "New", "Yo", "##rk", [SEP]`
对应 `word_ids()`：`[None, 0, 1, 1, None]`

这样你就能把词级标签对齐到 token 级：

- `New`（word_id=0）→ label 用原标签
- `Yo`（word_id=1）→ label 用原标签（通常是 B-LOC）
- `##rk`（word_id=1）→ 要么设为 `I-LOC`，要么设为 `-100`（常见做法：只训练第一个 subword，其余忽略）

------

## 3）不开会怎样？最常见的后果

1. 你拿不到正确的 `word_ids()`（或根本拿不到），就无法稳定对齐标签
2. 标签长度和 input_ids 长度对不上，训练直接报错
3. 更隐蔽的：你“对齐了”，但对齐错了——模型学到的是噪声，指标很怪（尤其是实体边界）

------

## 4）NER 里对 subword 标签的两种主流策略

有了 `word_ids()` 后，你一般会选其一：

### 策略 1：只给每个词的**第一个 subword**赋标签，其余设 `-100`

优点：最稳、最常见（Hugging Face 官方教程也常用这种思路）
效果：模型只在词的首 subword 学习标签

### 策略 2：把同一个词的标签“扩展”到所有 subword

比如 `B-LOC` 扩展到后续 subword 用 `I-LOC`
优点：token 级监督更密
缺点：要处理 BIO 转换细节，否则容易引入错误标注

------

## 5）一句话总结

`is_split_into_words=True` 的本质是：**NER 标注通常在“词级”，而 tokenizer 会变成“子词级”，这个参数让你获得词↔子词的映射，从而把 labels 正确对齐到模型输入。**

