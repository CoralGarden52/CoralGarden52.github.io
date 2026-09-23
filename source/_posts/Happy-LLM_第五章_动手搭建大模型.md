---
title: happy-llm第五章动手搭建大模型的阅读笔记
cover: /assets/posts/happy-llm/happy-llm-5.png 
categories: llm
tags:
  - happy-llm
---

# 面试速记

> 基于 Datawhale《Happy-LLM》第五章 **动手搭建大模型** 整理。本章的核心是：**手写 LLaMA2 → 训练 Tokenizer → 构造 Pretrain/SFT Dataset → 训练模型 → 自回归生成文本**。

## 背诵版总结

第五章可以记成三条主线：

```text
第一条：搭模型
ModelConfig
→ RMSNorm
→ GQA Attention
→ RoPE
→ Causal Mask / Flash Attention
→ SwiGLU 风格 MLP
→ DecoderLayer
→ Decoder-Only Transformer

第二条：造 Tokenizer
NFKC
→ ByteLevel
→ BPE
→ Special Tokens
→ tokenizer.json
→ tokenizer_config.json
→ special_tokens_map.json

第三条：训练与生成
文本 / 对话数据
→ Tokenizer
→ X / Y 右移一位
→ loss_mask
→ Pretrain
→ SFT
→ temperature + top-k
→ 自回归生成
```

最需要记住：

1. **LLaMA2 是 Decoder-Only Transformer**，第五章手写 RMSNorm、GQA、RoPE、门控 MLP、Decoder Layer 和完整 Transformer。
2. **RMSNorm** 不显式减均值，只根据均方根缩放隐藏状态。
3. **GQA** 减少 K/V Head 数，让多个 Q Head 共享 K/V，降低推理和 KV Cache 成本。
4. **RoPE** 对 Q、K 做位置相关旋转，把相对位置信息注入 Attention。
5. LLaMA 风格 MLP：

$$
FFN(x) =
W_2
\left(
SiLU(W_1x)
\odot
W_3x
\right)
$$

6. Tokenizer 最终采用 **BPE + ByteLevel**，并用 `NFKC` 做 Unicode 文本规范化。
7. `X=input_id[:-1]`、`Y=input_id[1:]` 的作用是构造 **Next Token Prediction**。
8. Pretrain 的 `loss_mask` 主要屏蔽 Padding；SFT 的 `loss_mask` 主要只保留 Assistant 回答区域。
9. `self.ctx` 是 **自动混合精度 AMP 的上下文管理器**。CPU 用 `nullcontext()`，GPU 用 `torch.amp.autocast(...)`。
10. `temperature` 控制采样随机性；`temperature=0` 时教程代码直接切换为 Greedy Decoding。
11. SFT 与 Pretrain 的训练循环基本相同，主要区别是 Dataset、loss mask，以及 SFT 会从预训练权重继续训练。
12. 文本生成循环：**最后位置 logits → temperature → top-k → Softmax → Sample → 拼回序列 → 继续生成**。

一句话背诵：

> **第五章从零完成一个小型 LLM 的完整闭环：先手写 LLaMA2，再训练 BPE Tokenizer，通过移位标签构造因果语言模型数据，完成 Pretrain 和 SFT，最后使用 temperature、top-k 做自回归生成。**

---

## 面试表达模板

### 1. 请介绍 Happy-LLM 第五章的主要内容

> 第五章把前面 Transformer 和 LLM 的理论落实到代码。首先使用 PyTorch 手写一个 LLaMA2 风格的 Decoder-Only Transformer，包括 RMSNorm、GQA、RoPE、Causal Self-Attention、门控 MLP 和 Decoder Layer；然后基于 Hugging Face Tokenizers 训练 BPE Tokenizer；接着将文本处理成因果语言模型需要的 X、Y 和 loss mask，完成 Pretrain 和 SFT；最后实现自回归生成，并通过 temperature 和 top-k 控制采样。

### 2. 从零训练一个小型 LLM 的流程是什么？

