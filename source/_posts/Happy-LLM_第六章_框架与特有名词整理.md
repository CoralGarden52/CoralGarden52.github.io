---
title: happy-llm第六章：大模型训练流程实践——框架与特有名词整理
cover: /assets/posts/happy-llm/happy-llm-6.png 
categories: llm
tags:
  - happy-llm
---

> 基于 Datawhale《Happy-LLM》第六章 **大模型训练流程实践** 整理。
> 本文重点把第六章中出现的 **框架、训练组件、分布式并行术语、数据处理概念、PEFT 方法和偏好对齐名词** 集中解释清楚。
>
> 第六章主线：
>
> ```text
> Transformers
> → Pretrain
> → Trainer
> → DeepSpeed / 多卡分布式
> → SFT
> → PEFT
> → LoRA
> → Preference Alignment
> ```

---

# 1. 第六章整体内容

第五章主要是：

```text
自己手写 LLaMA2
自己写训练循环
自己实现 Tokenizer / Pretrain / SFT
```

第六章则进入更接近工业实践的方式：

```text
使用 Hugging Face Transformers
        ↓
加载成熟模型结构
        ↓
使用 datasets 处理数据
        ↓
使用 Trainer 管理训练
        ↓
使用 DeepSpeed 做分布式训练
        ↓
使用 PEFT / LoRA 做低成本微调
        ↓
进一步进入 RLHF / DPO / KTO 等偏好对齐
```

因此第六章最重要的意义是：

> **从“理解并手写大模型”过渡到“使用业界成熟框架训练和微调大模型”。**

---

# 2. 核心框架总览

| 名词 | 类型 | 核心作用 | 一句话理解 |
|---|---|---|---|
| Transformers | 模型/训练框架 | 加载和训练 BERT、GPT、LLaMA、Qwen、T5 等模型 | **大模型开发主框架** |
| Datasets | 数据处理框架 | 加载、映射、缓存、处理训练数据 | **训练数据工具箱** |
| Evaluate | 评估框架 | 统一管理常见评估指标 | **模型指标工具箱** |
| Trainer | Transformers 训练器 | 封装训练循环、保存、日志、分布式训练等 | **不用自己写完整训练循环** |
| DeepSpeed | 分布式训练框架 | ZeRO、混合精度、多卡训练、显存优化 | **大模型训练加速器** |
| PEFT | 参数高效微调框架 | LoRA、Prefix Tuning、P-Tuning 等 | **只训练少量参数** |
| WandB | 实验跟踪平台 | Loss、LR、GPU、超参、实验对比 | **训练实验看板** |
| SwanLab | 实验跟踪平台 | 训练可视化、超参记录、实验管理 | **国产实验可视化平台** |
| DDP | 数据并行策略 | 多 GPU 各算一份数据，再同步梯度 | **切数据，不切模型** |
| Megatron-LM | 大模型分布式框架 | 张量并行、流水线并行等 | **超大 Transformer 模型并行** |

---

# 3. Transformers

`Transformers` 是 Hugging Face 开发的深度学习模型框架，已经实现大量主流模型：

```text
BERT
GPT
LLaMA
Qwen
T5
ViT
...
```

开发者不需要重复手写网络结构，只需使用 Auto 类即可加载模型、配置和 Tokenizer。

Transformers 主要统一了：

```text
模型
Tokenizer
配置
训练
保存
加载
分布式训练
```

可以简单理解为：

> **Transformers = LLM 训练与调用的主框架。**

---

# 4. AutoConfig

`AutoConfig` 用来加载模型的**配置文件**。

例如：

```python
config = AutoConfig.from_pretrained(model_path)
```

配置通常来自：

```text
config.json
```

里面会保存：

```text
hidden_size
num_hidden_layers
num_attention_heads
num_key_value_heads
vocab_size
max_position_embeddings
模型类型
激活函数
RoPE 配置
...
```

因此：

> **AutoConfig 管“模型应该长什么样”，但本身不包含模型权重。**

---

