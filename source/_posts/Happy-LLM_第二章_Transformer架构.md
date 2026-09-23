---
title: happy-llm第二章Transformer架构的阅读笔记
cover: /assets/posts/happy-llm/happy-llm-2.png 
categories: llm
tags:
  - happy-llm
---

# 面试速记

> 基于 Datawhale《Happy-LLM》第二章 **Transformer 架构**整理。重点面向大模型 / Agent / NLP 算法岗位面试，保留核心公式、实现机制和高频追问。

## 背诵版总结

Transformer 是一种**以 Attention 为核心、能够并行处理序列并建模长距离依赖**的神经网络架构。原始 Transformer 采用 **Encoder-Decoder** 结构，最初用于机器翻译等 Seq2Seq 任务。

完整流程可以记成：

```text
输入文本
  ↓
Tokenizer
  ↓
Token Embedding
  +
Positional Encoding
  ↓
Encoder × N
  ↓
Encoder Memory
  ↓
Decoder × N
  ↓
Linear
  ↓
Softmax
  ↓
下一个 Token / 输出序列
```

Transformer 最核心的几个组件：

1. **Self-Attention**：让序列中每个 token 与其他 token 建立直接关系。
2. **Scaled Dot-Product Attention**：

$$
Attention(Q,K,V) =
Softmax\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

3. **Multi-Head Attention（MHA）**：将注意力拆成多个头，不同 Head 学习不同语义关系，再拼接。
4. **Causal Mask / Masked Self-Attention**：Decoder 遮住未来 token，保证自回归生成时不能偷看答案。
5. **FFN**：对每个 token 的表示分别做非线性特征变换。
6. **Residual Connection**：保留原始信息，缓解深层网络退化和梯度传播问题。
7. **LayerNorm**：稳定每层输入分布，使深层 Transformer 更容易训练。
8. **Positional Encoding**：因为 Attention 本身不知道 token 顺序，需要额外加入位置信息。
9. **Encoder**：主要负责理解输入，使用**双向 Self-Attention**。
10. **Decoder**：负责自回归生成，包含**Masked Self-Attention + Cross-Attention + FFN**。

一句话背诵：

> **Transformer 用 Self-Attention 建模 token 间关系，用 Multi-Head Attention 学习多种关系，用位置编码补充顺序信息，用 Mask 保证自回归不看未来，再通过残差、LayerNorm 和 FFN 构建深层 Encoder-Decoder 网络。**

---

## 面试表达模板

### 1. 请介绍一下 Transformer

> Transformer 是一种完全基于 Attention 的序列建模架构。原始 Transformer 采用 Encoder-Decoder 结构。输入 token 首先经过 Embedding 和 Position Encoding，然后 Encoder 使用多头双向 Self-Attention 建模输入序列内部的依赖关系；Decoder 先通过 Masked Self-Attention 建模已经生成的内容，再通过 Cross-Attention 读取 Encoder 的输出，最后经过 FFN、Linear 和 Softmax 预测下一个 token。每个 Attention 和 FFN 子层外都有残差连接与 LayerNorm。相比 RNN，Transformer 可以在训练阶段并行处理整个序列，并且任意两个 token 可以直接通过 Attention 建立联系，因此更擅长建模长距离依赖。

### 2. Attention 是怎么计算的？

> Attention 有三个核心变量 Q、K、V。首先计算 Query 和 Key 的点积得到相关性分数，再除以 $\sqrt{d_k}$ 做缩放，然后经过 Softmax 得到注意力权重，最后对 Value 进行加权求和。

$$
Attention(Q,K,V) =
Softmax\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

直观上：

```text
Q：我要找什么
K：每个 token 提供什么索引信息
V：真正需要聚合的内容
```

### 3. Encoder 和 Decoder 有什么区别？

> Encoder 的核心是双向 Self-Attention，每个 token 可以看到整个输入序列，主要用于理解和编码；Decoder 的 Self-Attention 使用 Causal Mask，每个位置只能看到当前位置及之前的位置，同时 Decoder 还有 Cross-Attention，通过 Decoder hidden state 作为 Query、Encoder 输出作为 Key 和 Value，让生成过程能够读取输入信息。

### 4. 为什么 Transformer 比 RNN 更容易并行？