> 先准备预训练语料和 SFT 对话数据，再训练或加载 Tokenizer，把文本映射为 Token ID；然后构建 Decoder-Only Transformer。预训练阶段将序列右移一位构造 X 和 Y，通过 Cross Entropy 做 Next Token Prediction；SFT 阶段使用 Chat Template 格式化对话，并通过 loss mask 主要只计算 Assistant 回答部分的损失。训练时可以结合 AMP、梯度累积、梯度裁剪和学习率调度，最后通过自回归采样生成文本。

### 3. 第五章里的 LLaMA2 与原始 Transformer 有哪些关键区别？

> 它采用 Decoder-Only 架构，不再需要 Encoder 和 Cross-Attention；归一化使用 RMSNorm；位置信息使用 RoPE；Attention 支持 GQA；FFN 使用 SiLU 门控结构，也就是 LLaMA 常见的 SwiGLU 风格 FFN；Decoder Layer 使用 Pre-Norm 加残差连接。

### 4. Pretrain 与 SFT 在代码层面最大的区别是什么？

> 两者训练循环基本一致，都是 X 输入、Y 作为 Next Token Target，再计算 Cross Entropy。主要区别在 Dataset：Pretrain 通常对所有非 Padding 的文本 Token 计算损失；SFT 通过 Chat Template 区分 system、user、assistant，并通过 loss mask 主要只训练 Assistant 回答部分。另外 SFT 通常从 Base Model checkpoint 继续训练。

---

## 高频面试问题

### Q1：为什么要把 `input_id` 拆成 X 和 Y？

代码：

```python
X = input_id[:-1]
Y = input_id[1:]
```

例如：

```text
input_id = [BOS, 我, 喜欢, NLP, EOS]

X = [BOS, 我, 喜欢, NLP]
Y = [我, 喜欢, NLP, EOS]
```

这样每个位置都学习“根据前文预测下一个 Token”：

```text
BOS          → 我
BOS 我       → 喜欢
BOS 我 喜欢  → NLP
... NLP      → EOS
```

训练目标：

$$
\mathcal{L} =
-\sum_{t=1}^{T}
\log
P_{\theta}
\left(
x_t
\mid
x_{<t}
\right)
$$

作用：

> **把连续文本本身变成监督信号，一次前向传播即可并行计算所有位置的 Next Token Loss。**

---

### Q2：`loss_mask` 是做什么的？

`loss_mask` 决定哪些 Token 参与损失。

预训练：

```text
真实 Token → 1
Padding    → 0
```

SFT：

```text
System        → 0
User          → 0
Assistant回答 → 1
Padding       → 0
```

因此 SFT 主要学习：

> **给定 System、User 和历史对话，Assistant 应该生成什么。**

---

### Q3：`self.ctx` 是什么？

`self.ctx` 是一个 **自动混合精度（Automatic Mixed Precision, AMP）的上下文管理器**。

章节代码的核心逻辑：

```python
self.ctx = (
    nullcontext()
    if self.device_type == "cpu"
    else torch.amp.autocast(
        device_type=self.device_type,
        dtype=ptdtype
    )
)
```

使用：

```python
with self.ctx:
    output = model(...)
```

含义：

```text
CPU
→ nullcontext()
→ 正常执行

GPU
→ autocast()
→ 自动为不同算子选择适合的精度
```

AMP 的好处：

- 降低显存占用；
- 提高 Tensor Core 利用率；
- 加速矩阵计算；
- 对需要更高精度的算子自动采用更合适的精度。

需要区分：

```text
autocast
→ 控制前向计算精度

GradScaler
→ 主要避免 FP16 反向传播时的梯度下溢
```

---

### Q4：temperature 是什么？

Temperature 控制生成概率分布的尖锐程度。

$$
p_i =
\frac{
\exp(z_i/T)
}{
\sum_j \exp(z_j/T)
}
$$

其中：

```text
T = 1
→ logits 不变

0 < T < 1
→ 分布更尖锐
→ 更确定

T > 1
→ 分布更平坦
→ 更随机
```

教程对 `temperature=0` 单独判断：

```python
if temperature == 0.0:
    # greedy decoding
```

所以：

> **0.0 并不是拿 logits 去除以 0，而是直接切换成贪婪解码，每一步选择最大概率 Token。**

---

### Q5：Top-k 和 Temperature 有什么区别？

Temperature：

> 改变整个概率分布的形状。

Top-k：

> 只保留概率最高的 k 个候选 Token，其余候选直接排除。

