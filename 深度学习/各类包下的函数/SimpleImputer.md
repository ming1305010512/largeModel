[TOC]



# SimpleImputer(strategy="mean")

这是 **Scikit-learn 中用于处理缺失值（NaN）的一个工具类**，常用于预处理阶段。它的作用是：

> **用“每一列的均值”来填补该列中的缺失值（NaN）**

## **所在模块**

```
from sklearn.impute import SimpleImputer
```

## 示例

```
import numpy as np
from sklearn.impute import SimpleImputer

data = np.array([
    [1, 2],
    [np.nan, 3],
    [7, 6]
])

imp = SimpleImputer(strategy='mean')   # 用每列的均值填充
data_filled = imp.fit_transform(data)
print(data_filled)
```

### 输出结果：

```
[[1.  2. ]
 [4.  3. ]
 [7.  6. ]]
```

解释：

- 第一列原始数据：[1, NaN, 7] → 均值 = (1+7)/2 = 4 → 替换 NaN
- 第二列原始数据：[2, 3, 6] → 无缺失，原样保留

## 参数详解

| 参数名           | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| `strategy`       | 填充策略：常见有 `"mean"`、`"median"`、`"most_frequent"`、`"constant"` |
| `missing_values` | 缺失值标记，默认是 `np.nan`                                  |
| `fill_value`     | strategy="constant" 时指定填充值                             |
| `add_indicator`  | 是否加一列“是否是缺失值”的指示列，默认为 False               |

## 与 `fit()` / `transform()` 关系

```
imp.fit(data)       # 计算每一列的均值
imp.transform(data) # 将 NaN 替换为对应列均值
```

也可以一步搞定：

```
imp.fit_transform(data)
```

## 和 Pandas 的 `.fillna()` 对比

| 功能             | Pandas `fillna()`       | Sklearn `SimpleImputer`      |
| ---------------- | ----------------------- | ---------------------------- |
| 使用方式         | 面向 Series / DataFrame | 面向 numpy array / DataFrame |
| 是否自动计算均值 | ❌ 需要手动：`df.mean()` | ✅ 自动计算                   |
| 与 Pipeline 配合 | ❌ 不支持自动集成        | ✅ 可以用于 `Pipeline`        |

## 注意事项

- 只能用于**数值型特征**（非数值时用 strategy='most_frequent'）
- 如果某列全是 NaN，则会报错（因为没法计算均值）
- fit 后会记录 `.statistics_` 属性，表示每列的填充值

## 总结：

> `SimpleImputer(strategy="mean")` 是机器学习数据清洗中非常常用的一个步骤，它可以**稳妥、自动、统一地用每列均值填补缺失值**，适合用于数值特征，并且可以无缝集成到 Sklearn 的 Pipeline 中。