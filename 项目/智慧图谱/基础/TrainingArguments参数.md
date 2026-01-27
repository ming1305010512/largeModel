[TOC]

 `TrainingArguments` 参数关注：**它控制什么、常见用法、和其它参数的联动坑点**。尽量用“训练过程的时间线”来解释（什么时候评估、什么时候保存、步数怎么数）。

> 说明：这些行为以 Hugging Face Transformers `Trainer/TrainingArguments` 官方文档为准。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

------

## 1) 输出与覆盖

### `output_dir: str`

**模型 checkpoint、日志、预测结果**等默认都往这里写。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
常见结构类似：

- `output_dir/checkpoint-xxxx/`
- `output_dir/runs/...`（TensorBoard 日志默认会放这里，除非你指定 `logging_dir`）

**建议：**

- 用“实验名+时间戳”做目录，避免覆盖和混淆。
- 如果要断点续训，通常让 `output_dir` 指向同一个实验目录。

------

### `overwrite_output_dir: bool = False`

是否允许 **覆盖 `output_dir` 里已有内容**。([知乎专栏](https://zhuanlan.zhihu.com/p/363670628?utm_source=chatgpt.com))

- `False`：如果目录已有训练产物，很多情况下会阻止你误覆盖（更安全）
- `True`：允许覆盖。你列的描述里提到“继续训练”，但更严谨地说：
  **继续训练主要靠 `resume_from_checkpoint` / `trainer.train(resume_from_checkpoint=...)`**，`overwrite_output_dir=True` 只是让目录可被重写，不等于自动续训。

**常见坑：**

- 你想“续训”，结果开了 `overwrite_output_dir=True` 但没指定 checkpoint → 等于重新开练并可能覆盖旧结果。

------

## 2) 评估（Evaluation）

### `eval_strategy: "no" | "steps" | "epoch" = "no"`

训练过程中**何时跑验证集评估**。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

- `"no"`：训练时不评估
- `"steps"`：每隔 `eval_steps` 个**更新步**评估一次
- `"epoch"`：每个 epoch 结束评估一次

> 注意：这里的 “steps” 指的是 **optimizer update step**（更新步），不是样本数、也不是 gradient accumulation 前的 micro-step。

------

### `eval_steps: int | float (optional)`

当 `eval_strategy="steps"` 时生效。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

- `int`：每隔多少个 **更新步** 评估一次
- `float` 且 `<1`：表示总训练更新步数的比例（例如 0.1 代表总步数的 10% 评估一次）

**默认行为：**

- 如果你没显式设置 `eval_steps`，它会默认跟 `logging_steps` 一样。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

------

### `eval_accumulation_steps: int (optional)`

评估/预测时，为了省显存：**先在设备上累积若干 step 的 logits/labels，再搬到 CPU**。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

- 不设：会把整个评估集的预测张量尽量留在 GPU 上再一次性搬走（更快但更吃显存）
- 设了：更省显存，但可能更慢一些

**NER 场景非常常用**：序列长、batch 大、label 多时，eval 很容易 OOM。

------

## 3) Batch 与步数体系（最容易搞混）

### `per_device_train_batch_size: int = 8`

每张卡/每个设备上，每一步 forward 的样本数。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

- 单卡：全局 batch = `per_device_train_batch_size`
- 多卡：全局 batch = `per_device_train_batch_size * 设备数`

------

### `per_device_eval_batch_size: int = 8`

评估时每个设备的 batch size。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
通常可以比训练大一点（因为不反传），但遇到长序列/大模型仍可能 OOM。

------

### `gradient_accumulation_steps: int = 1`

**梯度累积**：做 N 次 forward/backward 才 `optimizer.step()` 一次。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

- 作用：用较小显存模拟更大 batch（提高“有效 batch”）
- **有效全局 batch**（粗略理解）：
  `per_device_train_batch_size * 设备数 * gradient_accumulation_steps`

**最关键的点：step 的定义会变：**

- Trainer 里的 `logging_steps / eval_steps / save_steps` 这些 “steps” 指的是 **更新步（optimizer 更新次数）**
- 你每积累 `gradient_accumulation_steps` 次反传，才产生 1 个更新步
  所以：**评估/保存/日志会变得更稀疏**（间隔变大），这一点你列的说明也提到了。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

------

## 4) 学习率与训练时长

### `learning_rate: float = 5e-5`

AdamW 的初始学习率。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
实际训练中会被 scheduler（下面的 `lr_scheduler_type`）动态调整。

------

### `num_train_epochs: float = 3.0`

训练多少个 epoch。可以是小数（例如 2.5 表示跑完 2 个 epoch 再跑半个）。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

------

### `max_steps: int = -1`

设为正数时：**强制训练更新步数到 max_steps 即停**，并且会 **覆盖 num_train_epochs**。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

常见用途：

- 想让不同数据规模的实验“训练更新步数一致”
- 断点续训/算力预算固定（比如只允许跑 10k steps）

------

## 5) Scheduler 与 warmup

### `lr_scheduler_type: "linear" | ...`

学习率调度器类型（linear/cosine/constant/…）。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
默认 `"linear"`：通常是 **warmup 后线性衰减到 0**（或到很小）。

------

### `lr_scheduler_kwargs: dict = {}`

给特定 scheduler 的额外参数（不同 scheduler 支持的参数不同）。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
例如某些 scheduler 会需要 `num_cycles`、`power` 之类的参数（取决于你选的类型）。

------

### `warmup_ratio: float = 0.0`

warmup 步数 = `总训练更新步数 * warmup_ratio`。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
常见经验：0.03~0.1 之间较常见（看任务和数据规模）。

------

### `warmup_steps: int = 0`

直接指定 warmup 更新步数。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
**优先级高于 warmup_ratio**（你列的描述也写了）。
一般：你知道总步数就用 ratio；你严格控 warmup 就用 steps。

------

## 6) Logging（训练过程记录）

### `logging_dir: str (optional)`

TensorBoard 日志目录。默认是 `output_dir/runs/<时间戳_主机名>`。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

------

### `logging_strategy: "no" | "steps" | "epoch" = "steps"`

什么时候写日志。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

- `"steps"`：每隔 `logging_steps` 个更新步写一次
- `"epoch"`：每个 epoch 末写一次

------

### `logging_steps: int | float = 500`

- `int`：每多少个更新步记录一次
- `float < 1`：总更新步数的比例（例如 0.01 代表每 1% 记录一次）([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

**常见建议：**

- 训练步数很少（比如只有几百步）时，用比例更稳，不然 500 可能一次都不 log。

------

## 7) 保存 checkpoint（Save）

### `save_strategy: "no" | "steps" | "epoch" | "best" = "steps"`

训练过程中什么时候保存 checkpoint。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

- `"steps"`：每隔 `save_steps` 个更新步保存
- `"epoch"`：每个 epoch 末保存
- `"best"`：当出现新的 best_metric 时保存（通常配合评估）([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
- `"no"`：不保存（一般不推荐，除非你只想最后手动 save）

------

### `save_steps: int | float = 500`

仅在 `save_strategy="steps"` 时使用。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
同样支持 `<1` 的比例写法。

------

### `save_total_limit: int (optional)`

限制 output_dir 里**最多保留多少个 checkpoint**，超了会删旧的。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

并且你列的重点也非常关键：
如果启用了 `load_best_model_at_end`，**best checkpoint 会额外保留**，即使超过上限也尽量保留 best + 最近的。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

**常见建议：**

- 空间紧张：设 1 或 2
- 想要“保留 best + latest”：设 2 是最常见折中（社区也常这么建议）。([Stack Overflow](https://stackoverflow.com/questions/62525680/save-only-best-weights-with-huggingface-transformers?utm_source=chatgpt.com))

------

## 8) 设备与混合精度

### `use_cpu: bool = False`

是否强制用 CPU 训练。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
一般只有在你没有 GPU / 或做极小实验、或调试时会用。

------

### `bf16: bool = False`

是否启用 bfloat16 混合精度训练。需要硬件/后端支持（例如较新的 NVIDIA 架构等）。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
特点：bf16 的动态范围更大，通常比 fp16 更不容易数值溢出，但不是所有 GPU 都支持。

------

### `fp16: bool = False`

是否启用 fp16 混合精度训练。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))
优点：省显存、提速
风险：更容易数值不稳定（loss scale 等问题，但 Trainer 一般会处理）

**bf16 vs fp16 怎么选：**

- 机器支持 bf16 → 优先 bf16（通常更稳）
- 不支持 → fp16

> 一般不要同时把 bf16 和 fp16 都开；实际行为以版本实现为准，容易困惑。

------

## 9) 训练结束加载“最佳模型”

### `load_best_model_at_end: bool = False`

训练结束时，自动把模型权重切换为“训练过程中评估到的最佳 checkpoint”。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

**非常重要的约束（你列的也对）：**

- `save_strategy` 必须和 `eval_strategy` 一致
- 如果是 `"steps"`：`save_steps` 必须是 `eval_steps` 的整数倍（round multiple）([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

否则 Trainer 无法保证“评估时刻对应的 checkpoint”一定被保存下来。

------

### `metric_for_best_model: str (optional)`

指定“best”的比较指标名称（来自 evaluate 结果）。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

- 可以写 `"eval_f1"` 或 `"f1"`（带不带 `eval_` 都行，取决于返回键）
- 不指定时：在 `load_best_model_at_end=True` 的情况下默认用 `loss`（评估 loss）。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

**NER 常用：**

- `"eval_f1"` 或 `"eval_overall_f1"`（取决于你 compute_metrics 返回啥键）

------

### `greater_is_better: bool (optional)`

配合 `metric_for_best_model` 决定“越大越好”还是“越小越好”。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

默认规则（官方文档写得很清楚）：

- metric 名字以 `"loss"` 结尾 → 默认 `False`（越小越好）
- 否则 → 默认 `True`（越大越好）([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

**坑点：**

- 你用的是 `wer`（越小越好）或 `error_rate`（越小越好）但名字不含 loss → 默认会被当成越大越好，得手动设 `greater_is_better=False`。

------

## 10) 断点续训

### `resume_from_checkpoint: str (optional)`

你列得很关键：这个参数**不是 TrainingArguments 内部直接被 Trainer 自动消费的“核心字段”**，更多是脚本层面的约定；很多官方示例会把它作为命令行参数传入，再调用 `trainer.train(resume_from_checkpoint=...)`。([Hugging Face](https://huggingface.co/docs/transformers/en/main_classes/trainer?utm_source=chatgpt.com))

**它的作用**：从某个 checkpoint 目录恢复：

- 模型权重
- optimizer 状态
- scheduler 状态
- 训练步数、随机种子相关状态（取决于保存内容）

**与 `overwrite_output_dir` 的关系：**

- 续训的关键是 `resume_from_checkpoint`
- `overwrite_output_dir` 只是影响目录能否覆盖/写入，不等同续训

------

## 把这些参数放到一条“训练时间线”里理解（最清晰）

训练循环（简化）是这样：

1. 每个 micro-step：forward → backward（如果开了梯度累积）
2. 每累计到 `gradient_accumulation_steps` 次 micro-step：
   - 执行一次 optimizer update（这就是 1 个 **update step**）
   - 然后看：
     - 是否到 `logging_steps`？→ log
     - 是否到 `eval_steps` 且 `eval_strategy="steps"`？→ eval
     - 是否到 `save_steps` 且 `save_strategy="steps"`？→ save checkpoint
3. epoch 结束时：
   - 如果策略是 `"epoch"`，则 eval/save/log 在 epoch 末触发



## 11) 英文版

- **output_dir** (`str`, *optional*, defaults to `"trainer_output"`) — The output directory where the model predictions and checkpoints will be written.

- **overwrite_output_dir** (`bool`, *optional*, defaults to `False`) — If `True`, overwrite the content of the output directory. Use this to continue training if `output_dir` points to a checkpoint directory.

- **eval_strategy** (`str` or [IntervalStrategy](https://hf-mirror.com/docs/transformers/v4.56.1/en/internal/trainer_utils#transformers.IntervalStrategy), *optional*, default） s to `"no"`) — The evaluation strategy to adopt during training. Possible values are:`"no"`: No evaluation is done during training.`"steps"`: Evaluation is done (and logged) every `eval_steps`.`"epoch"`: Evaluation is done at the end of each epoch.

- **per_device_train_batch_size** (`int`, *optional*, defaults to 8) — The batch size *per device*. The **global batch size** is computed as: `per_device_train_batch_size * number_of_devices` in multi-GPU or distributed setups.

- **per_device_eval_batch_size** (`int`, *optional*, defaults to 8) — The batch size per device accelerator core/CPU for evaluation.

- **gradient_accumulation_steps** (`int`, *optional*, defaults to 1) — Number of updates steps to accumulate the gradients for, before performing a backward/update pass.When using gradient accumulation, one step is counted as one step with backward pass. Therefore, logging, evaluation, save will be conducted every `gradient_accumulation_steps * xxx_step` training examples.

- **eval_accumulation_steps** (`int`, *optional*) — Number of predictions steps to accumulate the output tensors for, before moving the results to the CPU. If left unset, the whole predictions are accumulated on the device accelerator before being moved to the CPU (faster but requires more memory).

- **learning_rate** (`float`, *optional*, defaults to 5e-5) — The initial learning rate for `AdamW` optimizer.

- **num_train_epochs(`float`,** *optional*, defaults to 3.0) — Total number of training epochs to perform (if not an integer, will perform the decimal part percents of the last epoch before stopping training).

- **max_steps** (`int`, *optional*, defaults to -1) — If set to a positive number, the total number of training steps to perform. Overrides `num_train_epochs`. For a finite dataset, training is reiterated through the dataset (if all data is exhausted) until `max_steps` is reached.

- **lr_scheduler_type** (`str` or [SchedulerType](https://hf-mirror.com/docs/transformers/v4.56.1/en/main_classes/optimizer_schedules#transformers.SchedulerType), *optional*, defaults to `"linear"`) — The scheduler type to use. See the documentation of [SchedulerType](https://hf-mirror.com/docs/transformers/v4.56.1/en/main_classes/optimizer_schedules#transformers.SchedulerType) for all possible values.

- **lr_scheduler_kwargs** (‘dict’, *optional*, defaults to {}) — The extra arguments for the lr_scheduler. See the documentation of each scheduler for possible values.

- **warmup_ratio** (`float`, *optional*, defaults to 0.0) — Ratio of total training steps used for a linear warmup from 0 to `learning_rate`.

- **warmup_steps** (`int`, *optional*, defaults to 0) — Number of steps used for a linear warmup from 0 to `learning_rate`. Overrides any effect of `warmup_ratio`.

- **logging_dir** (`str`, *optional*) — [TensorBoard](https://www.tensorflow.org/tensorboard) log directory. Will default to *output_dir/runs/**CURRENT_DATETIME_HOSTNAME\***.

- **logging_strategy** (`str` or [IntervalStrategy](https://hf-mirror.com/docs/transformers/v4.56.1/en/internal/trainer_utils#transformers.IntervalStrategy), *optional*, defaults to `"steps"`) — The logging strategy to adopt during training. Possible values are:`"no"`: No logging is done during training.`"epoch"`: Logging is done at the end of each epoch.`"steps"`: Logging is done every `logging_steps`.

- **logging_steps** (`int` or `float`, *optional*, defaults to 500) — Number of update steps between two logs if `logging_strategy="steps"`. Should be an integer or a float in range `[0,1)`. If smaller than 1, will be interpreted as ratio of total training steps.

- **save_strategy** (`str` or `SaveStrategy`, *optional*, defaults to `"steps"`) — The checkpoint save strategy to adopt during training. Possible values are:`"no"`: No save is done during training.`"epoch"`: Save is done at the end of each epoch.`"steps"`: Save is done every `save_steps`.`"best"`: Save is done whenever a new `best_metric` is achieved.If `"epoch"` or `"steps"` is chosen, saving will also be performed at the very end of training, always.

- **save_steps** (`int` or `float`, *optional*, defaults to 500) — Number of updates steps before two checkpoint saves if `save_strategy="steps"`. Should be an integer or a float in range `[0,1)`. If smaller than 1, will be interpreted as ratio of total training steps.

- **save_total_limit** (`int`, *optional*) — If a value is passed, will limit the total amount of checkpoints. Deletes the older checkpoints in `output_dir`. When `load_best_model_at_end` is enabled, the “best” checkpoint according to `metric_for_best_model` will always be retained in addition to the most recent ones. For example, for `save_total_limit=5` and `load_best_model_at_end`, the four last checkpoints will always be retained alongside the best model. When `save_total_limit=1` and `load_best_model_at_end`, it is possible that two checkpoints are saved: the last one and the best one (if they are different).

- **use_cpu** (`bool`, *optional*, defaults to `False`) — Whether or not to use cpu. If set to False, we will use cuda or mps device if available.

- **bf16** (`bool`, *optional*, defaults to `False`) — Whether to use bf16 16-bit (mixed) precision training instead of 32-bit training. Requires Ampere or higher NVIDIA architecture or Intel XPU or using CPU (use_cpu) or Ascend NPU. This is an experimental API and it may change.

  > 参考资料：https://en.wikipedia.org/wiki/Bfloat16_floating-point_format

- **fp16** (`bool`, *optional*, defaults to `False`) — Whether to use fp16 16-bit (mixed) precision training instead of 32-bit training.

- **eval_steps** (`int` or `float`, *optional*) — Number of update steps between two evaluations if `eval_strategy="steps"`. Will default to the same value as `logging_steps` if not set. Should be an integer or a float in range `[0,1)`. If smaller than 1, will be interpreted as ratio of total training steps.

- **load_best_model_at_end** (`bool`, *optional*, defaults to `False`) — Whether or not to load the best model found during training at the end of training. When this option is enabled, the best checkpoint will always be saved. See [`save_total_limit`](https://huggingface.co/docs/transformers/main_classes/trainer#transformers.TrainingArguments.save_total_limit) for more.When set to `True`, the parameters `save_strategy` needs to be the same as `eval_strategy`, and in the case it is “steps”, `save_steps` must be a round multiple of `eval_steps`.

- **metric_for_best_model** (`str`, *optional*) — Use in conjunction with `load_best_model_at_end` to specify the metric to use to compare two different models. Must be the name of a metric returned by the evaluation with or without the prefix `"eval_"`.If not specified, this will default to `"loss"` when either `load_best_model_at_end == True` or `lr_scheduler_type == SchedulerType.REDUCE_ON_PLATEAU` (to use the evaluation loss).If you set this value, `greater_is_better` will default to `True` unless the name ends with “loss”. Don’t forget to set it to `False` if your metric is better when lower.

- **greater_is_better** (`bool`, *optional*) — Use in conjunction with `load_best_model_at_end` and `metric_for_best_model` to specify if better models should have a greater metric or not. Will default to:`True` if `metric_for_best_model` is set to a value that doesn’t end in `"loss"`.`False` if `metric_for_best_model` is not set, or set to a value that ends in `"loss"`.

- **resume_from_checkpoint** (`str`, *optional*) — The path to a folder with a valid checkpoint for your model. This argument is not directly used by [Trainer](https://hf-mirror.com/docs/transformers/v4.56.1/en/main_classes/trainer#transformers.Trainer), it’s intended to be used by your training/evaluation scripts instead. See the [example scripts](https://github.com/huggingface/transformers/tree/main/examples) for more details.
