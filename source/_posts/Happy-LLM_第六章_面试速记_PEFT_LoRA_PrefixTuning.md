---
title: happy-llm第六章：大模型训练流程实践的阅读笔记
cover: /assets/posts/happy-llm/happy-llm-6.png 
categories: llm
tags:
  - happy-llm
---

# 面试速记

> 基于 Datawhale《Happy-LLM》第六章 **大模型训练流程实践** 整理，并补充 LoRA、PEFT、ZeRO、Prefix Tuning 与 FFN 投影层等面试高频知识。

## 背诵版总结

1. **Transformers** 负责模型、Tokenizer、Trainer 等训练主流程；**DeepSpeed** 负责大模型分布式与显存优化；**PEFT** 负责参数高效微调。
2. **ZeRO-1 / 2 / 3** 依次分片：Optimizer State → + Gradient → + Parameter。
3. **SFT** 与 Pretrain 都可使用 CLM，但 SFT 用指令数据，通常主要对 Assistant 回答区域计算 Loss。
4. **LoRA** 冻结原权重，只训练低秩更新：

$$
W =
W_0
+
BA
$$

5. `target_modules` 决定 LoRA 插在哪些层。经典配置：

```python
target_modules = ["q_proj", "v_proj"]
```

6. `q_proj` 生成 Query，`v_proj` 生成 Value。经典 LoRA 实验表明 Q/V 是一个很好的性能—参数量折中，但不是所有任务固定最优。
7. 如果注意力之外也要适配，可加入：

```text
gate_proj
up_proj
down_proj
```

8. LLaMA 类 FFN 常用 SwiGLU：

$$
FFN(x) =
W_{down}
\left(
SiLU(W_{gate}x)
\odot
W_{up}x
\right)
$$

9. `gate_proj` 是 FFN 的门控分支；`up_proj` 升到中间维度；`down_proj` 再投影回 Hidden Size。
10. **SiLU**：

$$
SiLU(x) =
x\sigma(x)
$$

   **GELU**：

$$
GELU(x) =
x\Phi(x)
$$

11. **Prefix Tuning** 冻结基础模型，只训练连续 Prefix；这些 Prefix 会注入各层 Attention 的 K/V，而不仅仅是第一层输入。
12. `logging` 设置为 `INFO` 后，会输出 `INFO/WARNING/ERROR/CRITICAL`，而过滤 `DEBUG`。

一句话：

> **第六章的重点是工业级 LLM 训练：Transformers/Trainer 管训练流程，DeepSpeed/ZeRO 解决多卡和显存，PEFT/LoRA/Prefix Tuning 解决低成本下游适配。**

---

## 面试表达模板

### 1. 第六章主要讲什么？

> 第六章从手写 LLM 过渡到工业级训练框架。使用 Hugging Face Transformers 加载模型、Tokenizer 和 Trainer，通过 DeepSpeed/ZeRO 做多卡分布式训练与显存优化，再使用 PEFT 中的 LoRA、Prefix Tuning 等方法做参数高效微调。

### 2. LoRA 是怎么实现的？

> LoRA 假设下游任务真正需要的权重更新具有较低秩，因此不训练完整权重矩阵，而是在目标层旁边增加两个低秩矩阵 A、B。原模型权重冻结，只更新 A、B。通过 target_modules 指定 LoRA 插入 q_proj、v_proj 或 FFN 等哪些层。

### 3. 为什么经典 LoRA 经常选 q_proj 和 v_proj？

> q_proj 改变模型“如何发起查询”，v_proj 改变被注意到的位置“实际提供什么信息”。原始 LoRA 的消融实验表明，对 Query 和 Value 投影做适配具有很好的参数效率和性能折中，因此成为经典配置。但现代复杂任务中也常同时适配 Q/K/V/O 和 FFN。

### 4. Prefix Tuning 怎么实现？

> Prefix Tuning 冻结基础模型，为每个任务学习连续的 Virtual Prefix。与 Prompt Tuning 只在第一层输入前增加 Soft Prompt 不同，Prefix Tuning 会把 Prefix 转换为各层 Attention 可使用的 K/V 前缀，让真实 Token 的 Query 同时关注 Prefix KV 和真实上下文 KV。

