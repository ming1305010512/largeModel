[TOC]

ROUGE 是做**文本生成任务**（尤其摘要）的经典评估指标，核心思想很朴素：

> **看模型生成的文本（candidate）和参考答案（reference）有多少“重合”。**

它不是看“意思像不像”，而是看“词/片段/顺序”重叠程度，所以你用它时要知道它擅长什么、不擅长什么。

------

## 1) ROUGE 家族在评什么

最常用三类：

### ROUGE-1

- **按 1-gram（单词/字）**重叠
- 更像“关键词覆盖度”
- 对内容覆盖敏感，但不太管顺序

### ROUGE-2

- **按 2-gram（相邻两词/两字）**重叠
- 更像“短语一致性”
- 更严格，通常分数比 ROUGE-1 低

### ROUGE-L

- 基于 **LCS（Longest Common Subsequence，最长公共子序列）**
- 看“保持相对顺序”的最长匹配
- 兼顾一定的顺序信息，但不要求连续

> 直觉：
>
> - 1：有没有提到关键内容
> - 2：短语像不像
> - L：整体顺序/结构像不像

------

## 2) ROUGE 的 Precision / Recall / F1 分别是什么意思

ROUGE 不是只有一个数，它通常有三种：

- **Recall（召回）**：参考答案里有多少被你覆盖到了
  $$
  R = \frac{\text{overlap}}{\text{# units in reference}}
  $$
  
- **Precision（精确）**：你生成的内容里有多少是参考答案也有的
  
  P = \frac{\text{overlap}}{\text{# units in candidate}}
  
- **F1**：综合平衡（最常汇报）
  $$
  F1 = \frac{2PR}{P+R}
  $$

✅ 在摘要里经常会看到“ROUGE-1/2/L 的 F1”。

**举个直觉例子**：

- 你生成很长：Recall 可能高（覆盖多），Precision 可能低（废话多）
- 你生成很短：Precision 可能高（句句命中），Recall 可能低（漏了很多）

------

## 3) ROUGE-1 / ROUGE-2 怎么算（n-gram 版）

### 定义

把文本切成 n-gram（n=1 或 2），然后算重叠数量（按多重集合计数）。

比如（用“词”举例）：

reference: `the cat sat on the mat`
candidate: `the cat sat`

- reference 的 1-gram 计数：the×2, cat×1, sat×1, on×1, mat×1
- candidate 的 1-gram 计数：the×1, cat×1, sat×1

重叠（取 min 计数）：

- the: min(2,1)=1
- cat: 1
- sat: 1
  overlap=3

然后：

- R = 3 / 6
- P = 3 / 3
- F1 = 2PR/(P+R)

ROUGE-2 同理，只是把单位换成 2-gram（相邻词对）。

------

## 4) ROUGE-L 怎么算（LCS 版）

LCS（最长公共子序列）允许“跳着匹配”，但必须保持相对顺序。

reference: `A B C D E`
candidate: `A C E`

LCS = `A C E`，长度 = 3

ROUGE-L 的常见形式用 LCS 长度来算 P/R/F1：

- ($$ R \approx \frac{LCS}{|reference|} $$)
- ($$ P \approx \frac{LCS}{|candidate|} $$)

（不同实现会有些细节差异，比如是否对句子级 LCS 做合并、是否加权。）

------

## 5) 多参考答案怎么处理

如果一个样本有多个 reference（常见于摘要数据集）：

- 对每个 reference 分别算 ROUGE
- 再取 **最大值** 或 **平均值**（取决于实现/论文习惯）
  - 很多实现会对每个样本取 best，再对全体平均

------

## 6) 中文任务里 ROUGE 的“坑”和怎么做更靠谱

中文没有空格分词，ROUGE 的单位会直接影响分数：

### 方案 A：按“字”做 ROUGE（最稳定）

- 直接把句子当字符序列，1-gram=单字，2-gram=双字
- 优点：不依赖分词
- 缺点：可能高估“字面重叠”

### 方案 B：先分词再算（更贴近语义，但更不稳定）

- 依赖分词器质量
- 不同分词器会导致 ROUGE 不可比

📌 实务建议：

- 如果你要**对比自己模型的迭代**：保持同一套分词/预处理即可
- 如果你要**跟论文/榜单对齐**：用对方同样的 ROUGE 设定（字级/词级、是否去标点、是否lower等）

------

## 7) ROUGE 的局限（一定要知道）

ROUGE 只看重叠，所以：

- ✅ 擅长：摘要这种“信息压缩 + 复述”任务
- ❌ 不擅长：同义改写、表达方式不同但意思相同（重叠低但其实很好）
- ❌ 容易被“堆关键词”骗高 Recall
- ❌ 不能直接衡量事实性（hallucination 可能 ROUGE 还不低）

所以很多工作会 ROUGE + 人评/事实性指标/任务指标一起看。

------

## 8) 你在论文/训练日志里常见的写法

通常会报：

- `ROUGE-1 F1`
- `ROUGE-2 F1`
- `ROUGE-L F1`

有时也会报 Recall（尤其摘要领域更关注覆盖）。

