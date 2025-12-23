[TOC]

# 常用参数

## 1) `test_size`

测试集占比或数量。

- **float**：比如 `0.2` 表示 20% 做测试集
- **int**：比如 `2000` 表示测试集固定 2000 条样本

```
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
```

------

## 2) `train_size`

训练集占比或数量（和 `test_size` 类似）。

- 如果你只给了 `train_size`，剩下的自动当测试集
- 一般不和 `test_size` 同时用（除非你很明确要固定两边大小）

```
train_test_split(X, y, train_size=0.8)
```

------

## 3) `random_state`

随机种子，**保证每次划分结果可复现**。

```
train_test_split(X, y, test_size=0.2, random_state=42)
```

> 你在调参、写论文、复现实验时几乎都会加它。

------

## 4) `shuffle`

是否在划分前打乱数据，默认 `True`。

- **分类/回归的大多数情况：True（推荐）**
- **时间序列**：通常要 `False`（否则会“未来信息泄漏”）

```
train_test_split(X, y, shuffle=False)  # 时间序列常用
```

------

## 5) `stratify`

分层抽样：**让训练集/测试集的类别比例尽量和原数据一致**（分类任务非常常用）。

- 二分类、类别不平衡（比如 95%/5%）时尤其重要
- 一般传 `y`

```
train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)
```

> 注意：`stratify` 通常需要 `shuffle=True` 才有意义（默认就是 True）。

------

## 6) `X, y, *arrays`

你可以同时传入多个数组，它会**保持行对齐**一起切分。

常见：特征 `X`、标签 `y`，以及样本权重 `sample_weight`、分组信息等。

```
X_train, X_test, y_train, y_test, w_train, w_test = train_test_split(
    X, y, sample_weight, test_size=0.2, random_state=42
)
```

------

## 7) `return_train_test_split` 的返回顺序（容易搞错）

固定是：

> `X_train, X_test, y_train, y_test`（以及你传入的其他数组按同样顺序跟着返回）

------

# 典型场景怎么选参数

## A. 普通分类任务（最常见）

```
train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)
```

## B. 类别极度不平衡

仍然是 `stratify=y`，很关键：

```
train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)
```

## C. 时间序列（防泄漏）

```
train_test_split(X, y, test_size=0.2, shuffle=False)
```

> 时间序列更推荐用 `TimeSeriesSplit` 做交叉验证，但单次划分时这招最基本。

------

# 小坑提醒（你很可能会踩）

1. **不设 `random_state`**：你每次跑结果都不一样，不利于排查问题。
2. **分类不做 `stratify`**：测试集可能“刚好很偏”，评估不稳定。
3. **时间序列还 shuffle**：训练集会看到未来模式，线上直接崩（数据泄漏）。
4. `test_size` 太小：指标方差大；太大：训练不够充分（看数据量调）。