# 5. AutoModelForCausalLM

这是专门用于 **Causal Language Modeling** 的模型自动加载类。

```python
model = AutoModelForCausalLM.from_config(config)
```

表示：

```text
按照 config 创建模型结构
+
随机初始化权重
```

而：

```python
model = AutoModelForCausalLM.from_pretrained(model_path)
```

表示：

```text
加载模型结构
+
加载已经训练好的权重
```

记忆：

```text
from_config
= 从零创建

from_pretrained
= 加载已有模型
```

---

# 6. AutoTokenizer

`AutoTokenizer` 用于自动加载模型配套 Tokenizer：

```python
tokenizer = AutoTokenizer.from_pretrained(model_path)
```

负责：

```text
文本
→ Token
→ Token ID
```

以及反向：

```text
Token ID
→ 文本
```

SFT 中还会结合：

```text
Chat Template
Special Tokens
BOS / EOS / PAD
```

---

# 7. Hugging Face Datasets

`datasets` 是 Hugging Face 的数据处理库。

常用：

```python
from datasets import load_dataset
```

例如：

```python
ds = load_dataset(
    "json",
    data_files="train.jsonl"
)
```

加载后常见对象：

```text
DatasetDict
```

可理解为：

```python
{
    "train": Dataset(...),
    "validation": Dataset(...),
    "test": Dataset(...)
}
```

---

# 8. Dataset.map()

`map()` 是 Datasets 中非常重要的数据预处理接口。

```python
tokenized_datasets = ds.map(
    tokenize_function,
    batched=True,
    num_proc=10
)
```

作用：

> **对整个 Dataset 批量执行数据处理函数。**

常见用途：

```text
Tokenizer
删除无用列
拼接文本
构造 labels
格式转换
```

其中：

```text
batched=True
```

表示批量处理，

```text
num_proc=10
```

表示使用多个进程加速预处理。

---

# 9. Packing / Grouping / Chunking

预训练数据中大量文本长度不一致。

如果每条文本都单独 Padding，会浪费很多计算。

因此常把多个 Token 序列：

```text
文本1
+
文本2
+
文本3
+
...
```

拼成一个长 Token 流，再按固定长度切块：

```text
2048 Token
2048 Token
2048 Token
...
```

这类操作通常叫：

```text
Packing
Grouping
Chunking
```

第六章使用 `group_texts()` 完成这一过程。

---

# 10. CLM

CLM：

> **Causal Language Modeling，因果语言建模**

训练目标：

$$
P(x_1,\dots,x_T) =
\prod_{t=1}^{T}
P(x_t|x_{<t})
$$

即：

> **根据前面的 Token，预测下一个 Token。**

GPT、LLaMA、Qwen 等 Decoder-Only LLM 的预训练核心任务都是 CLM。

---

# 11. Trainer

`Trainer` 是 Transformers 的高级训练器。

它封装：

```text
Forward
Loss
Backward
Optimizer
Learning Rate Scheduler
Gradient Accumulation
Checkpoint
Logging
Distributed Training
Evaluation
模型保存
```

因此无需自己完整手写训练循环，只需：

```python
trainer.train()
```

---

# 12. TrainingArguments

`TrainingArguments` 用于统一配置 Trainer 超参数。

常见：

| 参数 | 含义 |
|---|---|
| `output_dir` | 模型/Checkpoint 保存目录 |
| `per_device_train_batch_size` | 每张 GPU 的 Batch Size |
| `gradient_accumulation_steps` | 梯度累积步数 |
| `logging_steps` | 日志记录间隔 |
| `num_train_epochs` | Epoch 数 |
| `save_steps` | Checkpoint 保存间隔 |
| `learning_rate` | 学习率 |
| `gradient_checkpointing` | 梯度检查点 |
| `bf16` | BF16 训练 |
| `deepspeed` | DeepSpeed 配置文件 |
| `report_to` | 日志输出到 SwanLab / WandB 等 |

---

# 13. HfArgumentParser

