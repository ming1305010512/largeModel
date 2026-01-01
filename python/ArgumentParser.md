[TOC]

`ArgumentParser` 通常指 Python 标准库 `argparse.ArgumentParser`：用来把命令行参数（`sys.argv`）解析成一个对象（Namespace），让你写 CLI 程序/训练脚本/数据处理脚本时不用自己手动切字符串、校验类型、输出 help。

下面按“你写训练脚本最常用到的能力”来讲。

------

## 1）最小用法：定义参数 → 解析 → 使用

```python
import argparse

parser = argparse.ArgumentParser(description="Train a model")
parser.add_argument("--epochs", type=int, default=3)
parser.add_argument("--lr", type=float, default=1e-4)
args = parser.parse_args()

print(args.epochs, args.lr)
```

命令行运行：

```bash
python train.py --epochs 10 --lr 3e-5
```

------

## 2）`add_argument` 常用参数（最重要的都在这）

```
parser.add_argument(name_or_flags, ...)
```

### 2.1 参数名写法：位置参数 vs 可选参数

- **位置参数**：必须按位置提供

  ```python
  parser.add_argument("data_path")
  ```

  运行：`python app.py /path/to/data`

- **可选参数**：用 `--xxx` 提供（推荐）

  ```python
  parser.add_argument("--data_path", required=True)
  ```

### 2.2 `type`：自动类型转换

```python
parser.add_argument("--batch_size", type=int, default=8)
```

用户传 `--batch_size abc` 会直接报错并给出提示。

### 2.3 `default`：默认值

```python
parser.add_argument("--seed", type=int, default=42)
```

### 2.4 `required`：是否必须提供（只对可选参数有意义）

```python
parser.add_argument("--model", required=True)
```

### 2.5 `choices`：限定取值范围（非常好用）

```python
parser.add_argument("--dtype", choices=["fp32", "fp16", "bf16"], default="bf16")
```

### 2.6 `help`：帮助说明（写清楚你未来会感谢自己）

```python
parser.add_argument("--lr", type=float, help="learning rate, e.g. 3e-5")
```

### 2.7 `action`：开关、计数、追加等（训练脚本很常用）

- **布尔开关**（Python 3.9+ 推荐 `BooleanOptionalAction`，但更常见是 store_true）

  ```python
  parser.add_argument("--use_amp", action="store_true")
  ```

  传了就是 True；不传就是 False。

- **计数器**（比如 -v -vv）

  ```python
  parser.add_argument("-v", action="count", default=0)
  ```

- **可重复参数收集为列表**

  ```python
  parser.add_argument("--tag", action="append")
  # --tag a --tag b -> ["a","b"]
  ```

### 2.8 `nargs`：一个参数接收多个值

- 固定 2 个：

  ```python
  parser.add_argument("--size", nargs=2, type=int)  # --size 224 224
  ```

- 任意多个：

  ```python
  parser.add_argument("--layers", nargs="+", type=int)  # --layers 2 4 6
  ```

- 可选 0 或 1 个：

  ```python
  parser.add_argument("--name", nargs="?")
  ```

### 2.9 `dest`：保存到 args 的字段名

```python
parser.add_argument("--learning-rate", dest="lr", type=float)
# args.lr
```

------

## 3）你在深度学习脚本里最常见的几个模式

### 3.1 “配置文件 + CLI 覆盖”

常见做法：先加载 yaml/json，再用命令行参数覆盖其中几项。

### 3.2 “训练/评估/预测”子命令（subcommands）

```python
parser = argparse.ArgumentParser()
sub = parser.add_subparsers(dest="cmd", required=True)

train = sub.add_parser("train")
train.add_argument("--epochs", type=int, default=3)

evalp = sub.add_parser("eval")
evalp.add_argument("--ckpt", required=True)

args = parser.parse_args()
```

用法：

```bash
python cli.py train --epochs 5
python cli.py eval --ckpt best.pt
```

------

## 4）错误提示与帮助输出（argparse 自带的“产品级体验”）

- 传错参数、漏必填参数，会自动打印 usage + 错误原因
- `-h/--help` 自动生成说明

这就是 argparse 最大的“值”：你不用自己写一堆校验逻辑。

------

## 5）两个经常踩坑的点

1. **布尔参数别用 `type=bool`**
   因为 `"False"` 在 Python 里也会变成 True（非空字符串都 True）。
   用 `action="store_true"` / `store_false` 或 `BooleanOptionalAction`。
2. **在 Jupyter 里 parse_args 会吃到 notebook 的参数**
   在 notebook 里运行时可能报错。解决方法：
   - 用 `args = parser.parse_args([])`（不给任何命令行参数）
   - 或用 `parse_known_args()` 忽略未知参数

------

## 6）给你一个“训练脚本实用模板”（你可以直接套）

```python
import argparse

def build_parser():
    p = argparse.ArgumentParser("train")
    p.add_argument("--model", default="bert-base-chinese")
    p.add_argument("--train_path", required=True)
    p.add_argument("--epochs", type=int, default=3)
    p.add_argument("--batch_size", type=int, default=8)
    p.add_argument("--lr", type=float, default=3e-5)
    p.add_argument("--max_length", type=int, default=256)
    p.add_argument("--seed", type=int, default=42)
    p.add_argument("--use_amp", action="store_true")
    p.add_argument("--dtype", choices=["fp32", "fp16", "bf16"], default="bf16")
    return p

def main():
    args = build_parser().parse_args()
    print(args)

if __name__ == "__main__":
    main()
```

