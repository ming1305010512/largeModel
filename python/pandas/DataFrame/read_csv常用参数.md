[TOC]



## 一、最最常用的几个参数

### 1. `filepath_or_buffer`

要读的文件路径 / 文件对象。

```python
df = pd.read_csv("data.csv")
```

------

### 2. `sep`：分隔符

默认是 **逗号`,`**。
如果是 **制表符（.tsv）、分号、竖线** 等，就要自己指定。

```python
# 制表符分隔
df = pd.read_csv("data.tsv", sep="\t")

# 竖线分隔
df = pd.read_csv("data.txt", sep="|")
```

------

### 3. `header`：表头所在行

- 默认：`header=0`，第一行是列名
- 没有表头：`header=None`，pandas 会自动用 0,1,2… 作为列名
- 表头在其他行：写对应行号

```python
# 没有表头
df = pd.read_csv("data.csv", header=None)

# 表头是第 2 行（从 0 数起）
df = pd.read_csv("data.csv", header=1)
```

------

### 4. `names`：自己指定列名

常用于：

- 原文件没有表头
- 或者想覆盖原来的列名

```python
cols = ["id", "name", "age"]
df = pd.read_csv("data.csv", header=None, names=cols)
```

> **注意**：如果文件本来有表头，你又传了 `names`，通常要配 `header=None`，否则原来的第一行会当数据读进去。

------

### 5. `index_col`：哪一列作为行索引

```python
# 用第一列作为 index
df = pd.read_csv("data.csv", index_col=0)

# 用某个字段做 index
df = pd.read_csv("data.csv", index_col="id")
```

------

### 6. `usecols`：只读取部分列（节省内存、提速）

```python
# 只读 name 和 age 两列
df = pd.read_csv("data.csv", usecols=["name", "age"])

# 或者按列号（从 0 开始）
df = pd.read_csv("data.csv", usecols=[0, 2, 5])
```

------

### 7. `dtype`：指定某些列的数据类型

解决：

- 自动识别错了
- 或者你要控制内存、保证类型一致

```python
df = pd.read_csv(
    "data.csv",
    dtype={
        "id": "int64",
        "age": "float32",
        "code": "string"
    }
)
```

------

### 8. `parse_dates`：把某些列解析成日期类型 `datetime64`

```python
df = pd.read_csv("data.csv", parse_dates=["order_date"])

# 多列合并成一个日期时间列
df = pd.read_csv(
    "data.csv",
    parse_dates={"datetime": ["date", "time"]}
)
```

还经常配合：

- `dayfirst=True`（日期格式是 31/12/2024 这种）
- `infer_datetime_format=True`（自动推断格式，加速）

------

### 9. `na_values` / `keep_default_na`：自定义缺失值

CSV 里有时不是用空字符串，而是写 `"NA"`, `"NULL"`, `"--"` 表示空。

```python
df = pd.read_csv(
    "data.csv",
    na_values=["NA", "NULL", "--"],  # 额外视为缺失
    keep_default_na=True             # 保留默认的缺失标记
)
```

------

### 10. `encoding`：文件编码

中文 CSV 最容易踩的坑。

```python
# 常见：
df = pd.read_csv("data.csv", encoding="utf-8")

# 如果出现乱码，可以试：
df = pd.read_csv("data.csv", encoding="gbk")
```

------

### 11. `nrows` / `skiprows`：只读一部分

- `nrows`：只读前几行
- `skiprows`：跳过前几行，或者跳过某些行号

```python
# 只读前 1000 行
df = pd.read_csv("data.csv", nrows=1000)

# 跳过前两行
df = pd.read_csv("data.csv", skiprows=2)

# 跳过指定的几行（行号从 0 算）
df = pd.read_csv("data.csv", skiprows=[0, 2, 5])
```

常用于：先抽几行“试读”看结构，或者日志文件前几行是说明文字时。

------

### 12. `sep` + `skipinitialspace`：处理怪格式

有的文件字段后面有多余空格：

```text
name, age, score
Tom,  18,  90
df = pd.read_csv("data.csv", sep=",", skipinitialspace=True)
```

可以自动去掉分隔符后面的空格。

------

## 二、进阶但很实用的参数

### 1. `converters`：对某一列自定义转换函数

比如某一列是 `"001", "002"`，你想保留前导 0：

```python
df = pd.read_csv(
    "data.csv",
    converters={
        "code": lambda x: x.zfill(5)   # 不足 5 位前面补 0
    }
)
```

或者某列是 `"yes"/"no"` 想转成 True/False：

```python
df = pd.read_csv(
    "data.csv",
    converters={
        "flag": lambda x: x == "yes"
    }
)
```

------

### 2. `thousands` / `decimal`：处理数字格式

```text
"1,234.56"  或  "1.234,56"
# 千位分隔符是逗号，小数点是点
df = pd.read_csv("data.csv", thousands=",", decimal=".")

# 欧洲式 小数用逗号
df = pd.read_csv("data.csv", thousands=".", decimal=",")
```

------

### 3. `error_bad_lines` / `on_bad_lines`（新版）

有些行列数不对、坏行，你不想因为一两行报错。

新版本用 `on_bad_lines`：

```python
df = pd.read_csv(
    "data.csv",
    on_bad_lines="skip"   # 直接跳过有问题的行
)
# 旧版本：error_bad_lines=False, warn_bad_lines=True
```

------

### 4. `chunksize`：分块读取（大文件必备）

文件特别大（几百 MB，几 GB），一口气读进内存会炸，就用 `chunksize` 分块。

```python
reader = pd.read_csv("big.csv", chunksize=100000)

for chunk in reader:
    # 每个 chunk 是一个 DataFrame
    # 可以逐块处理、聚合等
    ...
```

------

### 5. `lines=True`（其实是 to_json 的常用，这里提一嘴）

`read_csv` 没这个，别混了 😄
`lines=True` 是 `read_json` / `to_json` 行式 JSON 的参数。

------

## 三、一个稍微完整一点的例子

假设你有一个中文 CSV 文件：

- 制表符分隔（tsv）
- 第一行是说明文字，需要跳过
- 第二行是表头
- `id` 做索引
- 只要 `id, name, score, date` 四列
- `date` 是日期
- `"NA"` 也算缺失

你可以这样写：

```python
df = pd.read_csv(
    "data.tsv",
    sep="\t",
    header=1,                         # 第二行是表头（从 0 开始）
    index_col="id",                   # 用 id 做索引
    usecols=["id", "name", "score", "date"],
    parse_dates=["date"],             # 解析为日期
    na_values=["NA"],                 # 额外视为缺失
    encoding="utf-8"
)
```

------

## 四、快速记忆小表

| 目的          | 参数                      |
| ------------- | ------------------------- |
| 指定分隔符    | `sep=";"`, `sep="\t"`     |
| 指定表头行    | `header=0/1/None`         |
| 自定义列名    | `names=[...]`             |
| 指定索引列    | `index_col="id"`          |
| 只读部分列    | `usecols=[...]`           |
| 指定类型      | `dtype={"col": "int32"}`  |
| 解析日期      | `parse_dates=[...]`       |
| 缺失值规则    | `na_values=[...]`         |
| 跳过/限制行数 | `skiprows=..`, `nrows=..` |
| 大文件分块    | `chunksize=100000`        |
| 编码问题      | `encoding="utf-8"/"gbk"`  |