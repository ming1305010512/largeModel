[TOC]

`DataCollatorWithPadding` 是 Hugging Face Transformers 里最常用的 **“动态 padding 拼 batch”** 工具：它在 **每个 batch 组装时**，把这一批样本里的不同长度序列 pad 到统一长度，然后转成张量喂给模型。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))

------

## 它到底做了什么（核心行为）

当 `DataLoader/Trainer` 取到一个 batch 的 `features`（一个 list，每个元素是字典，比如 `{"input_ids":..., "attention_mask":..., "label":...}`）时，它会：

1. 调用 `tokenizer.pad(...)` 按你设定的策略 padding，并生成对应的 `attention_mask` 等字段（取决于 tokenizer / 你的输入里有哪些键）。([Hugging Face](https://huggingface.co/transformers/v4.12.5/_modules/transformers/data/data_collator.html))
2. 如果 batch 里有 `label` 或 `label_ids`，它会把它们统一改名成 `labels`（模型更通用地接收 `labels` 这个键）。([Hugging Face](https://huggingface.co/transformers/v4.12.5/_modules/transformers/data/data_collator.html))

> 注意：它只负责把“输入序列” pad 起来；如果你的 **labels 本身也是序列**（比如 NER 的每个 token 一个 label），那就别用它，应该用 `DataCollatorForTokenClassification`（它会把 labels 也按同样长度 pad，并用 `label_pad_token_id=-100` 填充）。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))

------

## 常用参数逐个讲清楚

### 1) `tokenizer`

必须传。它决定了：

- pad 用哪个 id（`pad_token_id`）
- pad 在左边还是右边（`padding_side`）
- `tokenizer.pad()` 怎么补齐各字段（`input_ids/attention_mask/token_type_ids/...`）([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))

**常见坑**：有些模型/tokenizer 默认没有 pad token（如 GPT2 系），会导致 padding 报错；通常做法是把 `tokenizer.pad_token = tokenizer.eos_token` 或在模型 config 里指定。

------

### 2) `padding`（最关键）

控制“pad 到哪里”：

- `True` / `"longest"`：**pad 到当前 batch 里最长的那条**（最常用，节省算力）([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))
- `"max_length"`：pad 到 `max_length`（或模型允许的最大长度）([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))
- `False` / `"do_not_pad"`：不 pad（很少用；因为模型通常需要同长度张量）([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))

**建议**：训练时一般用 `True`（动态 pad），比你在预处理阶段把整个数据集统一 pad 到 max_length 更省显存/更快。

------

### 3) `max_length`

只有当你用 `"max_length"` 或需要截断/限制 padding 长度时才关键。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))
常见用法：你固定训练长度，比如 128/256/512，就设 `padding="max_length", max_length=256`。

------

### 4) `pad_to_multiple_of`

把 padding 后的长度再补齐到某个倍数（比如 8、16、32）。

用途：**更利于 GPU Tensor Cores**（尤其混合精度训练时常用），提升吞吐。官方文档也明确提到它对 NVIDIA Tensor Cores 有帮助。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))

经验值：

- fp16/bf16 训练常用 `8`（或 `16`，看你硬件/框架习惯）
- 如果你不追求极致性能，可以不设。

------

### 5) `return_tensors`

返回哪种张量类型：

- `"pt"`：PyTorch（最常用）
- `"np"`：NumPy
- （有些版本也支持 `"tf"`）([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))

------

## 它和 `tokenizer(..., padding=...)` 有什么区别？

- `tokenizer(..., padding="max_length")`：**预处理阶段**就把每条样本 pad 成一样长（全数据集都一样长）→ 简单但浪费算力/显存。
- `DataCollatorWithPadding(padding=True)`：**组 batch 时**才 pad 到“本 batch 最长”→ 一般更省、更快。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))

------

## 典型用法（分类任务）

```python
from transformers import AutoTokenizer, DataCollatorWithPadding

tokenizer = AutoTokenizer.from_pretrained("bert-base-chinese")

def preprocess(examples):
    return tokenizer(examples["text"], truncation=True)  # 不在这里padding

data_collator = DataCollatorWithPadding(
    tokenizer=tokenizer,
    padding=True,              # 动态pad到batch最长
    pad_to_multiple_of=8       # 可选：提速
)
```

Trainer 里直接传 `data_collator=data_collator` 即可。

------

## 什么时候不要用它？

- **Token classification / NER**：labels 是变长序列，需要同步 padding → 用 `DataCollatorForTokenClassification`。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))
- **Seq2Seq**（翻译/摘要）：decoder 侧 labels 也要处理、还可能要生成 `decoder_input_ids` → 用 `DataCollatorForSeq2Seq`。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/data_collator))

