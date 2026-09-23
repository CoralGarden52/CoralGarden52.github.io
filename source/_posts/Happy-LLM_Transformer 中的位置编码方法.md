---
title: Transformer中的位置编码方法
cover: /assets/posts/happy-llm/happy-llm-2.png 
categories: llm
tags:
  - happy-llm
---

# 面试速记

## 背诵版总结

Transformer 本身的 Self-Attention **不具备顺序感知能力**，因此必须额外加入位置信息。位置编码大体可以分为 **绝对位置编码** 和 **相对位置编码** 两类。

常见方法可以这样记：

```text
Sinusoidal
= 固定正弦余弦位置编码
= 不需要训练
= 原始 Transformer

Learned Position Embedding
= 每个位置对应一个可训练向量
= 简单直接
= BERT / GPT-2

Relative Position
= 不强调“我在第几个位置”
= 更强调“两个 Token 相距多远”
= Transformer-XL / T5

RoPE
= 旋转 Q 和 K
= 将相对位置自然编码进 Attention
= 当前 LLM 主流方法
= LLaMA / Qwen / Mistral

ALiBi
= 在 Attention Score 中加入距离惩罚
= 距离越远，分数越低
= 实现简单、外推较强
= BLOOM
```

一句话总结：

> **早期 Transformer 主要显式加入绝对位置，而现代 LLM 更倾向于把相对位置信息直接融入 Attention，其中 RoPE 是目前最常见的方法之一。**

---

## 面试表达模板

> Transformer 的 Self-Attention 本身并不能感知 Token 的顺序，因此需要额外加入位置编码。位置编码主要分为绝对位置和相对位置两类。原始 Transformer 使用固定的 Sinusoidal Positional Encoding，BERT 和 GPT-2 使用 Learned Position Embedding，这两种方法都显式表示 Token 的绝对位置。后来模型逐渐转向相对位置编码，因为自然语言中往往更关注两个 Token 之间的距离。T5 使用 Relative Position Bias，在 Attention Score 中加入相对位置偏置；现代大语言模型中更常见的是 RoPE，它通过旋转 Query 和 Key，把相对位置关系直接融入 Attention；另外还有 ALiBi，它通过在线性地惩罚远距离 Token 的 Attention Score 实现位置建模，具有较好的长度外推能力。

如果面试官追问“当前大模型最常见的是哪一种”，可以回答：

> **目前 Decoder-Only LLM 中 RoPE 非常常见。它不直接把 Position Embedding 加到输入上，而是对 Q 和 K 进行位置相关的旋转，使 Q 和 K 的内积天然包含相对位置差。**

---

## 高频面试问题

### Q1：为什么 Transformer 必须使用位置编码？

Self-Attention 主要根据 Token 内容之间的相似度计算注意力，它本身并不知道 Token 的先后顺序。

例如：

```text
我 喜欢 你
你 喜欢 我
```

如果没有位置信息，两句话包含完全相同的 Token，模型很难区分它们的顺序。

因此 Transformer 需要额外告诉模型：

```text
Token Embedding：
这个 Token 是什么

Position Information：
这个 Token 在什么位置
```

---

### Q2：绝对位置编码和相对位置编码有什么区别？

绝对位置关注：

```text
Token A 位于第 5 个位置
Token B 位于第 8 个位置
```

相对位置关注：

```text
Token B 比 Token A 晚 3 个位置
```

因此：

```text
绝对位置：
关注“我在哪里”

相对位置：
关注“我和其他 Token 相距多远”
```

现代 LLM 更常使用相对位置思想，因为 Attention 本质上关注的是 Token 之间的关系。

---

### Q3：Sinusoidal Position Encoding 为什么使用正弦和余弦？

因为不同频率的正弦余弦函数可以为不同位置生成唯一、连续的位置表示，而且不需要额外学习参数。

另外，正弦余弦函数还具有一定的相对位置性质，因此模型可以从不同位置之间的函数关系中推断距离信息。

---

### Q4：Learned Position Embedding 的主要缺点是什么？

最大问题是：

> **长度外推能力较差。**

例如模型训练时只学习：

```text
Position 0 ~ 511
```

那么对于：

```text
Position 512
Position 513
```

模型并没有训练过对应的位置向量。

所以当输入长度明显超过训练长度时，Learned Position Embedding 通常比较受限。

