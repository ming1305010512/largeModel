[TOC]

NLTK（Natural Language Toolkit）是 Python 里非常经典的一套**自然语言处理教学/原型工具箱**。它的特点是：**覆盖面广、接口统一、配套语料/词典丰富、示例多**；但在工业级大规模场景（速度、深度学习、中文生态等）上通常会配合/替换成 spaCy、HuggingFace、jieba、Stanza 等。

下面把 NLTK 的核心内容按“你用它能做什么 → 常用模块 → 典型代码套路 → 经验坑点”讲清楚。

------

## 1）NLTK 能做什么（你会用到的能力清单）

**文本预处理**

- 分句（sentence split）、分词（tokenize）
- 规范化：小写化、去标点、停用词过滤、词形还原/词干提取

**基础语言学分析**

- 词性标注（POS tagging）
- 命名实体识别（NE chunk，偏传统方法）
- 短语分块（chunking）、句法分析（parsing）

**语料与词典**

- 内置/可下载多种语料（Brown、Reuters、Gutenberg…）
- WordNet（同义词、上下位词、词义消歧相关）

**统计 NLP**

- n-gram、频次统计、搭配（collocations）、TF/IDF（自己做也行）
- 朴素贝叶斯等传统分类器（text classification）
- 各种评估工具（准确率、混淆矩阵、标注一致性等）

一句话：NLTK 很像“**NLP 的瑞士军刀 + 教科书配套实验室**”。

------

## 2）安装与数据包（NLTK 最常见的第一道坎）

NLTK 的很多功能需要下载资源（分词模型、词性标注模型、停用词表、语料库等）。

```bash
pip install nltk
```

Python 里下载资源（会弹出 GUI 或走命令行）：

```python
import nltk
nltk.download('punkt')          # 分词/分句
nltk.download('averaged_perceptron_tagger')  # 词性标注
nltk.download('stopwords')      # 停用词
nltk.download('wordnet')        # WordNet
```

常见现象：

- 你代码没错，但报 `LookupError: Resource ... not found`：就是资源没下载或路径没配置好。
- 服务器/离线环境：需要把 nltk_data 目录打包拷到目标机，并设置 `NLTK_DATA` 环境变量或 `nltk.data.path.append(...)`。

------

## 3）最常用的模块与功能

### A. 分词与分句（tokenize）

```python
from nltk.tokenize import sent_tokenize, word_tokenize

text = "Hello world. NLTK is great!"
sents = sent_tokenize(text)
words = word_tokenize(text)
```

补充：

- 英文分句分词体验很好；中文需要额外方案（如 jieba），NLTK 不是强项。

### B. 停用词（stopwords）

```python
from nltk.corpus import stopwords

sw = set(stopwords.words('english'))
filtered = [w for w in words if w.lower() not in sw]
```

### C. 词干提取 vs 词形还原（stemming vs lemmatization）

**词干提取（更粗暴）**：把词砍到“词干”，速度快但可能不是真词。

```python
from nltk.stem import PorterStemmer
stemmer = PorterStemmer()
stemmer.stem("running")  # run
```

**词形还原（更像“词典形式”）**：更自然，但通常需要词性信息。

```python
from nltk.stem import WordNetLemmatizer
lemmatizer = WordNetLemmatizer()
lemmatizer.lemmatize("running", pos="v")  # run
lemmatizer.lemmatize("better", pos="a")   # good
```

### D. 词性标注（POS tagging）

```python
import nltk
tokens = word_tokenize("NLTK can tag parts of speech.")
tags = nltk.pos_tag(tokens)
# [('NLTK','NNP'), ('can','MD'), ('tag','VB'), ...]
```

说明：

- 词性标签通常是 Penn Treebank tagset（NN, VB, JJ…）。
- 你可以用 `nltk.help.upenn_tagset()` 查每个标签含义。

### E. 命名实体识别与短语分块（chunk）

```python
import nltk
tokens = word_tokenize("Barack Obama was born in Hawaii.")
tags = nltk.pos_tag(tokens)
tree = nltk.ne_chunk(tags)
```

注意：

- 这是传统方法路线，效果和现代深度学习 NER 不能比，但学习概念很方便。

### F. 频次统计（FreqDist）与搭配（collocations）

```python
from nltk import FreqDist
fd = FreqDist([w.lower() for w in words])
fd.most_common(10)
```

搭配（如 bigram）：

```python
from nltk.collocations import BigramCollocationFinder
from nltk.metrics import BigramAssocMeasures

finder = BigramCollocationFinder.from_words([w.lower() for w in words])
finder.nbest(BigramAssocMeasures.pmi, 10)
```

### G. WordNet（词义网络）

```python
from nltk.corpus import wordnet as wn

synsets = wn.synsets("bank")
synsets[0].definition()
synsets[0].examples()
```

你还可以查：

- 同义词（lemmas）
- 上位/下位词（hypernyms/hyponyms）
- 相似度（一些基于 WordNet 的传统相似度）

------

## 4）一个“典型 NLP 小流水线”长什么样

大概就是：

1. 文本清洗/规范化
2. 分句/分词
3. 去停用词/词形还原
4. 特征统计（n-gram、TF 等）
5. 训练分类器/做分析
6. 评估

NLTK 很适合把 1–6 全流程跑通做课程实验/原型验证。

------

## 5）NLTK 的定位与选型建议（你该什么时候用它）

**适合：**

- 学 NLP 基础（分词、词性、句法、语料、传统 ML）
- 做小规模实验、写教学 demo、验证想法

**不太适合：**

- 追求速度、生产级工程化
- 现代深度学习主导的 NLP（BERT/LLM 管线）
- 中文为主的工程（中文分词/词法资源不如专门中文工具链）

实用组合：

- 英文传统 NLP：NLTK + scikit-learn（向量化/分类）
- 工业英文 NLP：spaCy（速度、工程化）+ transformers（深度语义）
- 中文：jieba / pkuseg / HanLP / transformers（视需求）