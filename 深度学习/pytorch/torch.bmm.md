[TOC]

`torch.bmm` = **batch matrix multiply（批量矩阵乘法）**。它一次性对一批矩阵做“矩阵乘法”，相当于把很多个 `@` 打包一起算，速度更快、更省代码。

------

## 1) 形状规则（最重要）

`torch.bmm(A, B)` 要求：

- `A` 形状：`(B, n, m)`
- `B` 形状：`(B, m, p)`
- 输出：`(B, n, p)`

也就是：**batch 维度必须相同**，并且每个 batch 内做标准矩阵乘法 `(n×m) @ (m×p) = (n×p)`。

------

## 2) 最直观的理解

假设你有 `B` 个矩阵对：

- 第 0 对：`A[0] @ B[0]`
- 第 1 对：`A[1] @ B[1]`
- …
- 第 `B-1` 对：`A[B-1] @ B[B-1]`

`torch.bmm` 会一次算完，输出每对相乘的结果堆起来。

------

## 3) 一个小例子（能对上形状就不迷糊）

```python
import torch

A = torch.randn(10, 3, 4)   # (B=10, n=3, m=4)
B = torch.randn(10, 4, 5)   # (B=10, m=4, p=5)

C = torch.bmm(A, B)         # (10, 3, 5)
print(C.shape)              # torch.Size([10, 3, 5])
```

------

## 4) 常见用途：Attention 里做 QK^T、softmax、再乘 V

比如注意力里常见的：

- `scores = Q @ K^T`（对每个 batch 独立算）
- `out = softmax(scores) @ V`

用 `bmm` 写就是：

```python
scores = torch.bmm(Q, K.transpose(1, 2))   # (B, Lq, Lk)
attn = torch.softmax(scores, dim=-1)
out = torch.bmm(attn, V)                  # (B, Lq, d)
```

------

## 5) `torch.bmm` vs `torch.matmul` / `@`

### `bmm`

- **只支持 3D 张量**：`(B, n, m) bmm (B, m, p)`
- 语义明确：就是 batch 矩阵乘
- 一般在写 attention、图网络时很常用

### `matmul` / `@`

- 更通用：支持 1D/2D/ND，并且有广播规则
- 如果你写的是 `torch.matmul(A, B)`，形状可能会广播，逻辑更灵活但也更容易“看不懂/踩坑”

------

## 6) 常见报错原因（你大概率遇到过）

### (1) batch 维度不相同

`A: (B1, n, m)`，`B: (B2, m, p)`，`B1 != B2` 会报错。

### (2) 中间维不匹配

`A` 的最后一维必须等于 `B` 的倒数第二维：`m` 要对上。

------

## 7) 想“广播 batch”的情况怎么办？

`bmm` **不做 batch 广播**。如果你有：

- `A: (B, n, m)`
- `B: (1, m, p)` 或 `(m, p)`

你要先把 `B` 扩成 `(B, m, p)`，常见做法：

```python
B2 = B.expand(A.size(0), -1, -1)   # (B, m, p)
C = torch.bmm(A, B2)
```

或者直接用 `torch.matmul`（它能广播）：

```python
C = torch.matmul(A, B)  # B 可能是 (m,p) 或 (1,m,p)
```

------

## 8) 一个特别实用的“对照”写法

`bmm` 等价于下面这个循环（只是 `bmm` 更快）：

```python
C_list = []
for i in range(B):
    C_list.append(A[i] @ B[i])
C = torch.stack(C_list, dim=0)
```

