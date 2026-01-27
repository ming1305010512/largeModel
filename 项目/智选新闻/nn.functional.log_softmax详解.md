`log_softmax` 你可以把它理解成一句话：

> **先做 softmax 得到概率分布，再取对数，但用“数值稳定”的方式一次性算出来。**

它输出的是 **log 概率**（对数概率），不是 logits，也不是普通概率。

------

## 1) softmax 是什么（先回忆一下）

对一组 logits (z)（任意实数），softmax 把它变成概率：
$$
softmax(z_i)=\frac{e^{z_i}}{\sum_j e^{z_j}}
$$
特点：

- 每个值在 (0,1)
- 所有值加起来=1

------

## 2) log_softmax 是什么（定义）

$$
log_softmax(z_i)=\log(softmax(z_i))
= z_i - \log\left(\sum_j e^{z_j}\right)
$$

你会发现：它不是先算 softmax 再 log，而是有一个更紧凑的形式：

> **log_softmax = logits − logsumexp(logits)**

其中：
$$
logsumexp(z)=\log\left(\sum_j e^{z_j}\right)
$$

------

## 3) 为什么不直接 `torch.log(torch.softmax(...))` ？

因为会**数值不稳定**：

### 问题 A：exp 溢出

logits 可能很大，比如 1000：

- `exp(1000)` 直接溢出成 inf

### 问题 B：softmax 下溢

logits 很小或差距很大，softmax 会出现 0：

- `log(0)` → `-inf`

`log_softmax` 内部用稳定技巧避免这些问题。

------

## 4) 数值稳定的核心技巧：减最大值（log-sum-exp trick）

softmax 的稳定写法是：
$$
softmax(z_i)=\frac{e^{z_i - m}}{\sum_j e^{z_j - m}}
$$
其中$$ (m=\max_j z_j)$$

同理 log_softmax：
$$
log_softmax(z_i) = (z_i - m) - \log \left(\sum_j e^{z_j - m}\right)
$$
因为$$ (z_i-m \le 0)$$，`exp(<=0)` 不会溢出。

------

## 5) `dim=-1` 表示在哪个维度上做归一化

你写的：

```python
log_probs = F.log_softmax(logits, dim=-1)
```

`dim=-1` 是“最后一维”。

常见场景：

### 分类 logits: `[B, C]`

- dim=-1 等价于 dim=1
- 在类别维度上变成分布
- 输出 `[B, C]` 的 log 概率

### 生成 logits: `[B, T, V]`

- dim=-1 是 vocab 维度
- 每个时间步都对词表做一次分布
- 输出 `[B, T, V]` 的 log 概率

------

## 6) log_softmax 输出是什么样的数？

它输出的是 **log 概率**，所以：

- 每个值 ≤ 0（因为概率 ≤ 1）
- 指数回来就是概率：`probs = log_probs.exp()`
- 在 log 空间里相加对应概率相乘（很适合序列得分）

举例：如果某个 token 概率是 0.2
log 概率就是：
$$
\log(0.2)\approx -1.609
$$

------

## 7) 它和 CrossEntropyLoss / NLLLoss 的关系（非常关键）

### 7.1 CrossEntropyLoss

PyTorch 的 `nn.CrossEntropyLoss(logits, target)` **内部就做了 log_softmax**：

> CrossEntropyLoss = log_softmax + NLLLoss

所以训练分类时通常直接喂 logits，不需要手动 log_softmax。

### 7.2 NLLLoss

`nn.NLLLoss(log_probs, target)` 要求输入已经是 **log 概率**，所以你会看到：

```python
log_probs = F.log_softmax(logits, dim=-1)
loss = F.nll_loss(log_probs, target)
```

------

## 8) 在 beam search 里为什么用 log_softmax？

因为 beam search 用的是**累加得分**：

- 概率连乘：(\prod p_t)
- log 概率累加：(\sum \log p_t)

所以先把 logits 变成 log_probs 更方便且稳定。

------

## 9) 2 个小性质

如果 `log_probs = log_softmax(logits)`，那么：

1. `log_probs.exp().sum(dim=-1) == 1`（约等于）
2. `log_probs.max(dim=-1).values <= 0`

------