流程：

```text
logits
↓
÷ temperature
↓
保留 top-k
↓
Softmax
↓
随机采样
```

---

### Q6：`tokenizer.json`、`tokenizer_config.json`、`special_tokens_map.json` 有什么区别？

一句话：

```text
tokenizer.json
→ Tokenizer 本体

tokenizer_config.json
→ Tokenizer 使用说明书

special_tokens_map.json
→ 特殊 Token 的角色映射
```

详细内容见后文。

---

### Q7：`NFKC()` 是做什么的？

`NFKC` 是 Unicode 的 **Compatibility Normalization + Canonical Composition**。**Canonical（规范等价）** 是指两个字符在语义上完全等同，只是编码方式或组合顺序不同（如 `é` 与 `e`+`´`），必须视为同一字符；而 **Compatibility（兼容等价）** 是指两个字符视觉或历史上有关联但语义并不严格等同（如全角 `Ａ` 与半角 `A`、上标 `²` 与数字 `2`、罗马数字 `Ⅳ` 与字母 `IV`），是为了兼容旧编码或特殊排版而存在的差异，全角/半角只是兼容等价中的一种典型情况。

简单说：

> **把视觉或兼容意义上相同/接近、但 Unicode 编码形式不同的字符统一到标准形式。**

例如：

```text
ＡＢＣ１２３
↓
ABC123
```

```text
① → 1
ﬀ → ff
```

这样可以减少“同一种文本因为编码形式不同而被切成不同 Token”的问题，提高词表利用率和数据一致性。

---

### Q8：为什么 LLaMA 使用 RMSNorm？

公式：

$$
RMSNorm(x) =
\frac{x}{
\sqrt{
\frac{1}{n}
\sum_{i=1}^{n}
x_i^2
+
\epsilon
}
}
\odot
\gamma
$$

它不显式减均值，计算更简单，同时仍能稳定隐藏状态的数值尺度。

---

### Q9：GQA 是怎么实现的？

例如：

```text
Q Heads = 16
KV Heads = 8
```

则每 2 个 Q Head 共享一组 K/V。

核心目的：

> **减少 K/V Head 数，在保留多个 Query Head 的同时降低显存和推理成本。**

章节通过 `repeat_kv()` 在 Attention 前把 K/V 扩展到与 Q Head 数量兼容。

---

### Q10：RoPE 在代码里作用在哪里？

流程：

```text
Hidden State
↓
生成 Q、K、V
↓
对 Q、K 应用 RoPE
↓
QKᵀ
↓
Attention
```

RoPE 主要旋转 Q 和 K，不旋转 V，因为位置关系主要需要影响 Attention Score。

---

### Q11：SFT 为什么主要只对 Assistant 回答计算 Loss？

因为 System 和 User 主要作为上下文条件，SFT 真正希望模型学习的是：

```text
面对这样的上下文
Assistant 应该回答什么
```

因此通过 loss mask 将 Assistant 回答区域设置为 1，其余区域通常设为 0。

---

### Q12：第五章的文本生成使用 KV Cache 吗？

没有。章节中的 `generate()` 是教学用的简单实现，每生成一个新 Token 都重新对当前上下文做 Forward。

因此：

```text
优点：逻辑简单
缺点：推理效率低
```

实际生产推理通常使用 KV Cache。

---

# 第五章：动手搭建大模型

## 1. 本章整体结构

```text
5.1 动手实现 LLaMA2
5.2 训练 Tokenizer
5.3 数据处理、Pretrain、SFT 与文本生成
```

总流程：

```text
原始文本
↓
训练 Tokenizer
↓
Token IDs
↓
PretrainDataset / SFTDataset
↓
X + Y + loss_mask
↓
LLaMA2-style Decoder-Only Transformer
↓
Cross Entropy
↓
Pretrain
↓
Base Model
↓
SFT
↓
Chat Model
↓
Temperature / Top-k
↓
生成文本
```

---

# 2. LLaMA2 风格模型

## 2.1 ModelConfig

章节用 `ModelConfig` 管理模型超参数：