> RNN 的第 $t$ 个 hidden state 依赖第 $t-1$ 个 hidden state，因此必须按时间步顺序计算。Transformer 的 Self-Attention 可以直接用矩阵乘法一次计算序列中所有 token 两两之间的相关性，所以训练阶段能够对整个序列并行计算。Decoder 虽然推理阶段仍需要逐 token 自回归生成，但训练时可以利用 Causal Mask 一次并行计算所有位置。

---

## 高频面试问题

### Q1：Transformer 为什么要使用 Attention？

RNN 需要串行处理序列，而且两个距离很远的 token 之间的信息需要经过多个时间步传播。Attention 允许任意两个 token **直接计算相关性**，既有利于建模长距离依赖，也便于 GPU 并行计算。

### Q2：Q、K、V 分别是什么？

```text
Q：Query，我想找什么
K：Key，我这里有什么信息可以被匹配
V：Value，真正要返回、聚合的信息
```

通常由输入 $X$ 通过不同参数矩阵得到：

$$
Q=XW_Q,\quad K=XW_K,\quad V=XW_V
$$

### Q3：为什么 Attention 要除以 $\sqrt{d_k}$？

当 $d_k$ 较大时，Q 和 K 点积的幅度会变大，使 Softmax 更容易进入饱和区域，导致梯度变小、训练不稳定。

> **除以 $\sqrt{d_k}$ 是为了控制点积数值尺度，避免 Softmax 饱和，使训练更稳定。**

### Q4：什么是 Self-Attention？

Self-Attention 中 Q、K、V 都来自**同一个序列**，但使用不同可学习矩阵：

$$
Q=XW_Q,\quad K=XW_K,\quad V=XW_V
$$

这样每个 token 都能学习自己与序列中其他 token 的关系。

### Q5：Self-Attention 和 Cross-Attention 有什么区别？

| 类型 | Q 来源 | K/V 来源 | 作用 |
|---|---|---|---|
| Self-Attention | 当前序列 | 当前序列 | 建模序列内部关系 |
| Cross-Attention | Decoder | Encoder | 让 Decoder 读取输入信息 |

### Q6：什么是 Masked Self-Attention？

Masked Self-Attention 使用 **Causal Mask** 遮蔽未来位置，使第 $t$ 个 token 只能看到当前位置及之前的 token，不能看到 $t+1$ 之后的内容，从而保证自回归生成不会“偷看答案”。

### Q7：为什么用了 Mask 训练还能并行？

训练时整段序列一次输入，未来位置的 Attention Score 被加上 $-\infty$，经过 Softmax 后变成 0，因此**逻辑上看不到未来，但计算上所有位置可以同时进行**。

### Q8：什么是 Multi-Head Attention？

单个 Attention Head 只能在一个表示子空间中建模关系。MHA 使用多组不同的 Q/K/V 投影：

$$
head_i=Attention(QW_i^Q,KW_i^K,VW_i^V)
$$

$$
MultiHead=Concat(head_1,\dots,head_h)W^O
$$

不同 Head 可以学习语义、语法、指代、局部和长距离关系。

### Q9：为什么 Transformer 需要位置编码？

Self-Attention 本身不天然包含 token 顺序，因此需要：

$$
Input = TokenEmbedding + PositionalEncoding
$$

简单记：

```text
Token Embedding：告诉模型“我是谁”
Position Encoding：告诉模型“我在哪里”
```

### Q10：原始 Transformer 使用什么位置编码？

原始 Transformer 使用正余弦位置编码：

$$
PE(pos,2i)=\sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)
$$

$$
PE(pos,2i+1)=\cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)
$$

### Q11：FFN 是干什么的？

原始形式：

$$
FFN(x)=W_2ReLU(W_1x+b_1)+b_2
$$

简单理解：

```text
Attention：让不同 token 之间交换信息
FFN：对每个 token 的特征进一步非线性加工
```

### Q12：为什么需要残差连接？

$$
y=x+F(x)
$$

作用：保留输入信息、改善梯度传播、缓解深层网络退化，使深层 Transformer 更容易训练。

### Q13：为什么使用 LayerNorm 而不是 BatchNorm？

BatchNorm 依赖 batch 统计量，而 NLP 中 batch size、序列长度变化较大。LayerNorm 对单个 token 的 hidden dimension 做归一化，不依赖其他样本，因此更适合 Transformer。

