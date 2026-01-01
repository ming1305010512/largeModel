[TOC]

`torch.nn.utils.rnn.pad_sequence` 是 PyTorch 里专门给 **RNN/变长序列**做“**对齐成同一长度并堆叠成 batch**”的工具：把一组长度不同的序列（list of Tensor）补齐到同一最长长度 `T`，再按 batch 维堆起来。([PyTorch 文档](https://docs.pytorch.org/docs/stable/generated/torch.nn.utils.rnn.pad_sequence.html?utm_source=chatgpt.com))

------

## 1）函数签名与参数（你真正需要记住的点）

官方文档的核心参数是：([PyTorch 文档](https://docs.pytorch.org/docs/stable/generated/torch.nn.utils.rnn.pad_sequence.html?utm_source=chatgpt.com))

- **`sequences`**：`list[Tensor]`
  变长序列列表。每个张量形状一般是：
  - 纯 token id：`(L_i,)`
  - 每步有特征/embedding：`(L_i, D)` 或 `(L_i, *)`
    要求：**除了第一维长度 `L_i` 可变外，后面的维度必须一致**（比如都 `(?, 768)`）。
- **`batch_first=False`**：输出维度顺序
  - `False`（默认）：输出形状 **`(T, B, \*)`**（时间在前，RNN 传统格式）
  - `True`：输出形状 **`(B, T, \*)`**（更符合很多人习惯）
- **`padding_value=0.0`**：补齐用的值（默认 0）([PyTorch 文档](https://docs.pytorch.org/docs/stable/generated/torch.nn.utils.rnn.pad_sequence.html?utm_source=chatgpt.com))
  - 文档写的是 float，但实际会按输入 tensor dtype/device 生成 padded tensor；你用整型 token 时通常就给 `0`（PAD id）就行。
- **`padding_side='right'`**：从哪边补（默认右侧）([PyTorch 文档](https://docs.pytorch.org/docs/stable/generated/torch.nn.utils.rnn.pad_sequence.html?utm_source=chatgpt.com))
  - `'right'`：在序列末尾补齐（最常见，RNN/pack 更友好）
  - `'left'`：在序列开头补齐（Transformer 有时用，但 RNN 里要小心，下面会讲坑）

------

## 2）它的输入输出到底长什么样（用一个小例子讲透）

假设你有 3 条序列（长度分别 2、4、3），每步是一个 token id：

```python
import torch
from torch.nn.utils.rnn import pad_sequence

seqs = [
    torch.tensor([5, 6]),         # L=2
    torch.tensor([1, 2, 3, 4]),    # L=4 (max)
    torch.tensor([7, 8, 9])        # L=3
]

padded = pad_sequence(seqs, batch_first=True, padding_value=0)  # (B,T)
print(padded.shape)  # torch.Size([3, 4])
print(padded)
```

输出（右补齐）会类似：

- 第 1 条变成 `[5, 6, 0, 0]`
- 第 2 条 `[1, 2, 3, 4]`
- 第 3 条 `[7, 8, 9, 0]`

如果你的每个时间步不是一个标量，而是一个向量（例如 embedding，形状 `(L_i, D)`），输出就是 `(B, T, D)` 或 `(T, B, D)`。

------

## 3）最常见用法：配合 DataLoader 的 `collate_fn`

你在 `DataLoader` 里遇到的经典报错：

> ```
> stack expects each tensor to be equal size ...
> ```

本质就是默认 `collate_fn` 在做 `torch.stack`，而变长序列 stack 不了。正确姿势是：在 `collate_fn` 里用 `pad_sequence`。

```python
from torch.nn.utils.rnn import pad_sequence

PAD_ID = 0

def collate_fn(batch):
    # batch: list of (seq_tensor, label)
    seqs, labels = zip(*batch)

    lengths = torch.tensor([len(s) for s in seqs], dtype=torch.long)
    seqs_padded = pad_sequence(seqs, batch_first=True, padding_value=PAD_ID)  # (B,T)

    labels = torch.tensor(labels, dtype=torch.long)
    return seqs_padded, lengths, labels
```

这里 **`lengths` 非常关键**：后面你要么做 mask，要么做 pack。

------

## 4）配 RNN/LSTM 的两条路线：mask vs pack

### 路线 A：直接喂 padded + 自己做 mask（简单但可能慢）

- 你把 `(B,T,*)` 喂进 RNN
- padding 部分也会参与计算（无意义），你需要在 loss 或 pooling 时用 mask 去掉

### 路线 B：`pack_padded_sequence`（更“正统”，速度更好）

PyTorch 提供 `pack_padded_sequence`/`pad_packed_sequence` 专门让 RNN 跳过 padding 的计算。([PyTorch 文档](https://docs.pytorch.org/docs/stable/generated/torch.nn.utils.rnn.pack_padded_sequence.html?utm_source=chatgpt.com))

典型流程：

1. `pad_sequence` 得到 padded
2. 用 `lengths` pack
3. 丢给 LSTM/GRU
4. 需要时再 unpack

> 注意：pack 常见假设是 **右侧 padding**；也就是说“有效内容在前、padding 在后”最自然。

------

## 5）`padding_side='left'` 的坑（尤其你要 pack 的时候）

`pad_sequence` 现在支持 `'left'`（左补齐）。([PyTorch 文档](https://docs.pytorch.org/docs/stable/generated/torch.nn.utils.rnn.pad_sequence.html?utm_source=chatgpt.com))
但对 **RNN + pack** 来说，左补齐往往不划算：因为 pack/lengths 的语义通常是“前 `length` 个时间步是有效的”，左补齐会让有效内容在后面，容易导致你把 padding 当有效内容、把真实 token 截掉（除非你额外做移位/重排）。

经验：

- **RNN/LSTM/GRU：默认用右补齐**（`padding_side='right'`）
- **Transformer（某些实现/推理时）：可能用左补齐**（但那是另一个体系）

------

## 6）你会踩的 5 个高频错误（快速排雷）

1. **sequences 放的不是 Tensor（是 list[int]）**
   → 先 `torch.tensor(...)` 再 pad
2. **每条序列后面的维度不一致**
   例如有的 `(L, 768)` 有的 `(L, 1024)`
   → pad_sequence 没法救，得先统一特征维
3. **device 不一致（有的在 CPU，有的在 GPU）**
   → 统一搬到同一 device，再 pad（一般在 collate 里保持 CPU，训练 loop 里再 `.to(device)`）
4. **dtype 不一致**
   → 统一 dtype（token 用 long，embedding 用 float）
5. **padding_value 设错**
   比如你的 PAD token id 不是 0，而你 padding_value 还用 0
   → 后续 mask/embedding 就会乱