| 参数 | 含义 |
|---|---|
| `dim` | Hidden Size |
| `n_layers` | Decoder Layer 数 |
| `n_heads` | Query Head 数 |
| `n_kv_heads` | K/V Head 数 |
| `vocab_size` | 词表大小 |
| `hidden_dim` | FFN Hidden Size |
| `norm_eps` | RMSNorm 的 epsilon |
| `max_seq_len` | 最大序列长度 |
| `dropout` | Dropout |
| `flash_attn` | 是否启用高效 Attention |

作用：

> **所有模块从同一份配置读取参数，便于统一管理模型规模。**

---

# 3. RMSNorm

$$
RMS(x) =
\sqrt{
\frac{1}{n}
\sum_{i=1}^{n}
x_i^2
+
\epsilon
}
$$

$$
RMSNorm(x) =
\frac{x}{RMS(x)}
\odot
\gamma
$$

特点：

- 不显式计算均值中心化；
- 使用均方根控制 Hidden State 的尺度；
- `gamma` 是可学习缩放参数；
- `epsilon` 防止除零。

---

# 4. GQA

设：

```text
n_heads = 16
n_kv_heads = 8
```

则：

$$
n_{rep} =
\frac{n_Q}{n_{KV}} =
2
$$

即：

```text
2 个 Q Head
共享 1 组 K/V
```

对比：

| 方法 | Q Head | KV Head |
|---|---:|---:|
| MHA | h | h |
| GQA | h | g |
| MQA | h | 1 |

GQA 是 MHA 与 MQA 的折中。

---

# 5. RoPE

流程：

```text
x
↓
Wq / Wk / Wv
↓
Q / K / V
↓
Q、K 应用 RoPE
↓
Attention
```

RoPE 通过位置相关旋转，将相对位置信息融入 Q/K 内积。

---

# 6. Causal Attention 与 Flash Attention

普通 Attention：

$$
Attention(Q,K,V) =
Softmax
\left(
\frac{QK^T}{\sqrt{d_k}}
+
Mask
\right)V
$$

Causal Mask：

```text
        K1  K2  K3  K4
Q1      ✓   ×   ×   ×
Q2      ✓   ✓   ×   ×
Q3      ✓   ✓   ✓   ×
Q4      ✓   ✓   ✓   ✓
```

未来位置被设为 `-inf`，Softmax 后权重为 0。

章节还检查 PyTorch 是否提供：

```python
torch.nn.functional.scaled_dot_product_attention
```

支持时使用高效 Attention 路径，否则回退到手写 Attention + Mask。

---

# 7. LLaMA MLP

代码核心：

```python
self.w2(
    F.silu(self.w1(x)) * self.w3(x)
)
```

可写为：

$$
h_1 =
SiLU(W_1x)
$$

$$
h_2 =
W_3x
$$

$$
FFN(x) =
W_2
\left(
h_1
\odot
h_2
\right)
$$

即：

$$
FFN(x) =
W_2
\left(
SiLU(W_1x)
\odot
W_3x
\right)
$$

这是 LLaMA 常见的 **SwiGLU 风格门控 FFN**。

---

# 8. Decoder Layer

章节实现 Pre-Norm：

$$
h =
x
+
Attention
\left(
RMSNorm(x)
\right)
$$

$$
out =
h
+
FFN
\left(
RMSNorm(h)
\right)
$$

结构：

```text
x
↓
RMSNorm
↓
GQA + RoPE + Causal Attention
↓
Residual
↓
RMSNorm
↓
SwiGLU-style MLP
↓
Residual
```

---

# 9. 完整 Decoder-Only Transformer

```text
Token IDs
↓
Token Embedding
↓
DecoderLayer × N
↓
RMSNorm
↓
LM Head
↓
Vocabulary Logits
```

训练：

```text
所有位置输出 logits
→ 和 Y 计算 Cross Entropy
```

推理：

```text
只取最后一个位置 logits
→ 预测 Next Token
```

---

# 10. Tokenizer 基础

Tokenizer：

```text
Text
↓
Token
↓
Token ID
```

章节介绍：

| 类型 | 特点 |
|---|---|
| Word-based | 直观，但 OOV 和超大词表问题明显 |
| Character-based | 几乎无 OOV，但序列过长 |
| Subword | 在词和字符之间折中，是现代 LLM 主流 |

Subword 常见算法：

