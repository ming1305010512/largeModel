[TOC]

`GradScaler` 是 PyTorch AMP（自动混合精度）里负责**动态 Loss Scaling** 的组件：它通过“把 loss 放大再反传”，让 fp16 下很小的梯度别因为精度/范围太小而直接 **underflow 变成 0**；同时它还能检测 **inf/nan**，在不稳定时自动调低缩放因子并跳过这一步更新，从而让训练更稳。

下面把它讲透：它解决什么问题、怎么工作的、每个常用参数什么意思、以及你在 BERT/Transformer 训练里常见的坑怎么避。

------

## 1）为什么需要 GradScaler（Loss Scaling 的直觉）

### fp16 的问题：动态范围小、有效位少

- fp16 能表示的最小正数和精度都比 fp32 差很多
- 反向传播里梯度经常很小（尤其深层网络、LayerNorm 后、或者学习率小/正则强时）
- 结果：**很多梯度在 fp16 里会下溢（underflow）变成 0** → 参数不更新 → “看起来在训练，但学不动”

### 解决：把 loss 乘一个大数 S

如果我们把 `loss` 乘以一个缩放因子 `S`：

- 反向得到的梯度也会被放大 `S` 倍
- 这些梯度就更不容易 underflow
- 然后在真正更新前，再把梯度除回 `S`（恢复原来的梯度尺度）

这就是 **Loss Scaling**。

------

## 2）GradScaler 做了什么（它的四步流程）

典型训练步骤是：

```python
scaler = torch.cuda.amp.GradScaler()

with torch.cuda.amp.autocast():
    loss = model(**batch).loss

scaler.scale(loss).backward()
scaler.step(optimizer)
scaler.update()
```

把它翻译成“发生了什么”：

### Step A：`scale(loss)`

- 返回一个“被放大过的 loss”：`loss_scaled = loss * S`
- 然后你对这个 `loss_scaled` 做 backward

### Step B：`backward()`

- 反传产生的梯度也会整体放大 `S` 倍
- 这样小梯度更不容易 underflow

### Step C：`step(optimizer)`

这一步最关键：GradScaler 会在内部做两件事

1. **先 unscale 梯度**：把每个参数的 `.grad` 除回 `S`（恢复真实尺度）
2. **检查 inf/nan**：如果发现梯度里有 inf/nan，说明刚才溢出了
   - 就**跳过 optimizer.step()**（不更新参数，避免把 nan 写进权重）
   - 并把后续缩放因子调小（交给 update）

> 你自己如果要做 `torch.nn.utils.clip_grad_norm_`，必须在“unscale 之后”再 clip（下面会给标准写法）。

### Step D：`update()`

- 如果上一步发现了 inf/nan：`S` 会减小（比如除以 `backoff_factor`）
- 如果连续很多步都没溢出：`S` 会增大（比如乘以 `growth_factor`）
  这就是 **动态 scaling**：尽量让 `S` 大到避免 underflow，又不大到导致 overflow。

------

## 3）常用参数逐个解释（你怎么调）

创建方式：

```python
torch.cuda.amp.GradScaler(
    init_scale=2.**16,
    growth_factor=2.0,
    backoff_factor=0.5,
    growth_interval=2000,
    enabled=True
)
```

- **`init_scale`**：初始缩放因子 S
  常见 `2**16`。太小可能 underflow；太大容易一开始就 overflow（会频繁跳 step）。
- **`growth_factor`**：多久不溢出就把 S 乘多少
  默认 2.0（翻倍）。
- **`backoff_factor`**：一旦溢出就把 S 乘多少
  默认 0.5（减半）。
- **`growth_interval`**：连续多少步没有溢出才增长一次
  默认 2000（比较保守、稳）。
- **`enabled`**：关掉 scaler（调试用）
  `enabled=False` 时，`scale/step/update` 基本变成“直通”，方便同一套代码在 fp32 下跑。

------

## 4）和 autocast 的关系：谁负责什么？

- `autocast`：决定哪些算子用 fp16/bf16 计算，主要解决**速度/显存**
- `GradScaler`：通过缩放避免 fp16 下梯度 underflow，并在溢出时自动处理，主要解决**数值稳定**

它俩通常配套使用（尤其 fp16）。
**但 bf16 通常不需要 GradScaler**（bf16 动态范围和 fp32 接近，主要是精度少一点），很多训练直接用 autocast(bf16) 而不 scaler。

------

## 5）必须掌握的一个关键细节：梯度裁剪怎么做

如果你要 clip grad，标准写法是：

```python
scaler.scale(loss).backward()

scaler.unscale_(optimizer)  # 先把梯度除回真实尺度
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

scaler.step(optimizer)
scaler.update()
```

原因：如果你不先 unscale，就会在“被放大的梯度”上裁剪，裁剪阈值等价被放大了，结果不对。

------

## 6）分布式/梯度累积时怎么配（常见坑）

### 梯度累积

你每次 backward 都在累积梯度，通常做法：

- 每个 micro-batch：`scaler.scale(loss).backward()`
- 累积到一定步数才 `scaler.step` 和 `scaler.update`

注意：

- 只有真正 step 的那次才调用 `scaler.step/update`
- `optimizer.zero_grad()` 也只在 step 之后做

### DDP

DDP 下同样适用，基本写法不变。
注意 inf/nan 导致的“跳步”在多卡上也会发生，通常框架会一致处理；但如果你自己写了很多自定义逻辑，要确保各 rank 行为一致（尤其是学习率调度器 step 的时机）。

------

## 7）你怎么判断它在正常工作？

你可以打印 scale：

```python
print(scaler.get_scale())
```

现象：

- 正常训练：scale 会缓慢上升或在某个区间稳定
- 如果经常溢出：scale 会频繁下降、step 经常被跳过（loss 可能突然变 nan/inf 或学习停滞）

------

## 8）什么时候不要用 GradScaler？

- 你用的是 **bf16**（通常不需要）
- 你在 CPU 上训练（AMP 不适用）
- 你完全 fp32（没意义）

