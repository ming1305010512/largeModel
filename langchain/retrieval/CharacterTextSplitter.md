[TOC]

`CharacterTextSplitter` 是 LangChain 里最基础的“按字符分割”切块器：**先按某个分隔符把文本切成小段，再把小段“合并”成接近 `chunk_size` 的块，并按 `chunk_overlap` 做滑窗重叠**。它的核心价值就是：把长文切成适合检索/向量化/放进模型上下文的 chunks。 ([LangChain](https://python.langchain.com/docs/concepts/text_splitters/?utm_source=chatgpt.com))

------

## 它到底怎么切？（算法直觉）

可以把它理解成两步：

1. **Split（切片）**：用 `separator`（默认是 `"\n\n"`）把原文切成一段段 pieces。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/character_text_splitter?utm_source=chatgpt.com))
2. **Merge（组块）**：从前往后把 pieces 往一个 buffer 里塞，直到快到 `chunk_size`，就形成一个 chunk；然后往后滑动，按 `chunk_overlap` 保留一部分内容做重叠，再继续组下一个 chunk。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/character_text_splitter?utm_source=chatgpt.com))

> ⚠️ 注意：**它不是“强行每块都刚好 chunk_size”**。如果你的分隔符切出来的 pieces 本身很长（比如一个超长段落里没有 `\n\n`），那它可能产生 **大于 chunk_size 的 chunk**。这是 CharacterTextSplitter 常见“看起来不按 chunk_size 切”的原因。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/character_text_splitter?utm_source=chatgpt.com))

------

## 常用参数逐个讲清楚

### 1）`separator`

- **按什么字符/字符串切**（默认 `"\n\n"`，也就是按段落切）。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/character_text_splitter?utm_source=chatgpt.com))
- 你可以改成 `"\n"`（按行）、`" "`（按空格）等。

> 如果你希望“更稳地保证 chunk_size”，通常会用 `RecursiveCharacterTextSplitter`，它会尝试多级分隔符递归切，尽量让每块不超。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/recursive_text_splitter?utm_source=chatgpt.com))

------

### 2）`chunk_size`

- **chunk 的“最大目标长度”**，长度怎么计算由 `length_function` 决定；默认就是字符数（`len`）。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/character_text_splitter?utm_source=chatgpt.com))
- 重点是：它是“目标上限”，但不保证严格不超过（原因见上面 ⚠️）。

------

### 3）`chunk_overlap`

- 相邻 chunk 之间 **重叠多少长度**（同样用 `length_function` 计量）。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/recursive_text_splitter?utm_source=chatgpt.com))
- 作用：让跨 chunk 的信息不至于被切断（RAG 常用）。 ([LangChain](https://python.langchain.com/docs/concepts/text_splitters/?utm_source=chatgpt.com))

------

### 4）`length_function`

- 决定“长度单位”的函数：默认是 `len`（字符数）。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/recursive_text_splitter?utm_source=chatgpt.com))
- 你也可以用 tokenizer 的 token 数来当长度单位（但那通常更建议用 token splitter 或者递归 splitter）。

------

### 5）`is_separator_regex`

- 如果为 True，就把 separator 当正则（适合更复杂分隔模式）。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/recursive_text_splitter?utm_source=chatgpt.com))

### 6）`keep_separator`

- 有些版本支持：保留分隔符在 chunk 的开头或结尾，避免语义断裂（具体行为取决于你安装的 langchain-text-splitters 版本；不同版本字段名/默认值可能略有差异）。 ([LangChain](https://python.langchain.com/api_reference/text_splitters/character/langchain_text_splitters.character.CharacterTextSplitter.html?__hsfp=3087218293&__hssc=83405321.1.1749686400160&__hstc=83405321.73bd3bee6fa385653ecd7c9674ba06f0.1749686400157.1749686400158.1749686400159.1&utm_source=chatgpt.com))

------

## 最常见的使用方式（Python）

（不同版本 import 路径会略不同，下面是当前文档体系常见用法）

```python
from langchain_text_splitters import CharacterTextSplitter

splitter = CharacterTextSplitter(
    separator="\n\n",
    chunk_size=500,
    chunk_overlap=50,
    length_function=len,
)

chunks = splitter.split_text(long_text)
```

文档切分（会保留元数据、来源等）通常用：

- `split_documents(docs)` 或 `create_documents(texts, metadatas=...)`（看你版本提供哪个）。 ([LangChain Docs](https://reference.langchain.com/python/langchain_text_splitters/?utm_source=chatgpt.com))

------

## CharacterTextSplitter vs RecursiveCharacterTextSplitter（你该怎么选）

- **CharacterTextSplitter**：只按一个 separator 切，然后合并；简单、可控，但遇到“长段落无 separator”容易超块。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/character_text_splitter?utm_source=chatgpt.com))
- **RecursiveCharacterTextSplitter**：有 separators 列表（如 `["\n\n", "\n", " ", ""]`），会递归尝试更细粒度分割，**更接近你想要的 chunk_size**。 ([LangChain 文档](https://docs.langchain.com/oss/python/integrations/splitters/recursive_text_splitter?utm_source=chatgpt.com))

如果你做 RAG，大多数情况下直接上 `RecursiveCharacterTextSplitter` 更省心。

------

## 做 RAG 时的“经验规则”

- 你想按“段落”保语义：separator 用 `\n\n`
- 文本结构很乱/段落很长：用 recursive
- chunk_overlap 通常设成 chunk_size 的 10%~20%（不是硬规则，主要看你检索召回和上下文窗口）