| 方法 | 核心思想 |
|---|---|
| BPE | 反复合并最高频的字符/子词对 |
| WordPiece | 根据语言建模目标选择更有价值的子词 |
| Unigram | 基于概率模型选择最优子词组合 |

第五章采用：

> **BPE + ByteLevel**

---

# 11. BPE Tokenizer 训练流程

```text
JSONL 文本
↓
NFKC Normalizer
↓
ByteLevel Pre-Tokenizer
↓
BPE Trainer
↓
Special Tokens
↓
Vocab + Merge Rules
↓
tokenizer.json
↓
tokenizer_config.json
↓
special_tokens_map.json
```

特殊 Token：

```text
<unk>
<s>
</s>
<|im_start|>
<|im_end|>
```

示例 ID：

```text
<unk>        → 0
<s>          → 1
</s>         → 2
<|im_start|> → 3
<|im_end|>   → 4
```

---

# 12. `NFKC` 的作用

代码：

```python
from tokenizers.normalizers import NFKC

tokenizer.normalizer = NFKC()
```

NFKC：

> **Normalization Form Compatibility Composition**

可理解为：

```text
Compatibility Decomposition
↓
Canonical Composition
```

目的：

> **统一 Unicode 中兼容等价但编码不同的文本形式。**

示例：

```text
ＡＢＣ１２３ → ABC123
① → 1
ﬀ → ff
```

好处：

- 减少重复字符形式；
- 减少无意义的低频 Token；
- 提高词表利用率；
- 保证同类文本分词更一致。

注意：

> NFKC 会消除部分兼容性差异，因此对必须保留精确字形或排版语义的任务需要谨慎。

---

# 13. 三个 Tokenizer 文件

## 13.1 `tokenizer.json`

一句话：

> **Tokenizer 本体。**

第五章的 BPE Tokenizer 中，核心包括：

```text
Vocab
BPE Merge Rules
NFKC Normalizer
ByteLevel Pre-Tokenizer
BPE Model
ByteLevel Decoder
Added / Special Tokens
```

更准确地说，`tokenizer.json` 保存 Hugging Face Fast Tokenizer 底层的完整序列化分词 Pipeline，而不仅仅是词表。

---

## 13.2 `tokenizer_config.json`

一句话：

> **Tokenizer 的行为说明书。**

第五章包含：

```text
add_bos_token
add_eos_token
bos_token
eos_token
pad_token
unk_token
model_max_length
tokenizer_class
chat_template
...
```

作用：

> 告诉 Transformers 应该怎样加载和使用 Tokenizer。

其中 `chat_template` 非常重要，它将：

```python
{"role": "user", "content": "..."}
```

等结构化消息转换为：

```text
<|im_start|>user
...
<|im_end|>
```

这样的模型输入格式。

---

## 13.3 `special_tokens_map.json`

一句话：

> **特殊 Token 的角色映射表。**

例如：

```json
{
  "bos_token": "<|im_start|>",
  "eos_token": "<|im_end|>",
  "unk_token": "<unk>",
  "pad_token": "<|im_end|>",
  "additional_special_tokens": ["<s>", "</s>"]
}
```

它告诉 Transformers：

```text
哪个字符串是 BOS
哪个字符串是 EOS
哪个是 PAD
哪个是 UNK
```

---

## 13.4 三个文件对比

| 文件 | 存什么 | 作用 | 记忆 |
|---|---|---|---|
| `tokenizer.json` | Vocab、Merge Rules、Normalizer、PreTokenizer、Decoder 等 | 真正执行分词 | **怎么切** |
| `tokenizer_config.json` | 类、最大长度、Chat Template、BOS/EOS 行为等 | Transformers 使用方式 | **怎么用** |
| `special_tokens_map.json` | BOS/EOS/PAD/UNK 等角色映射 | 定义特殊 Token 身份 | **特殊 Token 是谁** |

最简单：

```text
tokenizer.json
= 怎么切

tokenizer_config.json
= 怎么用

special_tokens_map.json
= 特殊 Token 是谁
```

---

# 14. 数据预处理

预训练数据：

```json
{"text": "..."}
```

SFT 数据统一为：

```json
[
  {"role": "system", "content": "你是一个AI助手"},
  {"role": "user", "content": "..."},
  {"role": "assistant", "content": "..."}
]
```

