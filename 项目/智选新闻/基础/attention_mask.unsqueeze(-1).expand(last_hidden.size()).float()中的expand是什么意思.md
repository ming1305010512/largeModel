[TOC]

`expand` 的作用：**不复制数据**，只是把某些维度为 `1` 的张量“逻辑上广播”成更大的形状（返回一个 *view*），从而能和另一个张量做逐元素运算。

结合你这句：

```
attention_mask.unsqueeze(-1).expand(last_hidden.size()).float()
```

### 逐步看形状变化

假设：

- `last_hidden` 形状是 `[B, L, H]`（batch, 序列长度, hidden维）
- `attention_mask` 形状是 `[B, L]`（0/1，表示哪些token有效）

1. `unsqueeze(-1)`：在最后加一维
    `[B, L] -> [B, L, 1]`
2. `expand(last_hidden.size())`：把 `[B, L, 1]` **扩展**成 `[B, L, H]`
    也就是把那一列 mask 在 hidden 维上“铺开”，让每个 token 的 mask 值复制到 H 个 hidden 位置上（逻辑复制，底层不拷贝）。

这样你就能做类似：

```
masked_hidden = last_hidden * expanded_mask
```

把 padding 位置的 hidden 全部置 0。

### expand 的关键规则

- 只能把 **维度为 1 的轴**扩展到更大，比如 `1 -> H`
- 不能把 `L -> 2L` 这种非 1 的维度强行扩（会报错）
- 不会真实复制数据（更省内存），但很多后续操作如果需要连续内存，可能会触发拷贝

### 和 repeat 的区别（很重要）

- `expand`：不拷贝数据，靠“步长为0”的视图实现广播
- `repeat`：真的把数据复制出来，占内存