---

### Q5：RoPE 为什么能表示相对位置？

RoPE 会根据 Token 的位置，对 Query 和 Key 进行不同角度的旋转。

假设位置分别是 $m$ 和 $n$，旋转后的 Q 和 K 计算内积时，会出现：

$$
R_m^T R_n =
R_{n-m}
$$

因此最终的 Attention 只和：

$$
n-m
$$

有关。

也就是两个 Token 之间的**相对位置差**。

所以可以记：

> **RoPE 通过旋转 Q 和 K，让 Attention 内积天然包含相对位置信息。**

---

### Q6：RoPE 为什么只作用于 Q 和 K？

因为 Attention Score 是通过 Q 和 K 的内积计算的：

$$
Score =
QK^T
$$

位置关系真正影响的是：

> “当前位置应该对哪个位置给予多少注意力”。

所以将位置旋转加入 Q 和 K，就可以直接影响 Attention Score。

而 V 主要表示真正被聚合的信息，因此通常不需要进行位置旋转。

---

### Q7：ALiBi 和 RoPE 有什么区别？

RoPE：

```text
对 Q 和 K 做位置相关旋转
↓
通过内积体现相对位置
```

ALiBi：

```text
直接在 Attention Score 中
加入与距离相关的负偏置
```

简单记：

```text
RoPE：
改 Q/K

ALiBi：
改 Attention Score
```

---

### Q8：哪种位置编码长度外推最好？

一般来说：

```text
Learned Position
→ 较差

Sinusoidal
→ 一般到较好

Relative Position
→ 较好

RoPE
→ 较好，可配合 RoPE Scaling 进一步扩展

ALiBi
→ 通常具有较好的长度外推能力
```

但实际效果还取决于训练长度、数据分布和具体模型设计。

---

### Q9：为什么现代 LLM 更喜欢 RoPE？

主要因为 RoPE：

- 可以自然建模相对位置；
- 不需要传统的位置 Embedding Table；
- 额外参数很少；
- 实现和 Attention 高度结合；
- 对 Decoder-Only 架构非常适合；
- 可以配合 Scaling 方法扩展上下文长度；
- 实践中效果稳定。

因此 LLaMA、Qwen、Mistral 等大量现代 LLM 都采用了 RoPE 或其变体。

---

# Transformer 中的位置编码方法

## 1. 为什么需要位置编码？

Transformer 的 Self-Attention 本身主要通过 Token 之间的内容相关性进行计算，它并不会天然感知序列中 Token 的先后顺序。

例如：

```text
我 喜欢 你

你 喜欢 我
```

两句话包含完全相同的 Token，但顺序不同，语义也完全不同。

因此 Transformer 需要额外加入：

> **Positional Information**

使模型同时知道：

```text
Token Embedding：
这个 Token 是什么

Position Information：
这个 Token 在哪里
```

常见位置编码方法可以分成：

```text
绝对位置编码
├── Sinusoidal Position Encoding
└── Learned Position Embedding

相对位置编码
├── Relative Position Encoding
├── T5 Relative Position Bias
├── RoPE
└── ALiBi
```

---

# 2. 几种位置编码方法对比

| 方法 | 类型 | 核心思想 | 是否需要训练参数 | 长度外推能力 | 位置信息加入位置 | 代表模型 |
|---|---|---|---|---|---|---|
| Sinusoidal | 绝对位置 | 用正弦、余弦函数表示位置 | ❌ | 较好 | Embedding | 原始 Transformer |
| Learned Position Embedding | 绝对位置 | 为每个位置学习一个向量 | ✅ | 较差 | Embedding | BERT、GPT-2 |
| Relative Position Encoding | 相对位置 | 建模 Token 之间的相对距离 | 通常 ✅ | 较好 | Attention | Transformer-XL |
| T5 Relative Position Bias | 相对位置 | 在 Attention Score 中加入位置 Bias | ✅ | 较好 | Attention Score | T5 |
| RoPE | 相对位置思想 | 旋转 Q、K 注入位置信息 | ❌ | 较好 | Q / K | LLaMA、Qwen、Mistral |
| ALiBi | 相对位置偏置 | 距离越远，Attention Score 惩罚越大 | ❌ | 很好 | Attention Score | BLOOM |