---

## 高频面试问题

### Q1：ZeRO-1、ZeRO-2、ZeRO-3 有什么区别？

| 阶段 | Optimizer State | Gradient | Parameter | 单卡显存 |
|---|---|---|---|---|
| ZeRO-1 | 分片 | 完整/同步 | 完整 | ↓ |
| ZeRO-2 | 分片 | **分片** | 完整 | ↓↓ |
| ZeRO-3 | 分片 | 分片 | **分片** | ↓↓↓ |

记忆：

```text
ZeRO-1：切优化器状态
ZeRO-2：再切梯度
ZeRO-3：再切参数
```

阶段越高，显存越低，但通信复杂度越高。

---

### Q2：`logging` 设置成 INFO，“只输出该级别及以上”是什么意思？

日志等级：

```text
DEBUG < INFO < WARNING < ERROR < CRITICAL
```

如果：

```python
logger.setLevel(logging.INFO)
```

则：

```text
DEBUG      → 不输出
INFO       → 输出
WARNING    → 输出
ERROR      → 输出
CRITICAL   → 输出
```

这里的“以上”指的是**严重程度等级更高**，不是代码位置更高。

---

### Q3：`target_modules` 是什么？

它告诉 PEFT：

> **在哪些原模型层上插入 LoRA Adapter。**

例如：

```python
target_modules = ["q_proj", "v_proj"]
```

表示遍历模型，找到各层的 Query Projection 和 Value Projection，在这些层上加入低秩更新。

---

### Q4：为什么还需要指定 `q_proj`、`v_proj`？

因为 LoRA 不应该默认在所有层都插入。

LoRA 的目标是：

> **用尽可能少的可训练参数获得足够强的任务适配能力。**

只用：

```python
["q_proj", "v_proj"]
```

参数少。

如果使用：

```python
[
    "q_proj",
    "k_proj",
    "v_proj",
    "o_proj",
    "gate_proj",
    "up_proj",
    "down_proj"
]
```

适配能力更强，但训练参数、显存和 Adapter 大小都会增加。

---

### Q5：q_proj、k_proj、v_proj、o_proj 分别是什么？

$$
Q =
XW_Q
$$

$$
K =
XW_K
$$

$$
V =
XW_V
$$

$$
Attention(Q,K,V) =
Softmax
\left(
\frac{QK^T}{\sqrt{d_k}}
\right)V
$$

| 层 | 作用 |
|---|---|
| `q_proj` | 生成 Query：我要找什么 |
| `k_proj` | 生成 Key：我可以怎样被匹配 |
| `v_proj` | 生成 Value：真正提供什么信息 |
| `o_proj` | 把 Attention 输出投影回 Hidden Size |

---

### Q6：为什么经典 LoRA 常选 `q_proj + v_proj`？

经典 LoRA 实验发现 Q/V 是一个很好的性能—参数量折中。

功能上：

```text
q_proj
→ 改变“怎么查询”

v_proj
→ 改变“取回什么内容”
```

但不要理解成：

```text
k_proj / o_proj 没作用
```

现代训练中经常适配：

```text
Q + K + V + O
```

甚至所有 Linear 层。

---

### Q7：为什么要加入 `gate_proj`、`up_proj`、`down_proj`？

只适配 Attention：

```text
主要改变 Token 之间如何交换信息
```

同时适配 FFN：

```text
还能改变每个 Token 的内部特征加工方式
```

因此复杂任务中常见：

```python
target_modules = [
    "q_proj",
    "k_proj",
    "v_proj",
    "o_proj",
    "gate_proj",
    "up_proj",
    "down_proj"
]
```

---

### Q8：`gate_proj` 在哪里用？

`gate_proj` 位于 LLaMA 类 Transformer Block 的 **FFN / MLP** 中。

流程：

```text
Hidden x
├── gate_proj → SiLU ─┐
│                     ×
└── up_proj ──────────┘
                      ↓
                  down_proj
                      ↓
                    Output
```

数学形式：

$$
g =
SiLU(W_{gate}x)
$$

