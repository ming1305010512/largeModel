[TOC]

下面我**从“它解决什么问题 → 原理 → 参数 → 工作流程 → 例子 → 何时用/不用 → 和其它 splitter 对比 → 常见坑”**，把 **`SemanticChunker`** 一次讲透。
（这是 LangChain 里**语义级切分**的代表，和 Character/Recursive 完全不是一个思路）

------

# 一、SemanticChunker 是干什么的？

一句话：

> **SemanticChunker = 按“语义变化”切文本，而不是按字符/句号/长度**

也就是说：

- **不是**：每 500 字切一块
- **而是**：
  👉 当“话题明显发生变化”时才切一块

它的目标是：
**让每一个 chunk 在“语义上尽量自洽、完整”**

这对 **RAG（检索增强生成）** 非常重要。

------

# 二、为什么普通 TextSplitter 不够？

### 传统切分的问题（Character / Recursive）

```text
……患者既往有高血压病史10余年，平素服药不规律。
今晨突发头晕、恶心、呕吐……

【这里如果按长度切】
↑ 上一句在 chunk1
↓ 下一句在 chunk2
```

结果：

- 一个 chunk 只有“背景”
- 一个 chunk 只有“症状”
- **语义被人为撕裂**

------

### SemanticChunker 的思路

它会问一个“隐含问题”：

> **“这一句，和上一句，语义上还连贯吗？”**

- 连贯 → 放同一个 chunk
- 不连贯 → **在这里切**

------

# 三、SemanticChunker 的核心原理（非常重要）

### 核心三步：

## 1️⃣ 先把文本切成“很小的语义单位”

通常是：

- 句子（按 `.` / `。` / `\n`）
- 或短段落

> ⚠️ 注意：SemanticChunker **不是直接对整段算 embedding**，
> 而是先拆成很多“小片段”。

------

## 2️⃣ 给每个小片段算 embedding（向量）

例如：

```text
S1: “患者男性，65岁。”
S2: “既往有高血压病史10年。”
S3: “平素服药不规律。”
S4: “本次因突发头晕入院。”
S5: “CT提示脑梗死。”
```

每一句 → 一个向量

------

## 3️⃣ 比较“相邻句子”的语义距离

- 用 **余弦相似度**
- 如果：
  - 相似度 **高** → 同一话题 → 合并
  - 相似度 **低** → 话题变了 → **切块**

📌 **切分点 = 语义断层点**

------

# 四、SemanticChunker 常用参数（逐个讲）

### 1️⃣ `embeddings`（必传）

```python
SemanticChunker(embeddings=embeddings)
```

- 必须提供 embedding 模型
- 没 embedding，就无法判断语义距离

------

### 2️⃣ `breakpoint_threshold_type`

决定：**“多不像，才算该切？”**

常见值：

| 类型                   | 含义                             |
| ---------------------- | -------------------------------- |
| `"percentile"`         | 按相似度分布的百分位切（最常用） |
| `"standard_deviation"` | 超过均值 ± nσ 就切               |
| `"absolute"`           | 固定阈值（不常用）               |

📌 **推荐：`percentile`**

------

### 3️⃣ `breakpoint_threshold_amount`

和上面一起用，表示“多激进”。

```python
breakpoint_threshold_amount=95
```

含义：

- 相邻句子相似度
- **低于全局相似度分布的第 95 百分位**
- 就切

经验理解：

- 数值 **越大** → **越容易切（chunk 越小）**
- 数值 **越小** → chunk 越大

------

### 4️⃣ `min_chunk_size`

```python
min_chunk_size=100
```

- 防止切得太碎
- 就算语义断了，也要保证最小长度

------

# 五、完整示例（标准用法）

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

splitter = SemanticChunker(
    embeddings=embeddings,
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95,
)

chunks = splitter.split_text(long_text)

for i, c in enumerate(chunks):
    print(f"Chunk {i+1}:")
    print(c)
    print("------")
```

------

# 六、SemanticChunker 的“直觉示例”

### 原文

```text
患者男性，65岁。
既往有高血压病史10年，平素服药不规律。
本次因突发头晕、呕吐入院。
头颅CT提示脑梗死。
给予抗血小板、调脂治疗。

患者家属表示经济困难。
要求保守治疗。
```

### SemanticChunker 可能切成：

**Chunk 1（病情）**

```text
患者男性，65岁。
既往有高血压病史10年，平素服药不规律。
本次因突发头晕、呕吐入院。
头颅CT提示脑梗死。
给予抗血小板、调脂治疗。
```

**Chunk 2（沟通/社会因素）**

```text
患者家属表示经济困难。
要求保守治疗。
```

👉 这正是 **RAG 最想要的结构**

------

# 七、和其他 TextSplitter 的对比（重点）

| splitter                       | 切分依据       | 是否看语义 | 是否需要 embedding |
| ------------------------------ | -------------- | ---------- | ------------------ |
| CharacterTextSplitter          | 字符           | ❌          | ❌                  |
| RecursiveCharacterTextSplitter | 字符 + 递归    | ❌          | ❌                  |
| TokenTextSplitter              | token          | ❌          | ❌                  |
| **SemanticChunker**            | **语义相似度** | ✅          | ✅                  |

------

# 八、什么时候一定要用 SemanticChunker？

✅ **强烈推荐**

- RAG 问答系统
- 医疗/法律/指南类长文本
- 结构不规则、但语义清晰的文章
- “一段就是一个意思”的文档

❌ **不建议**

- 代码（语义 embedding 对代码不友好）
- 严格结构文档（JSON / 表格）
- 超大规模批量切分（embedding 成本高）

------

# 九、常见坑（非常重要）

### ❌ 1. embedding 模型不好 → 切得一塌糊涂

- embedding 不稳定
- 中文支持差
  👉 切点会很怪

------

### ❌ 2. 文本本来就很短

SemanticChunker **不会比 Recursive 更聪明**
短文本用它是浪费。

------

### ❌ 3. breakpoint_threshold_amount 设太高

- chunk 过碎
- 检索噪声变大

------

# 十、一句话总结

> **SemanticChunker = 用 embedding 判断“什么时候该换话题”，而不是“到多少字该切”。**