SFT 再通过 `apply_chat_template()` 转换为模型真正读取的字符串。

---

# 15. 为什么 X/Y 要错一位？

原始：

```text
[BOS, T1, T2, T3, T4, EOS]
```

构造：

```text
X = [BOS, T1, T2, T3, T4]
Y = [T1, T2, T3, T4, EOS]
```

对应：

| X | Y |
|---|---|
| BOS | T1 |
| T1 | T2 |
| T2 | T3 |
| T3 | T4 |
| T4 | EOS |

目标：

$$
P
\left(
x_{t+1}
\mid
x_{\le t}
\right)
$$

序列概率：

$$
P(x_1,\dots,x_T) =
\prod_{t=1}^{T}
P
\left(
x_t
\mid
x_{<t}
\right)
$$

这就是 Decoder-Only LLM 的因果语言建模目标。

---

# 16. Pretrain loss_mask

Padding：

```text
[BOS, T1, T2, EOS, PAD, PAD]
```

Mask：

```text
[1, 1, 1, 1, 0, 0]
```

最终：

$$
Loss =
\frac{
\sum_i m_iL_i
}{
\sum_i m_i
}
$$

避免模型在 Padding 上浪费梯度。

---

# 17. SFT loss_mask

SFT 仍然使用相同的 X/Y Shift。

关键差异：

```text
System        → mask 0
User          → mask 0
Assistant回答 → mask 1
Padding       → mask 0
```

因此：

> **对话上下文负责“给条件”，Assistant span 负责“提供监督标签”。**

---

# 18. Pretrain 训练流程

训练循环包含：

```text
DataLoader
↓
动态学习率
↓
AMP Forward
↓
Masked Cross Entropy
↓
Gradient Accumulation
↓
Backward
↓
Gradient Clipping
↓
Optimizer Step
↓
Checkpoint
```

## Warmup

$$
LR_t =
LR_{max}
\frac{t}{T_{warmup}}
$$

## Cosine Decay

$$
coeff =
\frac{1}{2}
\left(
1
+
\cos(\pi r)
\right)
$$

---

# 19. AMP：`self.ctx` / `ctx`

`self.ctx` 是一个 **Automatic Mixed Precision 上下文管理器**。

```python
with self.ctx:
    ...
```

GPU 时进入 `autocast`，很多计算自动使用 BF16/FP16 等较低精度。

主要价值：

```text
更少显存
+
更高吞吐
```

训练时通常还配合：

```python
GradScaler
```

来提高 FP16 梯度计算稳定性。

---

# 20. Gradient Accumulation

如果显存只能放：

```text
batch_size = 8
```

但希望有效 Batch 接近：

```text
32
```

可以连续 4 个 Mini-batch 做 Backward，再 Step。

$$
EffectiveBatch =
BatchSize
\times
AccumulationSteps
$$

这样可以：

> **用更多计算时间换更低显存占用。**

---

# 21. Gradient Clipping

代码：

```python
torch.nn.utils.clip_grad_norm_(
    model.parameters(),
    grad_clip
)
```

作用：

> 限制梯度范数，防止梯度突然爆炸导致训练不稳定。

---

# 22. SFT 训练

SFT：

```text
Pretrained Base Model
↓
加载预训练 checkpoint
↓
SFTDataset
↓
Chat Template
↓
Assistant-only loss mask
↓
继续优化
↓
Chat / Instruction Model
```

训练循环与 Pretrain 基本相同。

---

# 23. 自回归生成

```text
Prompt
↓
Tokenizer
↓
Input IDs
↓
Forward
↓
最后位置 logits
↓
Temperature
↓
Top-k
↓
Softmax
↓
Sample
↓
Next Token
↓
拼回 Input
↓
重复
```

直到：

```text
EOS / stop_id
或
max_new_tokens
```

---

# 24. Temperature 实现机制

$$
z_i' =
\frac{z_i}{T}
$$