$$
u =
W_{up}x
$$

$$
h =
g
\odot
u
$$

$$
FFN(x) =
W_{down}h
$$

`gate_proj` 的作用：

> **产生门控分支，控制 up_proj 产生的中间特征哪些被增强或抑制。**

---

### Q9：`up_proj` 和 `down_proj` 是什么？

在 LLaMA FFN 中：

```text
up_proj：
hidden_size
→ intermediate_size

gate_proj：
hidden_size
→ intermediate_size

down_proj：
intermediate_size
→ hidden_size
```

即：

```text
先升维
再降维
```

---

### Q10：这和 Adapter 的 down-project / up-project 一样吗？

**不一样。**

#### LLaMA FFN

```text
hidden
→ gate_proj / up_proj
→ 更大的 intermediate
→ down_proj
→ hidden
```

#### Bottleneck Adapter

```text
hidden
→ down-project
→ 更小 bottleneck
→ activation
→ up-project
→ hidden
```

所以：

```text
LLaMA FFN：先升维，再降维
Adapter：先降维，再升维
```

名称类似，但结构完全不同。

---

### Q11：SiLU 与 GELU 的区别？

#### SiLU

$$
SiLU(x) =
x
\cdot
\sigma(x)
$$

其中：

$$
\sigma(x) =
\frac{1}{1+e^{-x}}
$$

特点：

- 平滑；
- 允许负值；
- 有连续门控效果；
- 常与 GLU 结合形成 SwiGLU；
- LLaMA 类模型常用。

#### GELU

$$
GELU(x) =
x
\Phi(x)
$$

其中：

$$
\Phi(x)
$$

是标准高斯分布 CDF。

近似：

$$
GELU(x)
\approx
0.5x
\left[
1+
\tanh
\left(
\sqrt{\frac{2}{\pi}}
\left(
x+0.044715x^3
\right)
\right)
\right]
$$

常见于 BERT、GPT-2 等模型。

---

### Q12：SiLU 与 GELU 对比表

| 对比项 | SiLU | GELU |
|---|---|---|
| 公式核心 | \(x\sigma(x)\) | \(x\Phi(x)\) |
| 基于 | Sigmoid | Gaussian CDF |
| 平滑 | ✅ | ✅ |
| 可输出负值 | ✅ | ✅ |
| 常见组合 | SwiGLU | 标准 FFN / GeGLU |
| 常见模型 | LLaMA 类 | BERT、GPT-2 等 |
| 直观理解 | Sigmoid 控制 x 通过多少 | 高斯概率控制 x 通过多少 |

一句话：

> **两者都是平滑激活函数，SiLU 用 Sigmoid 做连续门控，GELU 用高斯累计概率做连续门控；LLaMA 的 SwiGLU 常用 SiLU。**

---

### Q13：教程里 `nn.Linear`、`nn.Embedding`、`nn.Conv2d` 三种有什么区别？

> 第六章教程代码语境中列出这三类 LoRA 目标层。当前 PEFT 已经支持更丰富的模块形式，因此不要把“三种”理解成当前版本的完整限制。

#### `nn.Linear`

线性映射：

$$
y =
xW^T+b
$$

Transformer 中：

```text
q_proj
k_proj
v_proj
o_proj
gate_proj
up_proj
down_proj
```

大多都是 `nn.Linear`。

因此文本 LLM 的 LoRA 目标绝大多数属于 Linear。

#### `nn.Embedding`

作用：

```text
Token ID
↓
查 Embedding Table
↓
Dense Vector
```

参数矩阵：

$$
E
\in
\mathbb{R}^{V\times d}
$$

本质是按 ID 查表，不是普通的全连接层。

#### `nn.Conv2d`

二维卷积层，常处理：

```text
Image
Feature Map
二维局部结构
```

更常见于 CNN、Vision、Diffusion，纯文本 LLM 中较少。

---

### Q14：三类层对比

| 类型 | 输入 | 核心操作 | 文本 LLM 常见程度 | LoRA 场景 |
|---|---|---|---|---|
| `nn.Linear` | Hidden State | 矩阵乘法 | **极高** | Attention / FFN |
| `nn.Embedding` | 离散 ID | 查表 | 高 | Token Embedding |
| `nn.Conv2d` | 2D Feature Map | 卷积 | 很低 | Vision / Diffusion |