`HfArgumentParser` 是 Transformers 的命令行参数解析工具。

它可以将：

```text
Shell 参数
```

自动解析成：

```text
Python dataclass
```

例如：

```python
parser = HfArgumentParser(
    (
        ModelArguments,
        DataTrainingArguments,
        TrainingArguments
    )
)
```

适合大型训练脚本统一管理配置。

---

# 14. Data Collator

`Data Collator` 可以理解为：

> **把 Dataset 中多个单样本整理成一个 Batch 的组件。**

例如：

```python
default_data_collator
```

常负责：

```text
Tensor 堆叠
Labels 整理
Batch 格式统一
```

记忆：

```text
Dataset
→ 单个样本

Data Collator
→ 多个样本组成 Batch
```

---

# 15. IterableWrapper

第六章使用：

```python
IterableWrapper(train_dataset)
```

来自 TorchData。

它的作用是：

> **把已有 Dataset 包装成可迭代数据流形式，便于 Trainer/DataPipe 使用。**

---

# 16. Checkpoint

Checkpoint：

> **训练过程中的阶段性快照。**

通常包括：

```text
模型权重
Optimizer 状态
Scheduler 状态
Global Step
随机状态
```

如果训练中断，可以从最近 Checkpoint 继续，而无需重新开始。

---

# 17. resume_from_checkpoint

`resume_from_checkpoint` 表示：

> **从已有 Checkpoint 恢复训练。**

第六章还使用：

```python
get_last_checkpoint()
```

自动找到输出目录中最近一次保存的 Checkpoint。

---

# 18. Logging

大型训练更推荐使用：

```python
logging
```

而不是大量 `print()`。

常见日志级别：

```text
DEBUG
INFO
WARNING
ERROR
CRITICAL
```

它便于：

```text
长期保存
统一格式
区分严重程度
训练异常排查
```

---

# 19. SwanLab

SwanLab 是：

> **机器学习实验跟踪和可视化平台。**

它不是用来运行训练代码的，而是：

```text
训练代码在服务器运行
        ↓
脚本记录训练指标
        ↓
同步给 SwanLab
        ↓
浏览器查看 Dashboard
```

常记录：

```text
train_loss
eval_loss
learning_rate
accuracy
GPU 使用率
超参数
训练时间
```

第六章通过：

```python
swanlab.init(...)
```

以及：

```text
--report_to swanlab
```

接入。

---

# 20. WandB

WandB 全称：

> **Weights & Biases**

也是实验跟踪平台。

主要功能：

```text
Loss 曲线
超参数记录
实验对比
GPU 监控
Sweeps
Artifacts
团队协作
```

简单理解：

```text
WandB
→ 国际使用广泛

SwanLab
→ 国内访问和中文生态更友好
```

---

# 21. SavingPolicy

第六章正文提到 `SavingPolicy`，但未展开具体实现。

可以理解为：

> **训练框架中管理“何时保存、保存什么、保留多少 Checkpoint”的策略组件。**

例如：

```text
每 100 Step 保存
每 Epoch 保存
只保存 Best Model
只保留最近 N 个 Checkpoint
```

注意：

> `SavingPolicy` 更像训练框架内部的策略概念，不是一个通用的独立 pip 包。

---

# 22. LoggingCallback

`LoggingCallback` 属于训练过程中的回调组件。

典型职责：

```text
记录 Loss
记录 Learning Rate
记录梯度范数
在固定 Step 执行日志逻辑
发送数据到 SwanLab / WandB
```

简单记：

```text
SavingPolicy
→ 管保存

LoggingCallback
→ 管日志
```

---

# 23. DeepSpeed

DeepSpeed 是 Microsoft 开源的：

> **大规模深度学习训练优化和分布式训练框架。**

主要解决：

```text
模型太大
显存不足
多卡训练复杂
训练速度慢
```

第六章使用：

```bash
deepspeed pretrain.py ...
```

启动多卡训练。

---

# 24. ZeRO

ZeRO：

> **Zero Redundancy Optimizer**

