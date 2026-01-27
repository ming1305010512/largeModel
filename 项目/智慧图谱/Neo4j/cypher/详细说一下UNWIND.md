[TOC]

`UNWIND` 的本质就一句话：

> **把一个 List（列表）“拆开”，让列表里的每个元素各占一行。**
> （也可以把一个“列表的列表”拆成多行，甚至做两层 UNWIND 形成配对。）

它就是 `collect()` 的“反操作搭档”。

------

## 1）最基础：List → 多行

```cypher
WITH [1,2,3] AS xs
UNWIND xs AS x
RETURN x;
```

结果：

| x    |
| ---- |
| 1    |
| 2    |
| 3    |

✅ **xs 是一个列表**
✅ `UNWIND xs AS x` 把它拆成 3 行，每行一个 `x`

------

## 2）`collect` + `UNWIND`：典型组合

先把多行收成一个列表：

```cypher
MATCH (p:Paper)<-[:BELONG]-(:Question)-[:HAVE]->(c:Concept)
WITH p, collect(DISTINCT c) AS cs
```

此时 `cs` 类似：

```text
cs = [C1, C2, C3]
```

再拆开：

```cypher
UNWIND cs AS c
RETURN p, c;
```

结果就变回：

| p    | c    |
| ---- | ---- |
| P1   | C1   |
| P1   | C2   |
| P1   | C3   |

> 你可以把它理解成：
> `collect` 把“多行”变“一个篮子”；`UNWIND` 把“篮子”倒回“多行”。

------

## 3）两层 UNWIND：生成“全组合/配对”（你建 RELATED 就靠它）

### 例子：同一张试卷下的知识点两两配对

```cypher
MATCH (p:Paper)<-[:BELONG]-(:Question)-[:HAVE]->(c:Concept)
WITH p, collect(DISTINCT c) AS cs
UNWIND cs AS c1
UNWIND cs AS c2
RETURN p, c1, c2;
```

如果 `cs = [A,B,C]`，你会得到 9 行（笛卡尔积）：

(A,A) (A,B) (A,C)
(B,A) (B,B) (B,C)
(C,A) (C,B) (C,C)

然后我们通常会过滤成“唯一配对”：

```cypher
WHERE c1 <> c2 AND c1.id < c2.id
```

就剩：

(A,B), (A,C), (B,C)

最后 `MERGE` 关系：

```cypher
MERGE (c1)-[:RELATED]->(c2)
```

✅ 这就是“共现关系/相关关系/同卷关联”的常用写法。

------

## 4）UNWIND 也能“把行内数据展开成多行”，用于批量写入

比如你要批量创建关系（前端传入参数）：

```cypher
UNWIND $rows AS row
MATCH (a:Concept {id: row.from})
MATCH (b:Concept {id: row.to})
MERGE (a)-[:RELATED]->(b);
```

这里 `$rows` 是一个 List：

```json
[
  {"from":"c1","to":"c2"},
  {"from":"c2","to":"c3"}
]
```

`UNWIND` 会让每个 row 变成一行处理一次，非常适合批量导入/写入。

------

## 5）UNWIND 的常见坑（你以后一定会遇到）

### 坑 A：List 里有 `null`

如果列表里有 null，会被展开成一行 `x = null`，可能导致后续 MATCH 失败或误写。

解决：过滤掉

```cypher
UNWIND xs AS x
WITH x WHERE x IS NOT NULL
...
```

### 坑 B：两层 UNWIND 会“爆炸”

如果列表长度是 N，两层就是 N² 行。
N=1000 就是 1,000,000 行，可能直接慢/爆内存。

解决思路：

- 控制 N（比如每张试卷不要 collect 太多）
- 或者用阈值/抽样/限制
- 或者做“按题目共现”而不是“全知识点全连接”

------

## 6）用一句话帮助你记牢

- `collect()`：**多行 → 列表（收起来）**
- `UNWIND`：**列表 → 多行（拆开）**
- 两层 `UNWIND`：**从列表生成配对/组合（笛卡尔积）**