---

### Q15：Prefix Tuning 是怎么实现的？

Prefix Tuning 不是真的把一段自然语言放到 Prompt 前面。

它学习的是：

> **连续可训练向量（Continuous Virtual Prefix）**

并冻结整个 Base Model。

---

# 第六章详细总结

## 1. 第六章的定位

第五章：

```text
手写 LLaMA
手写训练循环
```

第六章：

```text
Transformers
+
Datasets
+
Trainer
+
DeepSpeed
+
PEFT
```

重点：

> **从教学实现进入真实 LLM 工程训练。**

---

# 2. Transformers / Trainer

Transformers 提供：

```text
AutoConfig
AutoTokenizer
AutoModelForCausalLM
TrainingArguments
Trainer
HfArgumentParser
```

Trainer 封装：

```text
Forward
Loss
Backward
Optimizer
Scheduler
Gradient Accumulation
Checkpoint
Logging
Distributed Training
```

---

# 3. Pretrain 与 SFT

二者都可以使用 CLM：

$$
P(x_1,\dots,x_T) =
\prod_{t=1}^{T}
P(x_t|x_{<t})
$$

| 对比项 | Pretrain | SFT |
|---|---|---|
| 数据 | 无监督文本 | 指令数据 |
| 目标 | 学知识与语言 | 学指令遵循 |
| Loss | 大多数有效 Token | 通常主要 Assistant |

---

# 4. ZeRO 原理

标准 DDP：

```text
每张 GPU
都有完整 Parameter
完整 Gradient
完整 Optimizer State
```

ZeRO 逐级分片。

```text
DDP：
[P G O] [P G O] [P G O]

ZeRO-1：
[P G O1] [P G O2] [P G O3]

ZeRO-2：
[P G1 O1] [P G2 O2] [P G3 O3]

ZeRO-3：
[P1 G1 O1] [P2 G2 O2] [P3 G3 O3]
```

其中：

```text
P = Parameter
G = Gradient
O = Optimizer State
```

---

# 5. PEFT

PEFT：

> **Parameter-Efficient Fine-Tuning**

核心：

```text
冻结大部分 Base Model
+
只训练少量新增参数
```

常见：

```text
Adapter Tuning
Prefix Tuning
P-Tuning
LoRA
QLoRA
```

---

# 6. Adapter Tuning

Bottleneck Adapter：

```text
x
│
├─────────────────────┐
↓                     │
down-project          │
d → r                 │
↓                     │
Activation            │
↓                     │
up-project            │
r → d                 │
│                     │
└────── + x ──────────┘
          ↓
        output
```

其中：

$$
r
\ll
d
$$

基础模型冻结，只训练 Adapter。

---

# 7. Prefix Tuning

## 7.1 Prefix 是什么？

假设有：

```text
P1 P2 ... Pm
```

它们不是词表里的 Token，而是：

> **模型直接学习出来的连续向量。**

---

## 7.2 Prefix Tuning vs Prompt Tuning

Prompt Tuning：

```text
Virtual Token
↓
只放在第一层 Input Embedding 前
```

Prefix Tuning：

```text
Virtual Prefix
↓
注入 Transformer 各层 Attention
```

核心区别：

> **Prefix Tuning 影响多层 Attention；Prompt Tuning 主要修改输入层。**

---

# 8. Prefix Tuning 的具体实现

原本第 \(l\) 层：

$$
K^{(l)} =
X^{(l)}W_K^{(l)}
$$

$$
V^{(l)} =
X^{(l)}W_V^{(l)}
$$

Prefix Tuning 为每层增加：

$$
K_{prefix}^{(l)}
$$

$$
V_{prefix}^{(l)}
$$

拼接：

$$
\tilde{K}^{(l)} =
[
K_{prefix}^{(l)};
K^{(l)}
]
$$

$$
\tilde{V}^{(l)} =
[
V_{prefix}^{(l)};
V^{(l)}
]
$$

然后：