核心思想：

> **减少数据并行训练中模型状态在多张 GPU 上的重复存储。**

标准 DDP 中，每张 GPU 通常保存：

```text
完整参数
完整梯度
完整 Optimizer State
```

ZeRO 将这些状态分片到不同 GPU，从而降低单卡显存。

---

# 25. ZeRO 三个阶段

| 阶段 | 分片内容 |
|---|---|
| ZeRO-1 | Optimizer States |
| ZeRO-2 | Optimizer States + Gradients |
| ZeRO-3 | Optimizer States + Gradients + Parameters |

第六章使用：

```json
"zero_optimization": {
    "stage": 2
}
```

即 **ZeRO-2**。

---

# 26. ZeRO 是否需要多卡？

需要。

ZeRO 本质是：

> **分布式训练中的状态分片技术。**

准确理解：

```text
ZeRO
✅ 需要多 GPU

ZeRO
❌ 不等于传统模型并行
```

它的优势是可以在数据并行体系中通过状态分片降低显存，而不一定需要手工按层或按矩阵拆模型。

---

# 27. DDP

DDP：

> **Distributed Data Parallel，分布式数据并行**

核心：

```text
GPU0 → Batch A → 完整模型
GPU1 → Batch B → 完整模型
GPU2 → Batch C → 完整模型
```

每张卡独立：

```text
Forward
Backward
```

然后同步梯度，保证模型参数一致。

---

# 28. All-Reduce

All-Reduce 是 DDP 中最重要的通信操作之一。

假设：

```text
GPU0 gradient = 1
GPU1 gradient = 2
GPU2 gradient = 3
```

执行：

```text
All-Reduce SUM
```

后：

```text
GPU0 = 6
GPU1 = 6
GPU2 = 6
```

即：

> **所有 GPU 的数据先汇总，再让所有 GPU 都拿到汇总结果。**

DDP 中通常用它进行梯度同步。

---

# 29. Reduce / Broadcast / All-Reduce

| 操作 | 含义 |
|---|---|
| Reduce | 多卡数据汇总到一张卡 |
| Broadcast | 一张卡的数据发送给所有卡 |
| All-Reduce | 多卡汇总后，每张卡都得到结果 |
| Reduce-Scatter | Reduce 后，每张卡只保留一部分结果 |
| All-Gather | 收集各卡分片，使每张卡得到完整数据 |

---

# 30. Ring All-Reduce

Ring All-Reduce 是 All-Reduce 的高效实现之一。

GPU 排成环：

```text
GPU0 → GPU1 → GPU2 → GPU3 → GPU0
```

一般包含：

```text
Reduce-Scatter
+
All-Gather
```

优点：

> **避免所有通信都集中到某一张主卡，提升带宽利用率。**

---

# 31. Reduce-Scatter

可以理解成：

```text
先 Reduce
↓
把结果切开
↓
每张 GPU 只保留一部分
```

ZeRO-2 中梯度已分片，因此常使用 Reduce-Scatter。

---

# 32. All-Gather

All-Gather：

> **把各 GPU 上的不同分片收集起来，让每张 GPU 都得到完整数据。**

ZeRO-3 中参数本身被分片，因此 Forward/Backward 时常需要临时 All-Gather 参数。

---

# 33. overlap_comm

DeepSpeed 配置：

```json
"overlap_comm": true
```

表示：

> **通信与计算尽量重叠。**

例如：

```text
计算下一部分梯度
同时
同步上一部分梯度
```

用于隐藏通信延迟。

---

# 34. Gradient Accumulation

梯度累积：

```text
Mini-batch 1 → Backward
Mini-batch 2 → Backward
Mini-batch 3 → Backward
Mini-batch 4 → Backward
↓
Optimizer Step
```

有效 Batch Size：

$$
Batch_{effective} =
Batch_{per\_device}
\times
GPU_{num}
\times
AccumulationSteps
$$

作用：

> **显存不足时，用多个小 Batch 模拟大 Batch。**

---