可以简单理解为：

```text
Sinusoidal：
“我是第 5 个 Token”

Learned Position：
“模型自己学习第 5 个位置应该是什么表示”

Relative Position：
“我和另一个 Token 相距 3”

RoPE：
“把位置差编码进 Q、K 的旋转关系”

ALiBi：
“距离越远，Attention Score 越低”
```

---

# 3. Sinusoidal Positional Encoding

## 3.1 基本思想

Sinusoidal Positional Encoding 是原始 Transformer 使用的位置编码。

它使用不同频率的正弦和余弦函数来描述 Token 的位置。

偶数维：

$$
PE(pos,2i) =
\sin
\left(
\frac{pos}
{10000^{2i/d_{model}}}
\right)
$$

奇数维：

$$
PE(pos,2i+1) =
\cos
\left(
\frac{pos}
{10000^{2i/d_{model}}}
\right)
$$

其中：

- $pos$：Token 所在位置；
- $i$：位置向量的维度；
- $d_{model}$：模型隐藏维度。

最终输入为：

$$
X =
TokenEmbedding
+
PositionEncoding
$$

---

## 3.2 实现机制

```text
Token Embedding
      +
Sinusoidal Position Encoding
      ↓
Transformer
```

例如：

```text
Position 0 → PE0
Position 1 → PE1
Position 2 → PE2
...
```

每个位置都有一个唯一的正余弦向量。

---

## 3.3 优点

- 不需要训练参数；
- 实现简单；
- 可以直接计算任意位置；
- 理论上可以处理训练长度之外的位置；
- 不同频率能够描述不同尺度的位置变化。

## 3.4 缺点

- 位置表示是人工设计的；
- 不能根据数据自动学习；
- 更偏向绝对位置；
- 当前主流 Decoder-Only LLM 中已经不再常用。

## 3.5 代表模型

```text
Original Transformer
```

---

# 4. Learned Positional Embedding

## 4.1 基本思想

Learned Position Embedding 的思路非常直接：

> **每一个位置都对应一个可训练的位置向量。**

例如最大序列长度为 512：

```text
Position 0   → PositionEmbedding[0]
Position 1   → PositionEmbedding[1]
...
Position 511 → PositionEmbedding[511]
```

本质上与 Token Embedding 类似。

---

## 4.2 实现机制

定义一个位置 Embedding 矩阵：

$$
P
\in
\mathbb{R}^{L_{max}\times d_{model}}
$$

其中：

- $L_{max}$：最大序列长度；
- $d_{model}$：隐藏维度。

第 $i$ 个 Token：

$$
X_i =
TokenEmbedding(x_i)
+
PositionEmbedding(i)
$$

例如：

```text
Token “我”
=
TokenEmbedding(我)
+
PositionEmbedding(0)
```

---

## 4.3 优点

- 位置表示可以自动学习；
- 实现简单；
- 在固定上下文长度下效果通常较好；
- 不需要人为设计位置函数。

## 4.4 缺点

最大问题：

> **长度外推能力较差。**

例如：

```text
训练最大长度：
512
```

模型只学过：

```text
Position 0 ~ 511
```

超出这个长度的位置并没有经过训练。

因此直接扩展到更长上下文比较困难。

## 4.5 代表模型

```text
BERT
GPT-2
```

---

# 5. Relative Positional Encoding

## 5.1 基本思想

Relative Position Encoding 不再重点关心：

```text
这个 Token 是第几个
```

而是更加关注：

```text
这个 Token 和另一个 Token 相距多少
```

假设：

```text
Token A：Position 5
Token B：Position 8
```

则相对距离为：

$$
Relative(i,j) =
j-i
$$

所以：

$$
8-5=3
$$

模型需要知道的是：

> Token B 在 Token A 后面 3 个位置。

---

## 5.2 实现机制

一种典型方法是在 Attention Score 中加入相对位置项：

$$
AttentionScore_{ij} =
\frac{Q_iK_j^T}
{\sqrt{d_k}}
+
R_{i-j}
$$

其中：

$$
R_{i-j}
$$

表示 Token $i$ 与 Token $j$ 的相对位置关系。

---

## 5.3 优点

- 直接表示 Token 间距离；
- 更符合自然语言关系建模；
- 对局部依赖和长距离依赖更加自然；
- 长度泛化能力通常优于 Learned Position Embedding。