$$
Attention
\left(
Q^{(l)},
\tilde{K}^{(l)},
\tilde{V}^{(l)}
\right)
$$

于是每个真实 Token 的 Query 都可以关注：

```text
Prefix K/V
+
真实 Token K/V
```

---

# 9. Prefix Encoder

实际实现常用一个小型 Prefix Encoder：

```text
Virtual Token IDs
↓
Prefix Embedding
↓
MLP / Projection
↓
Layer 1 Prefix K/V
Layer 2 Prefix K/V
...
Layer N Prefix K/V
```

训练：

```text
Base LLM
→ Frozen

Prefix Encoder
→ Trainable
```

---

# 10. Hugging Face Prefix Tuning

```python
from peft import PrefixTuningConfig, get_peft_model

config = PrefixTuningConfig(
    task_type="CAUSAL_LM",
    num_virtual_tokens=20
)

model = get_peft_model(
    base_model,
    config
)
```

关键：

```text
num_virtual_tokens
→ Prefix 长度

prefix_projection
→ 是否通过 MLP 将 Prefix Embedding 投影到各层 KV
```

---

# 11. Prefix Tuning 完整流程

```text
加载预训练 LLM
↓
冻结所有 Base 参数
↓
创建 Virtual Prefix
↓
Prefix Encoder
↓
生成每层 Prefix K/V
↓
与真实 K/V 拼接
↓
Attention
↓
Task Loss
↓
Backward
↓
只更新 Prefix
```

---

# 12. Prefix Tuning 优缺点

优点：

- 训练参数少；
- 一个 Base Model 可以切换多个任务 Prefix；
- Checkpoint 小；
- 不破坏基础模型权重。

缺点：

- Prefix 太短，表达能力不足；
- Prefix 太长，Attention/KV 开销上升；
- 复杂知识注入能力有限；
- 不同模型结构效果差异明显。

---

# 13. LoRA 原理

$$
y =
W_0x
+
BAx
$$

训练：

```text
W0 → Frozen
A/B → Trainable
```

如果：

$$
W_0
\in
\mathbb{R}^{d\times k}
$$

Full FT 要训练：

$$
dk
$$

个参数。

LoRA：

$$
A
\in
\mathbb{R}^{r\times k}
$$

$$
B
\in
\mathbb{R}^{d\times r}
$$

参数数：

$$
r(k+d)
$$

当：

$$
r
\ll
d,k
$$

时，训练参数大幅下降。

---

# 14. `target_modules` 的匹配

教程用正则说明目标层查找。

模块名可能是：

```text
model.layers.0.self_attn.q_proj
model.layers.0.self_attn.v_proj
model.layers.1.self_attn.q_proj
...
```

当前 PEFT 中：

- `target_modules` 传字符串时可按 Regex 匹配；
- 传列表时一般按具体名称或名称后缀匹配。

所以：

```python
["q_proj", "v_proj"]
```

会匹配各 Transformer Block 中同名投影层。

---

# 15. 不同模型的层名不同

| 模型 | Query 投影常见名字 |
|---|---|
| LLaMA / Qwen | `q_proj` |
| BERT | `query` |
| GPT-2 | `c_attn` 中 QKV 融合 |
| T5 | `q` |

所以使用 LoRA 前最好：

```python
for name, module in model.named_modules():
    print(name)
```

确认实际层名。

---

# 16. 常见 LoRA 配置

## 经典省参数

```python
target_modules = [
    "q_proj",
    "v_proj"
]
```

## 全 Attention

```python
target_modules = [
    "q_proj",
    "k_proj",
    "v_proj",
    "o_proj"
]
```

## Attention + FFN

```python
target_modules = [
    "q_proj",
    "k_proj",
    "v_proj",
    "o_proj",
    "gate_proj",
    "up_proj",
    "down_proj"
]
```

## 当前 PEFT 便捷配置

```python
target_modules = "all-linear"
```

适合希望对多数 Linear 类层进行 LoRA 适配的情况。

---

# 17. FFN 投影层

LLaMA FFN：

$$
FFN(x) =
W_{down}
\left(
SiLU(W_{gate}x)
\odot
W_{up}x
\right)
$$