# 35. Gradient Checkpointing

梯度检查点：

> **用计算换显存。**

正常训练会保存大量 Forward 激活值供 Backward 使用。

Gradient Checkpointing：

```text
只保存部分关键激活
↓
Backward 时重新计算其他激活
```

结果：

```text
显存 ↓
计算量 ↑
```

---

# 36. BF16 / FP16

| 类型 | 每元素大小 | 特点 |
|---|---:|---|
| FP32 | 4 Bytes | 稳定但显存高 |
| FP16 | 2 Bytes | 快，但数值范围较小 |
| BF16 | 2 Bytes | 指数范围接近 FP32，更适合 LLM 训练 |

第六章推荐使用 BF16。

---

# 37. torch.autocast

`torch.autocast` 是 PyTorch 的：

> **Automatic Mixed Precision（AMP）自动混合精度上下文管理器。**

例如：

```python
with torch.autocast(
    device_type="cuda",
    dtype=torch.bfloat16
):
    output = model(input)
```

作用：

> **根据算子类型自动选择适合的精度。**

目的：

```text
降低显存
+
提高速度
+
尽量保证数值稳定
```

---

# 38. DDP、ZeRO 与模型并行

| 方法 | 切什么 | 核心目的 |
|---|---|---|
| DDP | 数据 | 提升训练吞吐 |
| ZeRO | 参数/梯度/优化器状态 | 降低数据并行显存冗余 |
| Tensor Parallel | 单层矩阵 | 单层太大 |
| Pipeline Parallel | 模型层 | 模型整体太深/太大 |

---

# 39. 模型并行

Model Parallelism：

> **把模型本身拆到多张 GPU 上。**

广义模型并行主要包括：

```text
Tensor Parallelism
Pipeline Parallelism
```

---

# 40. Tensor Parallelism

张量并行：

> **把同一层中的大矩阵切到多张 GPU。**

例如：

$$
W =
\begin{bmatrix}
W_1 \\
W_2
\end{bmatrix}
$$

可能：

```text
GPU0 → W1
GPU1 → W2
```

特点：

```text
层内部通信频繁
更适合 NVLink 等高速互联
```

---

# 41. Pipeline Parallelism

流水线并行：

> **按模型层切分。**

例如 24 层模型：

```text
GPU0 → Layer 1~6
GPU1 → Layer 7~12
GPU2 → Layer 13~18
GPU3 → Layer 19~24
```

数据依次经过每个 Stage。

---

# 42. Micro-Batch

流水线并行为了减少空闲，会把大 Batch 拆成多个：

> **Micro-Batch**

例如：

```text
Batch
↓
M1
M2
M3
M4
```

不同 Stage 可以同时处理不同 Micro-Batch，提高 GPU 利用率。

---

# 43. Pipeline Bubble

Pipeline Bubble：

> **流水线中某些 Stage 因等待而处于空闲的时间。**

Micro-Batch 更多时，通常能更充分填满流水线，降低 Bubble 比例。

---

# 44. 3D Parallelism

3D 并行通常指：

```text
Data Parallelism
+
Tensor Parallelism
+
Pipeline Parallelism
```

也就是：

```text
切数据
+
切矩阵
+
切层
```

注意：

> **Pipeline Parallelism 本身只是模型并行的一种，并不等于“模型并行 + 数据并行”。**

---

# 45. Megatron-LM

Megatron-LM 是 NVIDIA 面向超大 Transformer 的训练框架。

核心能力包括：

```text
Tensor Parallelism
Pipeline Parallelism
Sequence Parallelism
Distributed Optimizer
```

适合：

> **模型本身已经大到单 GPU 放不下的场景。**

---

# 46. SFT

SFT：

> **Supervised Fine-Tuning，有监督微调**

典型数据结构：

```text
System
User
Assistant
```

目标：

> **让模型学会根据输入指令生成正确回答。**

---

# 47. Chat Template

Chat Template：

> **规定结构化对话如何转换成模型真正看到的 Token 序列。**

例如：

