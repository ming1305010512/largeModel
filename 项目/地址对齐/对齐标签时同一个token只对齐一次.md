[TOC]

代码1：

```
def tokenize_and_align_labels(examples, tokenizer, label2id, max_len: int):
    """
    examples:
      {"text": [list[str]], "labels": [list[str]]}  (batched)
    """
    tokenized = tokenizer(
        examples["text"],
        is_split_into_words=True,
        truncation=True,
        max_length=max_len,
    )

    aligned_labels = []
    # For each sample in batch
    for i in range(len(examples["text"])):
        word_ids = tokenized.word_ids(batch_index=i)
        word_labels = examples["labels"][i]

        label_ids = []
        prev_word_id = None
        for word_id in word_ids:
            if word_id is None:
                label_ids.append(-100)
            else:
                if word_id != prev_word_id:
                    lab = word_labels[word_id]
                    label_ids.append(label2id.get(lab, label2id["O"]))
                else:
                    label_ids.append(-100)
            prev_word_id = word_id

        aligned_labels.append(label_ids)

    tokenized["labels"] = aligned_labels
    return tokenized
```

代码2：

```
    def tokenize_and_align_labels(self,batch,tokenizer,label2id):
        tokenized = tokenizer(
            batch["text"],
            truncation=True,
            max_length=self.datasetConfig.max_len,
            is_split_into_words=True,
        )
        # 分词之后，input_ids长度和目标标签不一致，需对齐
        # 对齐后的标签
        aligned_labels = []
        for i in range(len(batch["text"])):
            word_ids = tokenized.word_ids(batch_index = i)
            word_labels = batch["labels"][i]

            label_ids = []
            prev_word_id = None
            for word_id in word_ids:
                if word_id is None:
                    label_ids.append(-100)
                else:
                    lab = word_labels[word_id]
                    label_ids.append(label2id.get(lab, -100))
```





**从“事实标注”的角度**，同一个字拆成多个 subtoken，确实都属于同一个实体片段，看起来“都打同一个标签”很自然。

但训练里“不这么做”主要是**优化目标和权重分配**的问题，不是“事实对不对”的问题。核心有 3 点：

------

## 1) 重复监督 = 给某些字“加权了”

如果一个字被拆成 1 个 token，它只贡献 **1 次** loss；
如果它被拆成 3 个 subtokens，你给每个 subtoken 都打标签，它就贡献 **3 次** loss。

这等价于：**同一个字因为 tokenizer 切分方式不同，被自动加了权重**。

- 哪些更容易被拆？
  常见是：英文/拼音、罕见字、混合串、特殊符号、部分数字写法……
- 结果：模型会被迫“更在意”这些被拆开的片段，训练目标的权重分布被扭曲。

在地址里，手机号/门牌号/字母楼栋这类“被拆开概率更高”的字段，容易被**过度放大**，反而影响整体实体边界学习。

------

## 2) B/E 这种“边界标签”在 subtoken 上会变得不一致

你现在标签体系是 **B/I/E**（不是纯 IOB），边界语义很强：

- `B-phone` 表示“实体开始”
- `E-phone` 表示“实体结束”

如果一个字（或一个“word”）拆成多个 subtokens，你把 **同一个标签复制给所有 subtokens**，会出现很奇怪的监督：

- 第一个 subtoken 被标成 `B-xxx`（还算说得过去）
- 第二个 subtoken也被标成 `B-xxx`（这在序列标注语义里意味着“又开始一个实体”）
- 或者都标成 `E-xxx`（意味着“每个 subtoken 都是结尾”）

这会让模型在学习“实体边界模式”时很别扭，尤其你后面如果用 CRF/转移约束，这种监督会变得更冲突。

所以主流做法要么：

- **只在第一个 subtoken监督**（其他忽略），最简单稳；
- 或者 **后续 subtoken 改成 I-xxx**（把边界标签只放在边界上）。

------

## 3) 训练不稳定的本质：梯度方差变大 + 类别不平衡更严重

重复监督会带来两个副作用：

### (a) 梯度方差更大

同一个样本里某些片段突然贡献多倍 loss，batch 内 loss 波动变大，训练曲线更抖。

### (b) 类别不平衡更严重

地址 NER 本来 `O` 就多、实体标签少。
如果“容易被拆的实体”（如 phone/no）被重复算 loss，它们的相对权重变高，模型更倾向把其它字段往这些“强权重实体”上吸，引发混淆。

------

## 那为什么“只标首 subtoken”就更合理？

因为你的标注其实是对 **原始字符序列**（字级）定义的，而 tokenizer 的 subtoken 只是模型内部表示方式。
我们要避免“表示方式”影响训练目标权重，所以采用一个简单的约定：

> **每个原始字/词只贡献一次监督信号**（通常放在首 subtoken 上），这样训练目标与 tokenizer 切分无关，更稳定、更公平。

------

## 什么时候可以/应该给所有 subtokens打标签？

可以，但要满足两个条件：

1. 你的标签体系最好是 **BIO（或 BIES）** 并且你愿意做转换：
   - 首 subtoken：B-xxx / I-xxx / E-xxx
   - 非首 subtoken：统一变成 I-xxx（不要复制 B/E）
2. 你接受“被拆分的字会更重”的事实，或者你再做 re-weight 来抵消。

在你的地址任务上，**收益通常不明显**，而风险更高，所以我才建议用“首 subtoken监督”。

------

## 给你一句实操结论

- **用 B/I/E（你现在这个）**：强烈建议「首 subtoken监督，其余 -100」✅
- 如果你未来换成 **BIO/BIES + CRF**，且你非常想所有 subtokens监督：那就做“后续 subtoken → I-xxx”✅

如果你愿意，我可以根据你现在的标签集合（B-name/E-name…）给你写一个“把 BIE 自动转换成 BIES 或 BIO”的函数，后续你加 CRF 也更顺手。