其中：

```text
gate_proj
hidden → intermediate
负责门控

up_proj
hidden → intermediate
负责内容分支

down_proj
intermediate → hidden
负责映射回模型维度
```

---

# 18. 为什么 FFN 也适合 LoRA？

Attention：

```text
负责 Token 间的信息交互
```

FFN：

```text
负责每个 Token 内部表示的非线性加工
```

因此只微调 Attention 与同时微调 FFN 的能力不同。

复杂任务中加入：

```text
gate_proj
up_proj
down_proj
```

通常能提供更强的适配空间。

---

# 19. PEFT 三类典型层

## `nn.Linear`

```text
Hidden State
↓
矩阵乘法
↓
新 Hidden State
```

LLM 中最常见。

## `nn.Embedding`

```text
Token ID
↓
Embedding 查表
↓
Vector
```

用于 Token Embedding。

## `nn.Conv2d`

```text
2D Feature Map
↓
卷积核
↓
新的 Feature Map
```

更常用于视觉/扩散模型。

---

# 20. 当前 PEFT 的版本补充

第六章教程的“三种层”属于教程当时实现语境。

当前 PEFT 已支持更丰富的用法，例如：

```python
target_modules="all-linear"
```

以及：

```text
target_parameters
```

可用于部分 MoE 中并非 `nn.Linear`、而是直接存成 `nn.Parameter` 的融合专家权重。

所以实际工程中：

> **应以当前 PEFT 文档 + 模型自身结构为准。**

---

# 21. Prefix Tuning / Prompt Tuning / LoRA 对比

| 方法 | Base Model | 新参数位置 | 虚拟 Token | 特点 |
|---|---|---|---|---|
| Prompt Tuning | Frozen | 输入层 | ✅ | 最简单 |
| Prefix Tuning | Frozen | 多层 Attention KV | ✅ | 多层控制 |
| LoRA | Frozen | 权重旁路 | ❌ | 当前最常用 |

---

# 22. 一分钟背诵

```text
第六章
= 工业级训练

Transformers
= 模型 + Tokenizer + Trainer

DeepSpeed
= 多卡 + 显存优化

ZeRO：
1 切 Optimizer
2 + Gradient
3 + Parameter

logging INFO：
DEBUG 不输出
INFO 及以上输出

PEFT
= 参数高效微调

LoRA
= 冻结 W0
+ 训练 BA

target_modules
= LoRA 插在哪里

经典：
q_proj + v_proj

更充分：
q/k/v/o
+ gate/up/down

gate_proj
= FFN 门控

up_proj
= hidden → intermediate

down_proj
= intermediate → hidden

SiLU
= x * sigmoid(x)

GELU
= x * GaussianCDF(x)

Prefix Tuning
= 冻结 LLM
+ 学习 Prefix
+ 注入每层 Attention K/V

Prompt Tuning
= 只加到第一层输入

Adapter
= Bottleneck MLP
```

最终一句话：

> **第六章真正需要掌握的是三层能力：Transformers/Trainer 让训练流程工程化，DeepSpeed/ZeRO 让大模型在有限显存上训得动，PEFT/LoRA/Prefix Tuning 让下游微调的参数和成本降下来。**

---

# 参考资料

1. Datawhale Happy-LLM，第六章：大模型训练流程实践  
   https://datawhalechina.github.io/happy-llm/

2. Hugging Face PEFT — LoRA  
   https://huggingface.co/docs/peft/package_reference/lora

3. Hugging Face PEFT — Prefix Tuning  
   https://huggingface.co/docs/peft/main/package_reference/prefix_tuning

4. PyTorch Activation Functions  
   https://docs.pytorch.org/docs/main/nn.html

> **说明**
>
> - `q_proj + v_proj`、`nn.Linear / nn.Embedding / nn.Conv2d` 等保留 Happy-LLM 第六章教程语境。
> - `target_modules="all-linear"`、`target_parameters` 属于当前 PEFT 工程实践补充。
> - LLaMA FFN 的 `gate_proj/up_proj/down_proj` 与 Bottleneck Adapter 的 down/up projection 不是同一种结构。