```text
System:
You are a helpful assistant.

User:
介绍 Transformer

Assistant:
Transformer 是……
```

会转换成类似：

```text
<|im_start|>system
You are a helpful assistant.
<|im_end|>

<|im_start|>user
介绍 Transformer
<|im_end|>

<|im_start|>assistant
Transformer 是……
<|im_end|>
```

因此：

> **SFT 和推理阶段最好使用一致的 Chat Template。**

---

# 48. System / User / Assistant

| Role | 作用 |
|---|---|
| System | 定义模型身份和规则 |
| User | 用户输入 |
| Assistant | 模型输出 |

SFT 中常见：

```text
System
→ 条件

User
→ 条件

Assistant
→ 监督目标
```

---

# 49. BOS / EOS / PAD

| Token | 全称 | 作用 |
|---|---|---|
| BOS | Beginning of Sequence | 序列开始 |
| EOS | End of Sequence | 序列结束 |
| PAD | Padding | 补齐长度 |

---

# 50. IGNORE_TOKEN_ID

SFT 中常把：

```text
System
User
Padding
```

对应 Label 设置为：

```text
-100
```

因为 PyTorch CrossEntropyLoss 默认：

```text
ignore_index = -100
```

所以这些位置不会计算 Loss。

这使 SFT 主要学习 Assistant Response。

---

# 51. PEFT

PEFT：

> **Parameter-Efficient Fine-Tuning，参数高效微调**

核心：

```text
不更新全部模型参数
只训练少量新增或选定参数
```

优点：

```text
显存占用低
训练成本低
Adapter 文件小
任务切换方便
```

常见方法：

```text
LoRA
Adapter Tuning
Prefix Tuning
P-Tuning
QLoRA
```

---

# 52. Adapter Tuning

Adapter Tuning：

> **在 Transformer 中插入小型 Adapter 模块，只训练 Adapter。**

典型结构：

```text
d
↓
Down Projection
↓
m
↓
Nonlinear
↓
Up Projection
↓
d
```

其中：

$$
m \ll d
$$

缺点：

> **增加额外网络结构，因此可能产生额外推理延迟。**

---

# 53. Prefix Tuning

Prefix Tuning：

> **冻结基础模型，只训练一组连续的任务相关 Prefix。**

可以理解为：

```text
Virtual Token
Virtual Token
Virtual Token
+
真实输入
```

缺点：

> **Prefix 会占用一定上下文长度。**

---

# 54. P-Tuning

P-Tuning 属于 Prompt/Prefix 参数高效微调路线。

核心：

> **训练连续 Prompt Embedding，而不是更新全部模型权重。**

---

# 55. Intrinsic Rank

Intrinsic Rank：

> **本征秩**

LoRA 的核心直觉是：

> **针对一个具体下游任务，真正需要发生变化的参数方向可能只位于低维子空间。**

因此权重更新矩阵：

$$
\Delta W
$$

不一定需要满秩，可以用低秩矩阵近似。

---

# 56. LoRA

LoRA：

> **Low-Rank Adaptation**

核心：

```text
冻结 W0
只训练低秩更新 ΔW
```

$$
W =
W_0
+
\Delta W
$$

其中：

$$
\Delta W =
BA
$$

$$
B
\in
\mathbb{R}^{d\times r}
$$

$$
A
\in
\mathbb{R}^{r\times k}
$$

且：

$$
r
\ll
\min(d,k)
$$

---

# 57. LoRA Forward

原始层：

$$
h =
W_0x
$$

LoRA 后：

$$
h =
W_0x
+
BAx
$$

训练时：

```text
W0 → Freeze
A/B → Train
```

因此可训练参数显著减少。

---

# 58. LoRA Rank：r

`r` 是 LoRA 的低秩维度。

常见：

```text
4
8
16
...
```

通常：

```text
r 越大
→ 参数更多
→ 表达能力更强
→ 显存和训练成本也更高
```

---

# 59. lora_alpha

`lora_alpha` 是 LoRA 更新缩放参数。

