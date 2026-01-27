[TOC]

------

## 一、BART 是什么

**BART（Bidirectional and Auto-Regressive Transformers）**
👉 **把 BERT 的“强理解能力” + GPT 的“强生成能力”合在一起的序列到序列模型**。

> 如果一句话不够：
> **BART = Encoder 用 BERT 思想，Decoder 用 GPT 思想的 Transformer**

------

## 二、BART 解决了什么问题？（为什么要有它）

在 BART 出来之前：

| 模型             | 强项       | 弱点           |
| ---------------- | ---------- | -------------- |
| **BERT**         | 理解、分类 | ❌ 不会生成     |
| **GPT**          | 生成       | ❌ 上下文理解弱 |
| **传统 Seq2Seq** | 翻译       | ❌ 预训练弱     |

👉 **BART 的目标**：

> **一个模型，同时擅长理解 + 生成**

典型任务包括：

- 文本摘要
- 翻译
- 文本生成
- 对话
- 文本纠错 / 补全
- 信息抽取 → 文本

------

## 三、整体结构（核心中的核心）

### 1️⃣ 架构总览

![Image](https://media.geeksforgeeks.org/wp-content/uploads/20230321032520/bart1drawio-%282%29.png)

![Image](https://miro.medium.com/1%2AgyN8A6xKTsZhROKUOcmAAw.png)

BART 是一个**标准的 Encoder–Decoder Transformer**：

```
输入文本 → Encoder（双向） → Decoder（自回归） → 输出文本
```

### 2️⃣ Encoder（像 BERT）

- **双向注意力**
- 能同时看到上下文
- 擅长“理解被破坏的句子”

👉 功能：

- 建模全局语义
- 对输入文本进行“语义压缩”

### 3️⃣ Decoder（像 GPT）

- **自回归生成**
- 每一步只能看前面生成的 token
- 使用 **cross-attention** 看 Encoder 输出

👉 功能：

- 按顺序生成新文本
- 保证输出自然、有逻辑

------

## 四、BART 最关键的思想：去噪自编码（Denoising AutoEncoder）

这是 **BART 的灵魂**。

### 1️⃣ 核心思想

> **先把一段正常文本“故意破坏”，再让模型学会恢复原文**

也就是说：

```
原始文本 → 噪声函数 → 破坏文本 → BART → 原始文本
```

### 2️⃣ 常见的“破坏方式”（非常重要）

| 噪声方式         | 举例           | 学到什么   |
| ---------------- | -------------- | ---------- |
| Token Mask       | 替换为         | 补全能力   |
| Token Delete     | 删除词         | 推理上下文 |
| Text Infilling   | 连续 span 替换 | 结构理解   |
| Sentence Shuffle | 句子打乱       | 逻辑顺序   |
| Document Rotate  | 文本旋转       | 全局建模   |

📌 **Text Infilling 是 BART 最标志性的噪声**

例如：

```
原文：我今天去医院看病，医生建议住院观察。
破坏：我今天 <mask> 医生建议 <mask>。
目标：恢复完整句子
```

------

## 五、训练过程（一步一步）

### 1️⃣ 输入

```text
破坏后的文本
```

### 2️⃣ Encoder

- 对破坏文本进行**双向建模**
- 输出一组隐向量表示

### 3️⃣ Decoder

- 从 `<bos>` 开始
- 一步一步生成 token
- 每一步：
  - 看自己之前生成的内容
  - 通过 cross-attention 看 Encoder 输出

### 4️⃣ 损失函数

- **标准 CrossEntropyLoss**
- 教师强制（Teacher Forcing）

------

## 六、为什么 BART 特别适合“生成 + 理解”的任务？

我们用一句“模型性格”来形容：

| 模块    | 性格             |
| ------- | ---------------- |
| Encoder | “读懂全文的学霸” |
| Decoder | “会写文章的作家” |

👉 **所以它特别强在：**

- 摘要（压缩 + 重写）
- 翻译（理解 + 表达）
- 对话（上下文 + 生成）
- 文本纠错（看懂错误 → 改写）

------

## 七、BART vs T5 vs GPT（你以后一定会纠结）

| 对比       | BART            | T5              | GPT          |
| ---------- | --------------- | --------------- | ------------ |
| 架构       | Encoder-Decoder | Encoder-Decoder | Decoder-only |
| 预训练目标 | 去噪重建        | Text-to-Text    | LM           |
| 理解能力   | ⭐⭐⭐⭐            | ⭐⭐⭐⭐            | ⭐⭐           |
| 生成能力   | ⭐⭐⭐⭐            | ⭐⭐⭐⭐            | ⭐⭐⭐⭐⭐        |
| 任务统一性 | 中              | ⭐⭐⭐⭐⭐           | 低           |
| 中文可用性 | ⭐⭐⭐             | ⭐⭐⭐             | ⭐⭐⭐⭐         |

**实用建议：**

- **摘要 / 翻译 / 改写 → BART**
- **多任务统一 → T5**
- **自由生成 / 对话 → GPT**

------

## 八、你如果“要用”，最重要的 3 个点

### ✅ 1. 用哪个类？

```python
from transformers import BartForConditionalGeneration, BartTokenizer
```

📌 **不是 BertModel，也不是 GPTModel**

------

### ✅ 2. 输入 & 输出结构

```text
input_ids  → Encoder
labels     → Decoder（自动右移）
```

你**不需要**手动 shift labels，HF 会帮你做。

------

### ✅ 3. 什么时候 BART 不合适？

- 纯分类任务 → **BERT 更简单**
- 超长文本（>1024）→ 需要 Long-BART / LED
- 超自由对话 → GPT 系列更自然

------

## 九、给你一句“真正理解 BART 的标志”

> **你能解释清楚：
> BART 为什么既不像 BERT 那样只会理解，
> 也不像 GPT 那样只会往前猜，
> 而是靠“破坏 → 重建”把两者融合起来。**