$$
P_i =
Softmax(z_i')
$$

| Temperature | 效果 |
|---|---|
| `0` | 教程代码走 Greedy |
| `< 1` | 更确定 |
| `1` | 不改变 logits |
| `> 1` | 更随机 |

---

# 25. Top-k

例如：

```text
vocab_size = 6144
top_k = 50
```

只保留最高的 50 个 logits，其余设置为：

```text
-inf
```

然后在剩余候选中采样。

作用：

> **避免从概率极低的长尾 Token 中采到明显不合理的结果。**

---

# 26. `torch.no_grad()` / `torch.inference_mode()`

推理不需要梯度：

```text
不需要 backward
不需要 optimizer
```

因此关闭 Autograd 可以：

- 节省显存；
- 减少计算图开销；
- 提升推理效率。

---

# 27. Pretrain、SFT、Inference 对比

| 阶段 | 输入 | Target | Loss Mask | Backward | 目的 |
|---|---|---|---|---|---|
| Pretrain | 普通文本 | 下一个 Token | 屏蔽 Padding | ✅ | 学语言和知识 |
| SFT | 对话上下文 | 下一个 Token | 主要保留 Assistant | ✅ | 学指令和对话 |
| Inference | Prompt + 已生成 Token | 无 | 无 | ❌ | 生成回答 |

---

# 28. 第五章关键组件速查

| 组件 | 作用 |
|---|---|
| ModelConfig | 管理模型超参数 |
| RMSNorm | 稳定 Hidden State 尺度 |
| GQA | 减少 K/V Head |
| RoPE | 向 Q/K 注入位置 |
| Causal Mask | 防止看到未来 |
| Flash Attention 路径 | 高效 Attention |
| SwiGLU 风格 MLP | 门控非线性变换 |
| BPE | Subword 分词 |
| NFKC | Unicode 规范化 |
| X/Y Shift | Next Token Target |
| loss_mask | 控制哪些 Token 计算 Loss |
| AMP | 降显存、提吞吐 |
| GradScaler | 混合精度数值稳定 |
| Gradient Accumulation | 模拟更大 Batch |
| Gradient Clipping | 防梯度爆炸 |
| Temperature | 控制随机性 |
| Top-k | 限制候选范围 |

---

# 29. 从文本到一次参数更新

```text
"北京是中国的首都"
↓
NFKC
↓
BPE Tokenizer
↓
[BOS, 103, 251, 88, 29, EOS]
↓
X / Y Shift

X = [BOS, 103, 251, 88, 29]
Y = [103, 251, 88, 29, EOS]

↓
Embedding
↓
Decoder Layers
↓
LM Head
↓
Vocabulary Logits
↓
Cross Entropy
↓
loss_mask
↓
Backward
↓
Gradient Accumulation
↓
Gradient Clipping
↓
Optimizer Step
```

---

# 30. 从 Prompt 到回答

```text
“中国的首都是哪里？”
↓
Chat Template
↓
Tokenizer
↓
Prompt IDs
↓
LLM
↓
最后位置 logits
↓
Temperature
↓
Top-k
↓
Sample next token
↓
拼回上下文
↓
再次 Forward
↓
...
↓
EOS
↓
decode
↓
“中国的首都是北京。”
```

---

# 31. 一分钟背诵

```text
第五章
= 从零搭建小 LLM

模型：
LLaMA2-style Decoder Only
= RMSNorm
+ GQA
+ RoPE
+ Causal Attention
+ SwiGLU-style MLP

Tokenizer：
NFKC
→ ByteLevel
→ BPE

三个文件：
tokenizer.json
= 怎么切

tokenizer_config.json
= 怎么用

special_tokens_map.json
= 特殊 Token 是谁

Dataset：
X = input_id[:-1]
Y = input_id[1:]

作用：
前缀 → 下一个 Token

Pretrain：
主要所有非 Padding Token 算 Loss

SFT：
主要 Assistant Token 算 Loss

AMP：
self.ctx / ctx
= autocast 上下文

生成：
logits
→ temperature
→ top-k
→ softmax
→ sampling

temperature：
0 = greedy
<1 = 更确定
1 = 原分布
>1 = 更随机
```

最终一句话：

> **第五章把 LLM 从概念变成可以运行的完整代码：模型结构决定如何计算，Tokenizer 决定文字如何变成 Token，X/Y Shift 决定学习目标，loss mask 决定哪些 Token 学，AMP 决定怎样更高效地训练和推理，temperature/top-k 决定生成时怎样选择下一个 Token。**