### Q14：Pre-Norm 和 Post-Norm 有什么区别？

Post-Norm：

$$
y=LN(x+F(x))
$$

Pre-Norm：

$$
y=x+F(LN(x))
$$

Happy-LLM 第二章实现采用 **Pre-Norm**。现代 LLM 也常使用 Pre-Norm，因为深层模型训练通常更稳定。

### Q15：Encoder Layer 由什么组成？

```text
Input
  ↓
LayerNorm
  ↓
Multi-Head Self-Attention
  ↓
Residual Add
  ↓
LayerNorm
  ↓
FFN
  ↓
Residual Add
```

### Q16：Decoder Layer 由什么组成？

```text
Input
  ↓
LayerNorm
  ↓
Masked Multi-Head Self-Attention
  ↓
Residual
  ↓
LayerNorm
  ↓
Cross-Attention
  ↓
Residual
  ↓
LayerNorm
  ↓
FFN
  ↓
Residual
```

---

# 第二章 Transformer 架构详细总结

## 1. Transformer 为什么出现？

Transformer 出现之前，NLP 序列建模主要依赖 RNN / LSTM。RNN 主要有两个问题：

1. **难以并行计算**：第 $t$ 个 hidden state 依赖第 $t-1$ 个 hidden state。
2. **长距离依赖困难**：距离越远，信息需要经过越多时间步传播。

Transformer 使用 Attention 后，序列中任意 token 可以直接建立联系，并可通过矩阵运算并行计算。

---

# 2. Attention

Attention 的核心公式：

$$
Attention(Q,K,V) =
Softmax\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

计算步骤：

```text
输入 X
 ↓
线性映射得到 Q、K、V
 ↓
QKᵀ
 ↓
÷ √dk
 ↓
Softmax
 ↓
得到 Attention Weight
 ↓
× V
 ↓
输出
```

---

# 3. Self-Attention

Self-Attention：

```text
Q、K、V 都来自同一序列 X
```

但经过不同参数矩阵：

```text
X
├── WQ → Q
├── WK → K
└── WV → V
```

因此最终 token 表示是**上下文相关的动态表示**。

---

# 4. Causal Mask

Decoder 不能看到未来 token：

```text
        K1  K2  K3  K4
Q1      ✓   ×   ×   ×
Q2      ✓   ✓   ×   ×
Q3      ✓   ✓   ✓   ×
Q4      ✓   ✓   ✓   ✓
```

使用上三角 Mask：

```text
0   -∞  -∞  -∞
0    0  -∞  -∞
0    0   0  -∞
0    0   0   0
```

Softmax 后未来位置权重为 0。

---

# 5. Multi-Head Attention

假设：

```text
d_model = 512
num_heads = 8
```

则：

```text
head_dim = 64
```

结构：

```text
输入 X
  ↓
Head1 Head2 ... Head8
  ↓
Concat
  ↓
Linear Projection
  ↓
Output
```

MHA 的作用是从多个不同表示子空间学习关系。

---

# 6. Seq2Seq 与 Encoder-Decoder

Seq2Seq：

$$
(x_1,x_2,\dots,x_n)
\rightarrow
(y_1,y_2,\dots,y_m)
$$

原始 Transformer：

```text
Source Sequence
       ↓
    Encoder
       ↓
Encoder Memory
       ↓
    Decoder
       ↓
Target Sequence
```

---

# 7. Encoder

Encoder 负责：

> **将输入序列编码成包含上下文信息的表示。**

每层核心：

```text
Self-Attention
     +
    FFN
```

并配合 Residual 与 LayerNorm。

---

# 8. Decoder

Decoder 负责：

> **根据已生成 token 和 Encoder 输出，自回归生成目标序列。**

每层核心：

```text
Masked Self-Attention
        +
Cross-Attention
        +
       FFN
```

---

# 9. Cross-Attention

Cross-Attention 中：

$$
Q=H_{decoder}W_Q
$$

$$
K=H_{encoder}W_K
$$

$$
V=H_{encoder}W_V
$$

因此 Decoder 可以在每一步生成时读取 Encoder 的输入表示。

---

# 10. FFN、Residual、LayerNorm

## FFN

```text
d_model
 ↓
Linear
 ↓
Activation
 ↓
d_ff
 ↓
Linear
 ↓
d_model
```

## Residual

