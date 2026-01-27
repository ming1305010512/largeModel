好，这里我**系统、分层、偏实战**地把 **Python 的 `subprocess`** 给你讲清楚，目标是：
👉 **你看完后，能在真实项目里放心用，而不是“能跑但不懂”**。

------

# 一、`subprocess` 是干嘛的（一句话版）

> **`subprocess` 用来在 Python 里启动和控制“外部程序 / 命令行命令”**
> 比如：

- 执行 `ls / dir`
- 调用 `ffmpeg`
- 调用 `java / bash / sh`
- 跑一个 shell 脚本
- 在训练脚本里跑另一个 Python 程序

它是 **`os.system` 的现代、安全、可控替代品**。

------

# 二、为什么不用 `os.system`，而要用 `subprocess`

| 能力                 | os.system     | subprocess |
| -------------------- | ------------- | ---------- |
| 获取输出             | ❌             | ✅          |
| 获取返回码           | ❌             | ✅          |
| 传参安全             | ❌（容易注入） | ✅          |
| 实时读 stdout/stderr | ❌             | ✅          |
| 管道/重定向          | ❌             | ✅          |
| Windows / Linux 兼容 | 差            | 好         |

**结论**：
👉 **生产代码一律用 `subprocess`**

------

# 三、`subprocess` 的核心 API 结构

你只需要记住 **三层封装**（由低到高）：

```
Popen   →  最底层（最灵活）
run     →  最常用（推荐）
check_output →  简化版（只拿输出）
```

------

# 四、最推荐：`subprocess.run`（90% 场景）

## 1️⃣ 最基本用法（执行命令）

```python
import subprocess

result = subprocess.run(
    ["ls", "-l"],     # 命令 + 参数（强烈推荐用列表）
    capture_output=True,
    text=True
)

print(result.returncode)
print(result.stdout)
print(result.stderr)
```

### 返回的是一个 `CompletedProcess` 对象

常用字段：

- `result.returncode`：返回码（0 = 成功）
- `result.stdout`：标准输出
- `result.stderr`：错误输出

------

## 2️⃣ `shell=True` vs 不用 shell（**非常重要**）

### ✅ 推荐（安全）

```python
subprocess.run(["ls", "-l"])
```

### ⚠️ 不推荐（除非你很清楚）

```python
subprocess.run("ls -l", shell=True)
```

#### 区别

| 点       | shell=False（默认） | shell=True   |
| -------- | ------------------- | ------------ |
| 安全性   | ✅ 防命令注入        | ❌ 有注入风险 |
| 支持管道 | ❌                   | ✅            |
| 速度     | 快                  | 慢           |
| 跨平台   | 好                  | 差           |

> **原则**：
>
> - 能用列表 → 永远不用 `shell=True`
> - 只有用 `| && >` 等 shell 特性时才用

------

## 3️⃣ 捕获输出 vs 实时打印

### 捕获（等命令结束）

```python
subprocess.run(
    ["python", "train.py"],
    capture_output=True,
    text=True
)
```

### 实时打印（用于训练 / 下载进度）

👉 用 `Popen`（后面讲）

------

## 4️⃣ `check=True`：失败直接抛异常（推荐）

```python
subprocess.run(
    ["rm", "not_exist_file"],
    check=True
)
```

失败时会抛：

```python
subprocess.CalledProcessError
```

**适合：**

- 脚本化流程
- CI / 自动化任务
- 不允许悄悄失败的命令

------

# 五、`subprocess.Popen`（进阶 & 实时控制）

当你需要：

- 实时读取 stdout
- 长时间运行的进程
- 主进程与子进程通信

👉 用 `Popen`

------

## 1️⃣ 最典型：实时读取 stdout（训练/下载）

```python
import subprocess

p = subprocess.Popen(
    ["python", "train.py"],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True,
    bufsize=1
)

for line in p.stdout:
    print(line, end="")

p.wait()
```

**这在以下场景非常重要：**

- 模型训练日志
- tqdm 被吞
- huggingface 下载卡住

------

## 2️⃣ 手动终止进程

```python
p.terminate()  # SIGTERM
p.kill()       # SIGKILL
```

------

## 3️⃣ 等待并获取返回码

```python
code = p.wait()
```

------

# 六、stdin / stdout / stderr 的本质（你必须懂）

| 参数     | 含义             |
| -------- | ---------------- |
| `stdin`  | 子进程的输入     |
| `stdout` | 子进程的标准输出 |
| `stderr` | 子进程的错误输出 |

常见用法：

```python
stdout=subprocess.PIPE
stderr=subprocess.PIPE
stderr=subprocess.STDOUT
```

------

## 向子进程传输入（少见但有用）

```python
p = subprocess.Popen(
    ["python"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    text=True
)

p.stdin.write("print('hello')\n")
p.stdin.close()

print(p.stdout.read())
```

------

# 七、真实项目中的典型用法（结合你现在的工作）

## 1️⃣ 调用外部脚本（地址对齐 / 后处理）

```python
subprocess.run(
    ["python", "strong_fix.py", "--input", "data.json"],
    check=True
)
```

------

## 2️⃣ 执行 shell 脚本（仅当必要）

```python
subprocess.run(
    "bash deploy.sh",
    shell=True,
    check=True
)
```

------

## 3️⃣ 调用 MySQL（你现在很可能用得到）

```python
subprocess.run(
    ["mysql", "-u", "user", "-p123456", "db", "-e", "SELECT 1"],
    capture_output=True,
    text=True
)
```

------

## 4️⃣ 防止中文乱码（Windows 必看）

```python
subprocess.run(
    ["cmd", "/c", "dir"],
    text=True,
    encoding="gbk"
)
```

------

# 八、你现在最容易踩的坑（提前告诉你）

### ❌ 1. 用字符串 + shell=True 乱拼参数

```python
cmd = f"rm {filename}"   # filename 来自用户
subprocess.run(cmd, shell=True)
```

👉 **命令注入风险**

------

### ❌ 2. `capture_output=True` + tqdm

tqdm 会被“吞掉”，看起来像卡死
👉 改用 `Popen` 实时读

------

### ❌ 3. 不检查 returncode

```python
subprocess.run(...)
# 实际已经失败，但你不知道
```

👉 用 `check=True` 或检查 `returncode`

------

### ❌ 4. Windows 路径空格

```python
["C:\Program Files\xxx"]  # 会炸
```

👉 一定用列表参数，不要自己拼字符串

------

# 九、你可以直接记住的“使用模板”

### ✔ 安全、简洁、推荐

```python
subprocess.run(
    ["command", "arg1", "arg2"],
    check=True
)
```

### ✔ 需要输出

```python
res = subprocess.run(
    ["command"],
    capture_output=True,
    text=True
)
print(res.stdout)
```

### ✔ 实时日志（训练 / 下载）

```python
p = subprocess.Popen(
    ["command"],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True
)
for line in p.stdout:
    print(line, end="")
```

------

# 十、扩展

- 一个 **subprocess + tqdm 不冲突的标准模板**
-  **把现在 huggingface 下载 / 训练卡住的问题用 subprocess 重写**
- 写一个 **安全执行 shell 命令的工具函数**
- 结合 **地址对齐 pipeline**，设计一个「模型 → 规则 → subprocess 批处理」流程