## 5.4 缺点

- 实现比绝对位置编码更复杂；
- 可能增加额外参数和计算；
- 不同模型实现方式差异较大。

## 5.5 代表模型

```text
Transformer-XL
T5
```

---

# 6. T5 Relative Position Bias

T5 使用的是一种比较轻量的 Relative Position 方法。

它不会直接把位置向量加入 Token Embedding，而是：

> **在 Attention Score 中加入一个位置相关的 Bias。**

公式：

$$
Score_{ij} =
\frac{Q_iK_j^T}
{\sqrt{d_k}}
+
b_{Relative(i,j)}
$$

其中：

$$
b_{Relative(i,j)}
$$

是一个可学习的位置偏置。

---

## 6.1 Bucket 机制

T5 并不会为所有距离都学习不同参数。

它会把不同距离划分到多个 Bucket：

```text
距离 1
→ Bucket 1

距离 2
→ Bucket 2

距离 3~4
→ Bucket 3

距离 5~8
→ Bucket 4

更远距离
→ 更大的 Bucket
```

核心思想：

> **近距离位置关系需要更精细，远距离位置关系可以更粗略。**

因为：

```text
距离 1 和距离 2
通常差异比较重要

距离 100 和距离 101
通常差异没那么重要
```

---

# 7. RoPE：Rotary Position Embedding

## 7.1 基本思想

RoPE 全称：

> **Rotary Position Embedding，旋转位置编码。**

它是当前 Decoder-Only LLM 中非常常见的位置编码方法。

传统方法：

```text
Token Embedding
+
Position Embedding
```

而 RoPE：

```text
Q → 位置旋转
K → 位置旋转
```

也就是说：

> **RoPE 不直接修改 Token Embedding，而是把位置信息加入 Attention 的 Q 和 K。**

---

## 7.2 旋转矩阵

二维旋转矩阵为：

$$
R_{\theta} =
\begin{bmatrix}
\cos\theta & -\sin\theta \\
\sin\theta & \cos\theta
\end{bmatrix}
$$

RoPE 会将 Q 和 K 的维度两两组合，并根据当前位置应用不同角度的旋转。

例如：

```text
Position 1
→ 旋转 θ

Position 2
→ 旋转 2θ

Position 3
→ 旋转 3θ
```

对于位置 $m$：

$$
Q_m' =
R_mQ_m
$$

对于位置 $n$：

$$
K_n' =
R_nK_n
$$

---

## 7.3 为什么 RoPE 可以表示相对位置？

计算旋转后的内积：