$$
output=x+F(x)
$$

## LayerNorm

对每个 token 的 hidden dimension 归一化，稳定数值尺度和训练过程。

---

# 11. Embedding 与 Position Encoding

Tokenizer：

```text
文本 → Token → Token ID
```

Embedding：

```text
Token ID → Dense Vector
```

Transformer 最终输入：

$$
X=TokenEmbedding+PositionalEncoding
$$

---

# 12. 完整 Transformer 数据流

## Encoder

```text
Source Text
 ↓
Tokenizer
 ↓
Embedding + Position
 ↓
Encoder Layer × N
 ↓
Encoder Output
```

每个 Encoder Layer：

```text
Self-Attention
 ↓
Residual + Norm
 ↓
FFN
 ↓
Residual + Norm
```

## Decoder

```text
Target Prefix
 ↓
Embedding + Position
 ↓
Decoder Layer × N
 ↓
Linear
 ↓
Softmax
 ↓
Next Token
```

每个 Decoder Layer：

```text
Masked Self-Attention
 ↓
Cross-Attention ← Encoder Output
 ↓
FFN
```

并在各子层配合 Residual 与 LayerNorm。

---

# 13. Transformer 组件速查表

| 组件 | 解决的问题 | 核心作用 |
|---|---|---|
| Embedding | Token ID 无语义 | Token → 向量 |
| Position Encoding | Attention 不知道顺序 | 注入位置信息 |
| Self-Attention | 建模上下文依赖 | Token 间信息交互 |
| Scaled Attention | 点积过大 | 稳定 Softmax |
| MHA | 单一 Attention 表示有限 | 多子空间建模 |
| Causal Mask | Decoder 偷看未来 | 保证自回归 |
| Cross-Attention | Decoder 需要读取输入 | 连接 Encoder 与 Decoder |
| FFN | Attention 后需非线性变换 | Token 内特征加工 |
| Residual | 深层网络难训练 | 信息和梯度直通 |
| LayerNorm | 各层数值分布变化 | 稳定训练 |
| Linear + Softmax | Hidden State 不是 token | 转成词表概率 |

---

# 14. Encoder 和 Decoder 对比

| 对比项 | Encoder | Decoder |
|---|---|---|
| 主要作用 | 理解 / 编码 | 生成 / 解码 |
| Self-Attention | 双向 | Causal |
| 能否看未来 Token | ✅ | ❌ |
| Cross-Attention | ❌ | ✅ 原始 Transformer |
| FFN | ✅ | ✅ |
| Residual | ✅ | ✅ |
| LayerNorm | ✅ | ✅ |
| 输入 | Source Tokens | Target Prefix |
| 输出 | Encoder Memory | Target Hidden States |

---

# 15. Transformer 与 RNN 对比

| 对比项 | RNN | Transformer |
|---|---|---|
| 训练计算 | 串行 | 高度并行 |
| 长距离依赖 | 较困难 | Attention 直接连接 |
| 顺序信息 | 天然包含 | 需要 Position |
| 主要计算 | 递归 | 矩阵乘法 / Attention |
| GPU 友好程度 | 较低 | 高 |
| 标准 Attention 长序列复杂度 | — | 通常 $O(n^2)$ |

注意：

> Transformer 的并行优势主要指**训练阶段**。自回归 Decoder 在推理时仍通常逐 token 生成。

---

# 16. 最后一页：一分钟背诵

```text
Transformer
= Embedding + Position
+ Attention
+ Encoder / Decoder
+ FFN
+ Residual
+ LayerNorm
```

Attention：

```text
Q：找什么
K：怎么匹配
V：返回什么

QKᵀ
 ↓
÷ √dk
 ↓
Softmax
 ↓
× V
```

Encoder：

```text
Self-Attention + FFN
```

Decoder：

```text
Masked Self-Attention
+ Cross-Attention
+ FFN
```

三个高频问题：

```text
为什么 ÷ √dk？
→ 防止点积过大导致 Softmax 饱和。

为什么需要 Position？
→ Attention 本身不知道顺序。

为什么需要 Causal Mask？
→ 防止看到未来答案，同时训练阶段仍可并行。
```

Transformer 最大优势：

```text
RNN：
串行 + 长距离依赖困难

Transformer：
并行训练 + Attention 直接建模长距离依赖
```
