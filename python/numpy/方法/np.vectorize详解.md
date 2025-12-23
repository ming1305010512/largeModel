[TOC]



## 一、`np.vectorize()` 是什么？

> ✅ `np.vectorize()` 的作用：
> 把一个 **“按元素处理标量的函数”**，包装成一个 **可以接收数组并对每个元素自动循环调用的函数**。

也就是：你本来要自己写 for 循环去遍历数组，现在可以交给 `np.vectorize()` 帮你“套一层壳”，用看起来很“向量化”的写法。

⚠️ 重点：
**它并不是真正的底层向量化（不会自动用 C/底层优化），性能上和 Python 循环差不多，只是写法更好看一点。**

------

## 二、基本用法与语法

### 1. 基本语法

```python
np.vectorize(pyfunc, otypes=None, excluded=None, signature=None)
```

常用其实就两个：`pyfunc` 和 `otypes`，其它用得少。

- `pyfunc`：你自己写的函数（一般按标量处理）
- `otypes`：输出的数据类型（可选，像 `"i"`，`"f"` 等）

### 2. 最常见用法：包装一个标量函数

```python
import numpy as np

def my_func(x):
    return x**2 + 1

vfunc = np.vectorize(my_func)

a = np.array([1, 2, 3, 4])
res = vfunc(a)
print(res)   # [ 2  5 10 17]
```

等价于你写：

```python
np.array([my_func(x) for x in a])
```

只是写法更 NumPy 风格。

------

## 三、深入看一下执行逻辑

`np.vectorize()` 做事情大概是：

1. 检查你传入的数组 `a`
2. 遍历 `a` 的每个元素 `x`
3. 对每个 `x` 调用 `pyfunc(x)`
4. 把结果收集起来，拼成一个新的 `ndarray`，形状与输入相同

从这一点就能看出来：**它并不是像 `x\**2` 这种真正的“底层向量化运算”**，而是 Python 层的循环。

------

## 四、`otypes` 参数：指定输出类型（很重要）

默认情况下，`np.vectorize()` 会先用你的函数跑一个元素，猜一下输出类型。
有时候会猜错，或者你的函数对不同输入返回类型不一致，会出各种奇怪问题，所以可以 **显式指定输出类型**：

```python
def my_func(x):
    return x / 2

vfunc = np.vectorize(my_func, otypes=[float])

a = np.array([1, 2, 3])
res = vfunc(a)   # dtype 会是 float
```

`otypes` 写法可以是：

- 一个列表，如 `[float]`
- 一个类型码字符串，如 `"i"`（int）、`"f"`（float）、`"O"`（object）

------

## 五、支持多参数函数

你写的函数可以有多个参数，只要把多个数组一块传进去就行，NumPy 会对它们按位置逐元素传参：

```python
def my_func(x, y):
    return x**2 + y

vfunc = np.vectorize(my_func)

a = np.array([1, 2, 3])
b = np.array([10, 20, 30])

res = vfunc(a, b)
print(res)  # [11, 24, 39]
```

相当于：

```python
[my_func(1,10), my_func(2,20), my_func(3,30)]
```

只要这几个数组能广播成相同形状即可。

------

## 六、`excluded` 参数：部分参数不参与向量化

有时候你函数有多个参数，其中有些是“固定参数”，不希望 numpy 对它做逐元素匹配，而是当作常量使用，就可以用 `excluded`：

```python
def my_func(x, a, b):
    return a * x + b

# a, b 不参与向量化，固定为标量
vfunc = np.vectorize(my_func, excluded=["a", "b"])

x = np.array([1, 2, 3])
res = vfunc(x, a=2, b=3)
print(res)  # [5, 7, 9]
```

`excluded` 可以是参数名（推荐）或者它们的位置 index。

------

## 七、`signature`（了解一下即可）

`signature` 用来描述更复杂的广播规则，类似 ufunc 的“通用函数签名”，比如矩阵乘法那种多维形状映射，这个一般日常用不到，深度自定义时才会涉及。

------

## 八、和“真正向量化运算”的区别（非常重要）

**真正的向量化**，比如：

```python
a = np.array([1,2,3,4])
a**2 + 1
np.sqrt(a)
np.where(a > 0, 1, -1)
```

这些运算都是 NumPy 内部用 C 实现的，速度很快。

而：

```python
vfunc = np.vectorize(my_func)
vfunc(a)
```

本质就是在 Python 层 `for` 循环一圈，只是帮你写好了：

```python
np.array([my_func(x) for x in a])
```

📌 **结论**：
如果某件事已经可以用“数组运算 + 广播”实现，**尽量别用 `np.vectorize()`，性能会更慢**。
`np.vectorize()` 更适合：

- 逻辑比较复杂、暂时写不出纯 NumPy 表达式的函数；
- 你想先快速跑通代码，之后再考虑优化。

------

## 九、什么时候适合用 `np.vectorize()`？

✔ 适合：

- 你已经有一个针对标量写好的 Python 函数（比如 if-else 很复杂）
- 希望它能“看起来”支持数组输入，而不想自己写 for 循环
- 数据量不是特别大，性能要求一般

示例：

```python
def grade(score):
    if score >= 90:
        return "A"
    elif score >= 80:
        return "B"
    else:
        return "C"

vgrade = np.vectorize(grade)
scores = np.array([95, 82, 60])
labels = vgrade(scores)
# array(['A', 'B', 'C'], dtype='<U1')
```

✔ 不太适合：

- 大规模数值运算（几百万、上千万元素）
- 明明可以用 `np.where`、布尔索引、广播解决的简单逻辑

比如上面的 `grade`，其实完全可以写成：

```python
labels = np.where(scores >= 90, "A",
                  np.where(scores >= 80, "B", "C"))
```

------

## 十、和「真正的 ufunc（通用函数）」的关系

- `np.sin`、`np.exp`、`np.add` 这些是 **真正的 ufunc**，底层 C 实现，超快
- `np.vectorize()` 可以把你写的 Python 函数“包装成一个看起来像 ufunc 的东西”，但它 **只是壳**，性能不是 ufunc 级别

你可以简单记：

> 👉 `np.vectorize = 伪向量化（for 循环语法糖）`
> 👉 `x + y、np.sin(x) = 真向量化（底层优化，非常快）`

------

## 十一、小结速查

- 作用：把标量函数包装成“可对数组逐元素调用”的函数

- 核心用法：

  ```python
  vfunc = np.vectorize(f, otypes=[float])
  res = vfunc(arr)
  ```

- 优点：写法优雅、易读、兼容多入参、自定义逻辑方便

- 缺点：**不加速！只是 for 循环语法糖**

- 替代优先级：
  能用「数组运算 / 广播 / np.where / 布尔索引」→ 优先用这些
  实在写不出来再考虑 `np.vectorize()`。

------

如果你手上有一个实际的函数，比如「根据血压+心率判断等级」、「根据三个特征分类严重程度」，你可以把标量版函数丢给我，我可以帮你写一个：

- `np.vectorize()` 版
- 和一个**真正“NumPy 风格向量化版”**，对比一下写法和思路，你会更有感觉。