$$
(Q_m')^T K_n'
$$

代入后：

$$
(R_mQ_m)^T(R_nK_n)
$$

得到：

$$
Q_m^T
R_m^T
R_n
K_n
$$

而：

$$
R_m^T R_n =
R_{n-m}
$$

因此最终 Attention 与：

$$
n-m
$$

有关。

也就是：

> **两个 Token 的相对位置差。**

这是 RoPE 最核心的原理。

---

## 7.4 实现流程

```text
Hidden State
      ↓
生成 Q、K、V
      ↓
对 Q、K 应用 RoPE
      ↓
计算 QKᵀ
      ↓
Attention
```

注意：

> **RoPE 通常只作用于 Q 和 K，不作用于 V。**

---

## 7.5 为什么不旋转 V？

因为 Attention Score 来自：

$$
QK^T
$$

位置信息需要影响的是：

```text
当前 Token
应该关注哪个 Token
```

所以只需要通过 Q 和 K 改变：

> Attention 权重。

而 V 主要承担：

> 真正需要被加权汇聚的信息。

因此通常不需要进行旋转。

---

## 7.6 优点

- 自然表示相对位置；
- 不需要传统 Learned Position Embedding；
- 与 Attention 结构结合紧密；
- 参数开销小；
- 计算开销较低；
- 长上下文效果较好；
- 可以配合 RoPE Scaling 扩展上下文长度。

## 7.7 缺点

- 超出训练长度过多时仍可能退化；
- 长上下文通常还需要额外 Scaling；
- 相比传统 Position Embedding，实现稍复杂。

## 7.8 代表模型

```text
LLaMA
Qwen
Mistral
以及大量现代 Decoder-Only LLM
```

---

# 8. ALiBi

## 8.1 基本思想

ALiBi 全称：

> **Attention with Linear Biases**

它甚至不需要显式的位置 Embedding。

核心思想：

> **两个 Token 距离越远，就给它们的 Attention Score 越大的负偏置。**

---

## 8.2 实现机制

普通 Attention Score：

$$
Score_{ij} =
\frac{Q_iK_j^T}
{\sqrt{d_k}}
$$

ALiBi：

$$
Score_{ij} =
\frac{Q_iK_j^T}
{\sqrt{d_k}} -
m|i-j|
$$

其中：

- $|i-j|$：两个 Token 的位置距离；
- $m$：该 Attention Head 对应的斜率。

---

## 8.3 示例

假设：

$$
m=0.1
$$

那么：

```text
距离 1
→ -0.1

距离 5
→ -0.5

距离 10
→ -1.0
```

随着距离增加：

```text
负偏置越来越大
      ↓
Attention Score 越来越低
```

因此模型天然更倾向于关注较近的 Token。

---

## 8.4 优点

- 实现非常简单；
- 不需要 Position Embedding；
- 几乎不增加参数；
- 长度外推能力较好；
- 可以比较自然地扩展到更长序列。

## 8.5 缺点

- 位置表达形式较简单；
- 主要使用线性距离惩罚；
- 灵活性不如 RoPE；
- 当前主流 LLM 使用范围低于 RoPE。

## 8.6 代表模型

```text
BLOOM
```

---

# 9. 几种位置编码到底加在哪里？

这是非常适合面试记忆的一个对比。

## Sinusoidal

```text
Token Embedding
      +
Position Encoding
```

## Learned Position

```text
Token Embedding
      +
Learned Position Embedding
```

## Relative Position

```text
Attention
+
Relative Position Information
```

## T5 Relative Position Bias

```text
Attention Score
      +
Position Bias
```

## RoPE

```text
Q / K
 ↓
Position Rotation
 ↓
Attention
```

## ALiBi

```text
Attention Score
      +
Linear Distance Bias
```

---

# 10. 完整对比表

| 对比项 | Sinusoidal | Learned Position | Relative Position | RoPE | ALiBi |
|---|---|---|---|---|---|
| 类型 | 绝对 | 绝对 | 相对 | 相对位置思想 | 相对 |
| 是否学习参数 | ❌ | ✅ | 通常 ✅ | 通常 ❌ | ❌ |
| 加入位置 | Embedding | Embedding | Attention | Q / K | Attention Score |
| 修改 Token Embedding | ✅ | ✅ | 通常 ❌ | ❌ | ❌ |
| 能否表示相对距离 | 间接 | 较弱 | ✅ | **✅** | ✅ |
| 长度外推 | 一般/较好 | 较差 | 较好 | **较好** | **很好** |
| 参数量 | 0 | 有 | 通常有 | 极少 / 0 | 0 |
| 实现复杂度 | 低 | 最低 | 中 | 中 | 低 |
| 长上下文能力 | 一般 | 较差 | 较好 | **很好** | **很好** |
| 当前 LLM 常见程度 | 较少 | 较少 | 部分 | **非常常见** | 部分 |
| 代表模型 | Transformer | BERT / GPT-2 | Transformer-XL / T5 | LLaMA / Qwen / Mistral | BLOOM |

---

# 11. 最后总结

位置编码的发展趋势可以理解为：

```text
最早：

直接告诉模型
“这个 Token 在第几个位置”

Sinusoidal
Learned Position
      ↓

后来：

告诉模型
“两个 Token 之间是什么位置关系”

Relative Position
      ↓

现代 LLM：

直接把位置关系融入 Attention

RoPE
ALiBi
```

其中最值得重点掌握的是：

```text
Sinusoidal
→ 原始 Transformer

Learned Position
→ BERT / GPT-2

T5 Relative Bias
→ Attention Score 加位置 Bias

RoPE
→ 旋转 Q/K
→ 当前大模型高频

ALiBi
→ Attention Score 加距离惩罚
→ 外推能力强
```

最终一句话：

> **Transformer 的位置编码从显式绝对位置逐渐发展到直接建模相对位置关系；现代大语言模型更倾向于把位置信息直接融入 Attention，其中 RoPE 是当前非常主流的位置编码方案。**