通常：

$$
scaling =
\frac{\alpha}{r}
$$

LoRA 分支最终会乘该缩放系数。

---

# 60. lora_dropout

`lora_dropout`：

> **LoRA 分支上的 Dropout。**

主要用于：

```text
降低过拟合
提升泛化
```

---

# 61. target_modules

`target_modules` 指定：

> **哪些模型层需要插入 LoRA。**

例如：

```python
target_modules = [
    "q_proj",
    "v_proj"
]
```

对应 Attention 中的 Query 和 Value Projection。

实际项目还常见：

```text
q_proj
k_proj
v_proj
o_proj
gate_proj
up_proj
down_proj
```

---

# 62. LoraConfig

PEFT 中：

```python
LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=8,
    lora_alpha=32,
    lora_dropout=0.1
)
```

用来定义 LoRA 超参数。

---

# 63. get_peft_model

```python
model = get_peft_model(
    model,
    peft_config
)
```

作用：

> **根据 PEFT 配置，把基础模型包装成带 LoRA Adapter 的可训练模型。**

之后仍可以直接交给 Trainer 训练。

---

# 64. PeftModel

`PeftModel`：

> **表示“基础模型 + PEFT Adapter”的模型封装。**

便于：

```text
加载 Adapter
保存 Adapter
切换 Adapter
合并 Adapter
```

---

# 65. QLoRA

第六章的章节说明和后文衔接提到 QLoRA，但正文主要详细展开 LoRA。

QLoRA：

> **Quantized LoRA**

核心：

```text
Base Model
→ 低比特量化
→ Freeze

LoRA Adapter
→ 保持可训练
```

相比普通 LoRA，可以进一步降低显存。

---

# 66. Full Fine-Tuning / LoRA / QLoRA

| 项目 | Full Fine-Tuning | LoRA | QLoRA |
|---|---|---|---|
| 更新全部参数 | ✅ | ❌ | ❌ |
| Base Model 冻结 | ❌ | ✅ | ✅ |
| 使用 Adapter | ❌ | ✅ | ✅ |
| Base Model 量化 | 通常否 | 通常否 | ✅ |
| 显存 | 最高 | 低 | 更低 |
| 适合资源受限 | 较差 | ✅ | ✅✅ |

---

# 67. Post-Training

Post-Training：

> **基础预训练完成后的后训练阶段。**

常见包括：

```text
SFT
Preference Alignment
Domain Adaptation
Reasoning Training
RL
```

记忆：

```text
Pretrain
→ 学知识和语言规律

Post-Training
→ 学任务、行为、偏好和能力
```

---

# 68. Preference Alignment

偏好对齐：

> **让模型在多个可能回答中，更倾向于人类认为更好的回答。**

关注：

```text
Helpful
Safe
Correct
Clear
Stable
```

通常在 SFT 之后进行。

---

# 69. RLHF

RLHF：

> **Reinforcement Learning from Human Feedback**

经典路线：

```text
SFT Model
↓
人类偏好数据
↓
Reward Model
↓
强化学习优化 Policy
```

---

# 70. DPO

DPO：

> **Direct Preference Optimization**

直接使用：

```text
Prompt
Chosen
Rejected
```

优化模型。

相比经典 RLHF：

```text
不需要显式训练 Reward Model
不需要完整 PPO 强化学习流程
```

---

# 71. KTO

KTO：

> **Kahneman-Tversky Optimization**

属于直接偏好优化路线。

它更强调：

```text
Good Response
Bad Response
```

之间的效用差异。

第六章只做概览，没有展开算法细节。

---

# 72. 框架和术语关系图

```text
Hugging Face
│
├── Transformers
│   ├── AutoConfig
│   ├── AutoTokenizer
│   ├── AutoModelForCausalLM
│   ├── TrainingArguments
│   ├── Trainer
│   └── HfArgumentParser
│
├── Datasets
│   ├── load_dataset
│   └── map
│
└── PEFT
    ├── LoRA
    ├── Adapter Tuning
    ├── Prefix / P-Tuning
    └── QLoRA

训练加速：
DeepSpeed
├── ZeRO-1
├── ZeRO-2
└── ZeRO-3

分布式并行：
├── DDP
│   └── All-Reduce
│
└── Model Parallel
    ├── Tensor Parallel
    └── Pipeline Parallel

实验跟踪：
├── SwanLab
└── WandB

训练阶段：
Pretrain
↓
SFT
↓
Preference Alignment
├── RLHF
├── DPO
└── KTO
```

---

# 73. 最值得记住的区别表

## Transformers / DeepSpeed / PEFT / SwanLab

| 框架 | 负责什么 |
|---|---|
| Transformers | 模型、Tokenizer、Trainer、训练流程 |
| DeepSpeed | 分布式、ZeRO、显存优化 |
| PEFT | LoRA 等低成本微调 |
| SwanLab / WandB | 训练实验记录与可视化 |

一句话：

```text
Transformers
→ 怎么训练模型

DeepSpeed
→ 怎么让大模型高效多卡训练

PEFT
→ 怎么少训练参数

SwanLab / WandB
→ 怎么观察训练过程
```

---

## DDP / Tensor Parallel / Pipeline Parallel

| 方法 | 切分对象 | 目的 |
|---|---|---|
| DDP | Batch / Data | 提升吞吐 |
| Tensor Parallel | 单层矩阵 | 单层太大 |
| Pipeline Parallel | 模型 Layer | 模型太深 |
| ZeRO | 参数/梯度/优化器状态 | 降低数据并行显存冗余 |

---

## Pretrain / SFT / PEFT / Preference Alignment

| 阶段/方法 | 目标 |
|---|---|
| Pretrain | 学语言规律和知识 |
| SFT | 学会遵循指令 |
| PEFT | 用低成本完成微调 |
| Preference Alignment | 学会什么回答更符合人类偏好 |

---

# 74. 一页速记

```text
第六章
= 大模型工业训练实践

Transformers
= LLM 主训练框架

Trainer
= 自动训练循环

Datasets
= 数据加载和预处理

DeepSpeed
= 大模型分布式训练
= ZeRO

DDP
= 数据并行
= 多卡完整模型
= All-Reduce 同步梯度

ZeRO
= 模型状态分片
= ZeRO-1 优化器
= ZeRO-2 + 梯度
= ZeRO-3 + 参数

Megatron-LM
= 模型并行
= Tensor Parallel
+ Pipeline Parallel

3D Parallel
= Data
+ Tensor
+ Pipeline

SwanLab / WandB
= 实验跟踪和可视化

SFT
= 指令监督微调

Chat Template
= 对话转成模型输入格式

IGNORE_TOKEN_ID = -100
= 不计算这些位置 Loss

PEFT
= 参数高效微调

Adapter
= 加小模块

Prefix Tuning
= 加可训练虚拟 Token

LoRA
= 冻结 W0
+ 训练 BA

QLoRA
= 量化 Base Model
+ LoRA

Post-Training
= Pretrain 之后的训练

Preference Alignment
= 对齐人类偏好

RLHF
= Reward Model + RL

DPO
= 直接利用偏好对

KTO
= 直接偏好优化方法
```

---

# 75. 最后总结

Happy-LLM 第六章最重要的是建立一套完整的工业训练框架认知：

```text
模型层：
Transformers

数据层：
Datasets

训练管理：
Trainer + TrainingArguments

分布式加速：
DeepSpeed / DDP / Megatron-LM

显存优化：
ZeRO
Gradient Checkpointing
BF16

实验监控：
SwanLab / WandB

监督微调：
SFT + Chat Template

低成本微调：
PEFT + LoRA / QLoRA

进一步后训练：
RLHF / DPO / KTO
```

最终一句话：

> **第五章教你“一个 LLM 内部是怎么写出来的”，第六章则教你“真正做项目时，如何借助成熟框架把 LLM 高效训练起来”。**
