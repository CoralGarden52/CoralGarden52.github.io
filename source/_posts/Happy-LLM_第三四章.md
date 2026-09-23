---
title: happy-llm第三四章预训练语言模型和大语言模型的阅读笔记
cover: /assets/posts/happy-llm/happy-llm-3-4.png 
categories: llm
tags:
  - happy-llm
---

# 面试速记

> 基于 Datawhale《Happy-LLM》第三章“预训练语言模型”和第四章“大语言模型”整理，并补充面试常问的 DPO、GRPO、MHA/MQA/GQA。
>
> 核心主线：**Transformer → PLM 三种架构 → LLM → Pretrain → SFT → Preference Alignment（RLHF / DPO / GRPO）**。

## 背诵版总结

1. **Encoder-Only**：只保留 Transformer Encoder，使用**双向自注意力**，每个 token 都能看左右两侧上下文，擅长**理解类任务 NLU**。代表模型是 **BERT / RoBERTa / ALBERT**。经典预训练任务是 **MLM**，BERT 还使用 **NSP**。
2. **Encoder-Decoder**：Encoder 双向理解输入，Decoder 使用**因果自注意力**生成输出，并通过 **Cross-Attention**读取 Encoder 表示，天然适合 **Seq2Seq**。代表模型是 **T5**。
3. **Decoder-Only**：只保留 Decoder 的核心结构，使用 **Causal Mask**，每个 token 只能看到自己及之前的 token，通过 **CLM / Next Token Prediction** 自回归生成。代表模型是 **GPT、LLaMA、Qwen 等主流 LLM**。
4. **Pretrain**：用海量无标注文本训练“下一个 token 预测”等自监督任务，主要解决“**让模型学知识和语言规律**”。
5. **SFT**：用高质量“指令—回答”数据做监督微调，主要解决“**让模型学会听懂指令并按格式回答**”。
6. **RLHF**：先用人类偏好数据训练 **Reward Model（RM）**，再用 PPO 等强化学习算法优化策略模型，使输出更符合人类偏好；经典 PPO-RLHF 一般需要 **Actor、Reference、Reward、Critic**。
7. **DPO**：直接用 `(prompt, chosen, rejected)` 偏好对优化语言模型，**不显式训练 Reward Model，也不需要 Critic，不需要 PPO 在线 RL rollout**，训练简单稳定。
8. **GRPO**：一种 PPO 变体。对同一个 prompt 一次采样多条回答，用**组内相对奖励**估计 advantage，**去掉 Critic / Value Model**，仍属于在线策略优化。常见于数学、代码等可验证任务。
9. **MHA**：每个 Query Head 都有自己的 K/V Head，表达能力强但 KV Cache 大。
10. **MQA**：多个 Query Head **共享一套 K/V**，KV Cache 最小、推理最快，但可能损失部分表达能力。
11. **GQA**：将 Query Head 分组，每组共享一套 K/V，是 MHA 和 MQA 的折中；当前大模型中非常常见。

一句话串起来：

> **Pretrain 让模型“有知识”，SFT 让模型“会听话”，Preference Alignment 让模型“更符合人类偏好”；Decoder-Only + CLM 是当前主流 LLM 的基本路线。**

---

## 面试表达模板

### 1. 三种 PLM 架构怎么回答

> Transformer 时代的预训练语言模型主要分为 Encoder-Only、Encoder-Decoder 和 Decoder-Only。Encoder-Only 采用双向 Self-Attention，更擅长文本理解，比如 BERT；Encoder-Decoder 将输入理解和输出生成拆开，Decoder 通过 Cross-Attention 读取 Encoder 表示，适合翻译、摘要等 Seq2Seq 任务，比如 T5；Decoder-Only 使用 Causal Mask 做自回归的 Next Token Prediction，训练目标和推理形式一致，扩展到大规模数据和参数后形成了 GPT、LLaMA 等主流 LLM。

### 2. Pretrain、SFT、RLHF 怎么回答

> Pretrain 使用海量无标注数据做自监督学习，核心目标是学习语言规律和世界知识；SFT 使用人工或模型构造的高质量指令数据，通过交叉熵训练模型按照指令输出目标回答，获得指令遵循和对话能力；RLHF 再利用人类偏好排序训练奖励模型，并用 PPO 等强化学习算法提高高奖励回答的概率，同时通过 KL 约束避免策略偏离参考模型太远。

### 3. DPO 和 RLHF 怎么回答

> 经典 PPO-RLHF 要先训练 Reward Model，再进行在线 PPO 强化学习，还需要 Critic 做 advantage 估计。DPO 将这个偏好优化问题推导成一个直接的二分类式目标，只需要 chosen/rejected 偏好对和一个 reference policy，就能直接提高 chosen 相对 rejected 的概率，因此没有显式 Reward Model、没有 Critic，也不需要 PPO rollout，工程上更简单。

### 4. GRPO 和 PPO 怎么回答

> GRPO 可以理解为不使用 Critic 的 PPO 类方法。它对同一个 prompt 采样一组回答，分别计算 reward，再用组内 reward 的均值和标准差得到相对 advantage。高于组平均的回答被增强，低于组平均的回答被抑制，因此省掉了通常与策略模型同量级的 Value/Critic 模型，特别适合数学、代码等能够自动判分的任务。

### 5. MHA、MQA、GQA 怎么回答

> MHA 中每个 Query Head 都有独立的 K/V Head；MQA 保留多个 Query Head，但所有 Query Head 共享同一组 K/V，大幅减少 KV Cache；GQA 则把 Query Head 分组，每组共享一组 K/V，是表达能力和推理效率的折中。假设有 32 个 Q Head，MHA 可以有 32 个 KV Head，MQA 只有 1 个 KV Head，GQA 例如使用 8 个 KV Head。

---

## 高频面试问题

### Q1：为什么 BERT 更适合 NLU，而 GPT 更适合 NLG？

BERT 使用 Encoder 的**双向注意力**，一个 token 可以同时利用左、右上下文，因此适合分类、NER、匹配、抽取等理解任务。GPT 使用**因果注意力**，每个位置只能看历史 token，天然和“根据上文生成下一个 token”的过程一致，因此更适合文本生成。

### Q2：Decoder-Only 为什么成为 LLM 主流？

核心原因包括：

- **训练目标与生成方式一致**：训练和推理都是 Next Token Prediction。
- **架构简单统一**：理解、问答、翻译、摘要、代码等都能转化为“上下文 → 继续生成”。
- **容易做规模扩展**：数据、参数、算力增加时能力持续增强。
- **天然支持 In-Context Learning**：任务描述和示例都可以放进上下文中。
- **自回归生成部署成熟**：可结合 KV Cache、MQA/GQA 等优化推理。

### Q3：BERT 的 MLM 是怎么做的？

随机选约 15% token 作为预测位置。经典 BERT 中，对这些位置：

- 80% 替换为 `[MASK]`
- 10% 替换为随机 token
- 10% 保持原 token

模型利用左右上下文预测原 token。

这样设计是为了既让模型学会**利用上下文恢复词语**，又减少预训练和下游任务之间的差异。80% 使用 `[MASK]` 提供明确的预测目标；10% 保持原词，可以避免模型只适应 `[MASK]` 这种实际任务中不存在的符号；10% 随机替换则防止模型直接复制当前位置的词，迫使它结合左右上下文判断当前词是否合理，从而提升模型的语义理解、鲁棒性和泛化能力。

### Q4：NSP 是什么？为什么 RoBERTa 去掉它？

NSP（Next Sentence Prediction）判断两个句子是否为连续上下文，用来学习句间关系。RoBERTa 的实验认为 NSP 太容易、贡献有限，因此去掉 NSP，只保留并改进 MLM。

### Q5：SOP 和 NSP 的区别？

NSP 的负样本通常来自**随机不连续句子**；SOP（Sentence Order Prediction）的负样本是把真实连续句子的**顺序交换**。因此 SOP 更关注句子顺序和篇章连贯性，任务更难。

### Q6：T5 和 BERT 的“Mask”一样吗？

不完全一样。教程为了入门把 T5 概括为 MLM，但经典 T5 更准确地说使用 **Span Corruption / Text Infilling**：随机删除连续 token span，用 `<extra_id_n>` 等 sentinel token 代替；Encoder 看损坏后的文本，Decoder 生成被删除的 span。

### Q7：RLHF 一定等于 PPO 吗？

不是。**RLHF 是“利用人类反馈进行对齐”的框架/范式，PPO 只是其中一种优化算法。**  
经典 InstructGPT 式 RLHF 常使用 Reward Model + PPO，但也可以使用其他强化学习方法。

### Q8：DPO 属于强化学习吗？

通常将 DPO 称为 **RL-free preference optimization**。它解决的是和 RLHF 相同的偏好对齐问题，但训练时直接优化偏好损失，不显式训练奖励模型，也不执行 PPO 式在线强化学习。

### Q9：GRPO 需要 Reward Model 吗？

**不一定。**GRPO 需要的是“reward 信号”，这个 reward 可以来自：

- 训练好的 Reward Model；
- 数学答案正确性；
- 单元测试/代码执行结果；
- 格式检查器；
- 规则或 verifier。

因此在可验证任务里可以不使用人类偏好 RM。

### Q10：GRPO 和 DPO 最大区别是什么？

- **DPO**：通常是**离线偏好优化**，使用固定 `(chosen, rejected)` 数据。
- **GRPO**：是**在线 RL**，模型当前策略对同一 prompt 生成多条 response，在线打分后更新策略。
- DPO 不需要 reward function；GRPO 必须有某种 reward 信号。
- 二者都可以省掉经典 PPO-RLHF 中的一部分复杂组件，但省掉的方式不同。

---

# 第三章：预训练语言模型（PLM）

## 1. 第三章主线

第三章从 Transformer 的 Encoder 和 Decoder 出发，将 PLM 分为三条路线：

| 架构 | 代表模型 | 注意力方式 | 典型预训练目标 | 强项 |
|---|---|---|---|---|
| Encoder-Only | BERT、RoBERTa、ALBERT | 双向 Self-Attention | MLM、NSP/SOP | NLU、表示学习 |
| Encoder-Decoder | T5 | Encoder 双向；Decoder 因果；Cross-Attention | Span Corruption / Text-to-Text | Seq2Seq、条件生成 |
| Decoder-Only | GPT、LLaMA | Causal Self-Attention | CLM / Next Token Prediction | NLG、通用 LLM |

第三章还展示了 PLM 的演化逻辑：

**BERT → RoBERTa/ALBERT：优化理解模型的训练目标和参数效率**

**GPT → 更大参数 + 更多数据 → GPT-3 → LLM**

**GPT/LLaMA 路线最终成为当前通用大模型的主流路线。**

---

# 三种 PLM 架构详解

## 2. Encoder-Only PLM

### 2.1 是什么

Encoder-Only 只使用 Transformer 的 Encoder 堆叠。

典型代表：

- BERT
- RoBERTa
- ALBERT
- DistilBERT

### 2.2 核心实现机制

输入：

```text
文本
  ↓
Tokenizer
  ↓
Token IDs
  ↓
Token Embedding + Position Embedding
  ↓
Encoder Block × N
  ↓
每个 token 的双向上下文表示
  ↓
任务 Head
```

每一层核心：

```text
Hidden States
      ↓
Multi-Head Self-Attention
      ↓
Residual + Norm
      ↓
FFN
      ↓
Residual + Norm
```

最关键的是：

> **Self-Attention 不使用 Causal Mask。**

因此对于位置 `i`：

```text
token_i ← 可以关注 token_1 ... token_n
```

也就是同时看到左侧和右侧上下文。

### 2.3 为什么适合 NLU

例如：

```text
我去银行办理了贷款。
```

“银行”是金融机构。

```text
我坐在河流的银行……
```

如果只根据左侧上下文判断会受限制，而双向上下文可以结合完整句子进行语义理解。

### 2.4 BERT 的预训练

#### MLM

Masked Language Model，掩码语言模型。

```text
输入：I [MASK] you.
目标：love
```

模型利用 `[MASK]` 左右两侧信息预测被遮住的 token。

#### NSP

Next Sentence Prediction，下一句预测。

```text
Sentence A: I love you.
Sentence B: Because you are wonderful.
Label: IsNext
```

判断 B 是否是 A 后面的真实连续句子。

### 2.5 RoBERTa

核心思路：

- 仍然是 BERT Encoder 架构；
- 去掉 NSP；
- 使用动态 Mask；
- 使用更多数据、更大的 batch、更长训练；
- 强调“把 BERT 训练充分”本身就能明显提升效果。

### 2.6 ALBERT

主要优化：

- Embedding 参数分解；
- Encoder 跨层参数共享；
- 使用 SOP 替代 NSP。

SOP：

```text
正例：
A → B

负例：
B → A
```

让模型判断句子顺序是否正确。

---

## 3. Encoder-Decoder PLM

### 3.1 是什么

Encoder-Decoder 同时保留 Transformer 的两个部分。

代表：

- T5
- BART（同类 Seq2Seq PLM）

### 3.2 核心实现机制

```text
Input Tokens
     ↓
Encoder
     ↓
Encoder Memory
     ↓
────────────────────────
     ↓ Cross-Attention
Decoder ← 已生成的 Target Tokens
     ↓
Next Target Token
```

Encoder：

- 使用双向 Self-Attention；
- 负责理解完整输入。

Decoder：

1. Causal Self-Attention：只看已经生成的 token；
2. Cross-Attention：Query 来自 Decoder，K/V 来自 Encoder；
3. FFN；
4. 输出下一个 token。

Cross-Attention 可以写成：

$$
Q = H_{decoder}W_Q,\quad
K = H_{encoder}W_K,\quad
V = H_{encoder}W_V
$$

$$
Attention(Q,K,V) =
softmax\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

因此 Decoder 每生成一个 token，都可以查询 Encoder 对输入的完整语义表示。

### 3.3 T5 的核心思想

T5 = **Text-to-Text Transfer Transformer**。

所有任务统一成：

```text
文本输入 → 文本输出
```

例如：

```text
情感分类：
Input: sentiment: 这个产品很好
Output: positive
```

```text
翻译：
Input: translate English to German: How are you?
Output: Wie geht es dir?
```

```text
摘要：
Input: summarize: <long document>
Output: <summary>
```

### 3.4 T5 的预训练机制

更严格地说，经典 T5 最终采用 **Span Corruption**：

原文：

```text
Thank you for inviting me to your party last week.
```

损坏输入：

```text
Thank you <extra_id_0> me to your party <extra_id_1> week.
```

Decoder 目标：

```text
<extra_id_0> for inviting <extra_id_1> last <extra_id_2>
```

机制：

1. 随机选择约 15% token；
2. 将连续被选中的 token 合并为 span；
3. 每个 span 用一个 sentinel token 替换；
4. Encoder 输入损坏文本；
5. Decoder 自回归生成被删除的 span。

---

## 4. Decoder-Only PLM

### 4.1 是什么

Decoder-Only 保留 Transformer Decoder 的自回归部分，但没有 Encoder，因此也没有 Encoder-Decoder Cross-Attention。

代表：

- GPT 系列
- LLaMA 系列
- Qwen 等主流生成式 LLM

### 4.2 核心实现机制

```text
Tokens
  ↓
Embedding + Position Encoding
  ↓
Causal Self-Attention
  ↓
FFN
  ↓
... × N Layers
  ↓
LM Head
  ↓
Vocabulary Probability
  ↓
Next Token
```

最关键的是 **Causal Mask**。

假设序列：

```text
A B C D
```

注意力可见范围：

```text
A → A
B → A B
C → A B C
D → A B C D
```

不能：

```text
B → C/D
C → D
```

因此模型训练时模拟真实生成过程。

### 4.3 CLM：Decoder-Only 的核心训练任务

CLM = Causal Language Modeling。

给定：

$$
x_1,x_2,\dots,x_T
$$

训练目标：

$$
P(x_1,\dots,x_T) =
\prod_{t=1}^{T}P(x_t|x_{<t})
$$

Loss：

$$
\mathcal{L}_{CLM} =
-\sum_t \log P_\theta(x_t|x_{<t})
$$

代码层面的直觉就是“标签右移一位”：

```text
Input : 我 今天 去 北京
Label : 今天 去 北京 <EOS>
```

### 4.4 推理过程

给定 prompt：

```text
中国的首都是
```

模型得到：

```text
P(北京 | 中国的首都是)
```

选出“北京”后，将它拼回输入：

```text
中国的首都是 北京
```

继续：

```text
P(next_token | 中国的首都是 北京)
```

一直生成到 `<EOS>` 或最大长度。

---

# PLM 三种架构总表

| 对比项 | Encoder-Only | Encoder-Decoder | Decoder-Only |
|---|---|---|---|
| 代表模型 | BERT | T5 | GPT / LLaMA |
| Encoder | ✅ | ✅ | ❌ |
| Decoder | ❌ | ✅ | ✅式自回归 Block |
| 输入注意力 | 双向 | Encoder 双向 | 因果单向 |
| Cross-Attention | ❌ | ✅ | ❌ |
| 典型预训练任务 | MLM | Span Corruption | CLM |
| 天然强项 | 理解 | 条件生成/Seq2Seq | 自由生成/通用 LLM |
| 典型任务 | 分类、NER、匹配 | 翻译、摘要、QA | 对话、生成、代码、推理 |
| 每个 token 能看未来吗 | ✅ | Encoder ✅ / Decoder ❌ | ❌ |
| 自回归生成 | 不擅长 | ✅ | ✅ |
| 当前通用 LLM 主流 | 否 | 少数 | **是** |

---

# 第四章：大语言模型（LLM）

## 5. 什么是 LLM

LLM 本质仍然是语言模型，但相比传统 PLM：

- 参数规模更大；
- 预训练 token 更多；
- 数据类型更多；
- 算力投入更大；
- 出现更强的通用能力。

第四章重点强调的能力包括：

### 5.1 Emergent Abilities：涌现能力

模型规模扩大后，一些复杂能力可能在较大规模时明显提升。

面试时不要把“涌现”说成一个绝对严格的参数阈值，核心是：

> 规模增加后，某些复杂任务能力出现明显的非线性提升。

### 5.2 In-Context Learning：上下文学习

无需更新参数，仅把任务说明或示例放进 Prompt：

```text
正面：这个电影非常好看
负面：这个电影太无聊

判断：
这个故事很精彩 → ?
```

模型从上下文中的模式完成新任务。

包含：

- Zero-shot
- One-shot
- Few-shot

### 5.3 Instruction Following：指令遵循

经过指令微调后，模型可以按照自然语言要求完成未见过的任务。

例如：

```text
请把下面内容总结成三点……
```

### 5.4 Step-by-Step Reasoning / CoT

模型通过显式或隐式的中间推理过程处理多步任务。

典型形式：

```text
Question → Intermediate Reasoning → Answer
```

---

# LLM 三阶段训练

## 6. 总览

```text
大量无标注文本
      ↓
   Pretrain
      ↓
Base Model
      ↓
高质量指令数据
      ↓
     SFT
      ↓
Instruct / Chat Model
      ↓
人类偏好 / Reward
      ↓
RLHF / DPO / GRPO 等
      ↓
Aligned Model
```

一句话：

```text
Pretrain：学知识
SFT：学怎么回答
Alignment：学什么回答更好
```

---

## 7. Pretrain

### 7.1 是什么

Pretrain = 预训练。

对于当前 Decoder-Only LLM，最经典目标就是 **CLM / Next Token Prediction**。

不需要人工给每句话打标签，因为原始文本本身就能构造训练标签。

### 7.2 实现机制

#### Step 1：收集数据

例如：

- Common Crawl
- Wikipedia
- Books
- Code
- ArXiv
- Forums / QA
- 多语言语料

#### Step 2：数据清洗

包括：

- 去重；
- 去乱码；
- 语言识别；
- 质量过滤；
- 敏感/隐私过滤；
- 文档切分。

#### Step 3：Tokenizer

```text
文本 → token IDs
```

#### Step 4：Packing

把多个短文本拼成固定长度训练序列，提高 GPU 利用率。

#### Step 5：构造 CLM 标签

```text
Input : x1 x2 x3 x4
Target: x2 x3 x4 x5
```

#### Step 6：计算交叉熵

$$
\mathcal{L}_{pretrain} =
-\sum_t
\log P_\theta(x_t|x_{<t})
$$

#### Step 7：反向传播

使用 AdamW 等优化器，在多 GPU / TPU 上做：

- Data Parallel
- Tensor Parallel
- Pipeline Parallel
- ZeRO 等

更新模型参数。

### 7.3 Pretrain 学到了什么

主要获得：

- 语法；
- 词义；
- 世界知识；
- 文本风格；
- 一定程度的推理；
- 代码模式；
- 多语言映射。

但 Base Model 通常**不一定听指令**。

---

## 8. SFT

### 8.1 是什么

SFT = Supervised Fine-Tuning，监督微调。

训练数据：

```json
{
  "instruction": "将下列文本翻译成英文",
  "input": "今天天气真好",
  "output": "The weather is nice today."
}
```

现代 Chat 模型还常使用：

```text
System
User
Assistant
User
Assistant
...
```

### 8.2 实现机制

#### Step 1：构造 Chat Template

例如：

```text
<System> You are a helpful assistant.
<User> 解释 Transformer
<Assistant> Transformer 是……
```

#### Step 2：Tokenize

完整对话转换成 token IDs。

#### Step 3：构造 Loss Mask

常见做法：

```text
System tokens      → 不算 loss
User tokens        → 不算 loss
Assistant response → 计算 loss
```

在 PyTorch / Transformers 中通常将不参与 loss 的 label 设置为：

```text
-100
```

#### Step 4：仍然做 Next Token Prediction

SFT 并没有换掉 Decoder-Only 的语言建模本质。

$$
\mathcal{L}_{SFT} =
-\sum_{t \in assistant}
\log P_\theta(y_t|x,y_{<t})
$$

#### Step 5：更新参数

可以：

- Full Fine-Tuning；
- LoRA；
- QLoRA；
- 其他 PEFT。

### 8.3 SFT 主要带来什么

- 指令遵循；
- 对话格式；
- 多轮对话；
- 特定回答风格；
- 任务泛化。

### 8.4 一个很重要的理解

> **模型支持多轮对话不是因为 Transformer 自动“记住历史”，而是因为历史对话被重新放进上下文，并且 SFT 数据教会模型如何利用这种格式。**

---

## 9. RLHF

### 9.1 是什么

RLHF = Reinforcement Learning from Human Feedback。

核心目标：

> 让模型不仅“能回答”，而且更倾向于生成**人类认为更好、更有帮助、更安全**的回答。

经典 PPO-RLHF 可以拆成：

```text
SFT Model
   ↓
收集多个回答
   ↓
人类排序
   ↓
训练 Reward Model
   ↓
PPO 强化学习
   ↓
Aligned Model
```

---

## 10. Reward Model（RM）

### 10.1 偏好数据

典型数据：

```text
Prompt: x
Chosen: y_w
Rejected: y_l
```

即：

```text
y_w > y_l
```

### 10.2 RM 做什么

Reward Model：

```text
(prompt, response) → scalar reward
```

例如：

```text
好回答 → 4.2
差回答 → -1.3
```

### 10.3 训练思想

希望：

$$
r_\phi(x,y_w) > r_\phi(x,y_l)
$$

常用 Bradley-Terry 风格 pairwise loss：

$$
\mathcal{L}_{RM} =
-\log \sigma(r_\phi(x,y_w)-r_\phi(x,y_l))
$$

---

## 11. PPO-RLHF

经典 PPO-RLHF 中可以理解为有四个角色：

| 模型 | 是否更新 | 作用 |
|---|---:|---|
| Actor / Policy | ✅ | 被训练的 LLM |
| Reference Model | ❌ | 防止策略偏离 SFT 模型太远 |
| Reward Model | ❌ | 对回答打偏好分 |
| Critic / Value Model | ✅ | 估计 value / advantage |

训练循环：

```text
Prompt
  ↓
Actor 生成 Response
  ↓
Reward Model 打分
  ↓
Reference Model 计算 KL
  ↓
Critic 估计 Value
  ↓
计算 Advantage
  ↓
PPO Clipped Objective
  ↓
更新 Actor + Critic
```

为什么要 Reference Model？

防止模型为了“刷高 reward”完全偏离原本语言能力。

常见 reward：

$$
R =
R_{RM} -
\beta D_{KL}(\pi_\theta||\pi_{ref})
$$

---

# DPO

## 12. DPO 是什么

DPO = **Direct Preference Optimization**。

核心思想：

> 不再“先训练 Reward Model → 再用 PPO 最大化 reward”，而是直接让 policy 对 **chosen 的相对偏好高于 rejected**。

训练数据仍然是：

```text
(x, y_w, y_l)
```

其中：

- `x`：prompt；
- `y_w`：chosen；
- `y_l`：rejected。

---

## 13. DPO 实现原理

需要：

1. 当前训练模型 \(\pi_\theta\)
2. Reference Model \(\pi_{ref}\)
3. Preference Pair

分别计算 chosen 和 rejected 的 sequence log probability。

核心直觉：

```text
当前模型相对于 reference：
更应该提升 chosen
更应该压低 rejected
```

经典 DPO loss：

$$
\mathcal L_{DPO} =
-\log \sigma
\left(
\beta
\left[
\log\frac{\pi_\theta(y_w|x)}
{\pi_{ref}(y_w|x)} -
\log\frac{\pi_\theta(y_l|x)}
{\pi_{ref}(y_l|x)}
\right]
\right)
$$

训练结果：

```text
P(chosen | prompt) ↑
P(rejected | prompt) ↓
```

### DPO 的关键特点

- 不需要显式 Reward Model；
- 不需要 Critic；
- 不执行 PPO；
- 不需要训练过程中在线生成 rollout（经典 DPO）；
- 直接基于离线 preference pairs；
- 本质上更接近监督学习式偏好优化。

---

# GRPO

## 14. GRPO 是什么

GRPO = **Group Relative Policy Optimization**。

最初由 DeepSeekMath 系统性提出，是 PPO 的一种变体。

关键改动：

> **去掉 Critic / Value Model，用同一 prompt 下多条回答的“组内相对奖励”作为 advantage。**

---

## 15. GRPO 实现原理

对一个 Prompt \(q\)：

### Step 1：一次生成 G 个回答

$$
o_1,o_2,\dots,o_G
$$

例如：

```text
Prompt: 2 + 3 × 4 = ?

o1 = 14
o2 = 20
o3 = 14
o4 = 11
```

### Step 2：计算 reward

例如使用答案正确性：

```text
r = [1, 0, 1, 0]
```

也可以使用：

- Reward Model；
- Rule-based Reward；
- Verifier；
- Unit Test；
- Format Reward。

### Step 3：组内标准化

$$
A_i =
\frac{r_i-\mu_r}
{\sigma_r+\epsilon}
$$

其中：

$$
\mu_r = mean(r_1,\dots,r_G)
$$

$$
\sigma_r = std(r_1,\dots,r_G)
$$

于是：

```text
高于组平均 → A > 0 → 提高概率
低于组平均 → A < 0 → 降低概率
```

### Step 4：PPO-style clipped update

GRPO 仍然使用类似 PPO 的 ratio clipping：

$$
\rho_{i,t} =
\frac{
\pi_\theta(o_{i,t}|q,o_{i,<t})
}{
\pi_{\theta_{old}}(o_{i,t}|q,o_{i,<t})
}
$$

目标中使用：

$$
\min(
\rho_{i,t}A_i,
clip(\rho_{i,t},1-\epsilon,1+\epsilon)A_i
)
$$

同时通常加入针对 reference policy 的 KL penalty。

### GRPO 为什么省显存

PPO 需要一个通常很大的 Critic / Value Model。

GRPO：

```text
Critic ❌
组内 reward 统计 → baseline / advantage
```

因此显著降低强化学习阶段的显存和计算负担。

---

# RLHF、DPO、GRPO 对比

> 严格来说，**RLHF 是一个“使用人类反馈进行对齐”的大范式，而 PPO、GRPO 是优化算法，DPO 是直接偏好优化算法。**
>
> 所以下表实际比较的是最常见的 **PPO-RLHF vs DPO vs GRPO**。

| 对比项 | 经典 PPO-RLHF | DPO | GRPO |
|---|---|---|---|
| 是否做偏好/对齐训练 | ✅ | ✅ | ✅ |
| 是否属于在线 RL | ✅ | ❌，经典 DPO 是离线偏好优化 | ✅ |
| 是否需要 Preference Pair | ✅ 用来训练 RM | ✅ 直接训练 Policy | 不一定 |
| 是否需要显式 Reward Model | ✅ | ❌ | 不一定 |
| 是否需要 Reward Signal | ✅ | 偏好对本身提供监督 | ✅ |
| 是否需要 Critic / Value Model | ✅ | ❌ | **❌** |
| 是否需要 Reference Model | 通常 ✅ | 通常 ✅ | 通常 ✅ |
| 是否需要在线生成 Response | ✅ | ❌ | **✅，同一 Prompt 采样一组** |
| 是否有 PPO-style Clip | ✅ | ❌ | ✅ |
| 是否计算 Advantage | ✅，通常靠 Critic/GAE | ❌ | ✅，靠组内相对奖励 |
| 人类必须实时参与训练吗 | ❌，人类偏好通常先离线收集 | ❌ | ❌ |
| 可以使用自动可验证 Reward 吗 | ✅ | 不直接适用其经典形式 | **非常适合** |
| 工程复杂度 | 高 | 低 | 中等 |
| 显存开销 | 高 | 较低 | 比 PPO 低 |
| 典型数据 | Prompt + 偏好排名 + Rollout | Chosen / Rejected Pair | Prompt + Group Rollouts + Reward |
| 典型优势 | 成熟、在线探索能力强 | 简单稳定、成本低 | 无 Critic、适合 reasoning RL |
| 典型限制 | 复杂、昂贵、不稳定 | 受离线偏好数据质量约束 | 同一 prompt 要采样多条回答，生成成本高 |

---

# “有什么 / 没有什么”速查

| 方法 | Actor | Reference | Reward Model | Critic | Chosen/Rejected | 在线 Rollout | Group Sampling |
|---|---:|---:|---:|---:|---:|---:|---:|
| PPO-RLHF | ✅ | ✅ | ✅ | ✅ | ✅（RM 阶段） | ✅ | ❌ |
| DPO | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| GRPO | ✅ | 通常 ✅ | 可选 | **❌** | ❌/可选 | **✅** | **✅** |

最适合背的一句话：

```text
PPO：RM + Critic 都有
DPO：RM、Critic 都没有，直接吃偏好对
GRPO：Critic 没有，但仍然在线采样并需要 Reward
```

---

# 自注意力机制：MHA / MQA / GQA

## 16. 先回顾 Attention

$$
Attention(Q,K,V) =
softmax\left(
\frac{QK^T}{\sqrt{d_k}}
\right)V
$$

含义：

- Q（Query）：我正在找什么；
- K（Key）：每个 token 提供什么“索引”；
- V（Value）：真正被聚合的信息。

---

## 17. MHA：Multi-Head Attention

### 17.1 实现

假设：

```text
num_heads = 8
```

MHA：

```text
Q heads = 8
K heads = 8
V heads = 8
```

每个 head 都有独立：

$$
W_i^Q,W_i^K,W_i^V
$$

$$
head_i =
Attention(QW_i^Q,KW_i^K,VW_i^V)
$$

最终：

$$
MultiHead =
Concat(head_1,\dots,head_h)W^O
$$

### 17.2 特点

优点：

- 不同 head 学不同关系；
- 表达能力强。

缺点：

- 自回归推理时每个 head 都要缓存自己的 K/V；
- KV Cache 很大；
- 长上下文时内存带宽压力明显。

---

## 18. MQA：Multi-Query Attention

### 18.1 实现

仍然保留多个 Query Head：

```text
Q heads = 8
```

但 K/V 只有：

```text
K heads = 1
V heads = 1
```

也就是：

```text
Q1 ─┐
Q2 ─┤
Q3 ─┤
... ├── shared K / V
Q8 ─┘
```

所有 Q Head 都访问相同 K/V。

### 18.2 为什么快

自回归推理需要缓存历史 token 的 K/V。

MHA：

```text
每层缓存 h 组 K/V
```

MQA：

```text
每层只缓存 1 组 K/V
```

因此 KV Cache 大幅下降，内存带宽压力也大幅下降。

### 18.3 缺点

所有 Query Head 共用同一 K/V，信息多样性下降，可能损失模型质量。

---

## 19. GQA：Grouped-Query Attention

### 19.1 实现

GQA 是 MHA 和 MQA 的折中。

假设：

```text
Q heads = 8
KV heads = 2
```

那么：

```text
Q1 Q2 Q3 Q4 → K1/V1
Q5 Q6 Q7 Q8 → K2/V2
```

一般：

```text
h = Query Head 数
g = KV Head 数
```

每：

$$
h/g
$$

个 Query Head 共享一个 KV Head。

边界情况：

```text
g = h → MHA
g = 1 → MQA
1 < g < h → GQA
```

### 19.2 为什么 GQA 常用

它在两者之间取得平衡：

```text
MHA：质量高，KV Cache 大
         ↓
GQA：质量/效率折中
         ↓
MQA：KV Cache 最小，可能有更明显质量损失
```

---

# MHA / MQA / GQA 对比表

假设 Query Head 数 \(h=32\)：

| 对比项 | MHA | MQA | GQA（例：8 KV Heads） |
|---|---:|---:|---:|
| Query Heads | 32 | 32 | 32 |
| Key Heads | 32 | 1 | 8 |
| Value Heads | 32 | 1 | 8 |
| 每个 Q 是否有独立 K/V | ✅ | ❌ | 部分共享 |
| KV Cache | 最大 | 最小 | 中间 |
| 推理速度 | 相对较慢 | 快 | 快 |
| 表达能力 | 最强 | 可能下降 | 接近 MHA |
| 典型定位 | 质量优先 | 极致效率 | **质量/效率平衡** |

相对 MHA，GQA 的 KV Cache 大约可以缩小为：

$$
\frac{g}{h}
$$

例如：

```text
h = 32
g = 8
```

则 KV Head 数下降到原来的：

```text
8 / 32 = 1 / 4
```

---

# 第三、四章常见任务与术语速查

## 20. NLU

**Natural Language Understanding，自然语言理解。**

让模型理解文本含义。

典型：

- 文本分类；
- 情感分析；
- NER；
- NLI；
- 句子匹配；
- 抽取式 QA。

---

## 21. NLG

**Natural Language Generation，自然语言生成。**

让模型生成自然语言。

典型：

- 对话；
- 文本续写；
- 摘要；
- 翻译；
- 写作；
- 代码生成。

---

## 22. LM

**Language Modeling，语言模型。**

学习一个 token 序列的概率：

$$
P(x_1,\dots,x_T)
$$

最典型任务是：

```text
根据前文预测后文
```

---

## 23. CLM

**Causal Language Modeling，因果语言模型。**

每个 token 只依赖之前 token：

$$
P(x_t|x_{<t})
$$

GPT/LLaMA 的核心预训练任务。

---

## 24. MLM

**Masked Language Modeling，掩码语言模型。**

遮住部分 token，利用上下文恢复它们。

```text
北京是中国的 [MASK]
→ 首都
```

典型模型：BERT。

---

## 25. NSP

**Next Sentence Prediction，下一句预测。**

判断 Sentence B 是否真实跟在 Sentence A 后。

目的：学习句间关系。

典型模型：BERT。

---

## 26. SOP

**Sentence Order Prediction，句序预测。**

给出真实相邻句子：

```text
A → B
```

负例：

```text
B → A
```

判断顺序是否正确。

典型模型：ALBERT。

---

## 27. Seq2Seq

**Sequence-to-Sequence，序列到序列。**

输入一个序列，输出另一个序列。

例如：

```text
英文 → 中文
文章 → 摘要
问题 + 文档 → 答案
```

典型架构：Encoder-Decoder。

---

## 28. NLI

**Natural Language Inference，自然语言推理。**

判断 Hypothesis 与 Premise 的关系：

- Entailment：蕴含；
- Contradiction：矛盾；
- Neutral：中立。

---

## 29. QA

**Question Answering，问答。**

根据问题直接回答，或结合给定文本找到答案。

可分为：

- Extractive QA；
- Generative QA。

---

## 30. Text Classification

文本 → 类别。

例如：

```text
“这个电影很好看”
→ Positive
```

---

## 31. Span Corruption / Text Infilling

遮掉一段连续 token，让模型恢复整个 span。

T5 的经典预训练任务属于这一类。

---

## 32. Autoregressive Blank Infilling

GLM 等模型探索过的一类训练方式：

- 输入中挖掉一个或多个连续 span；
- 模型结合上下文；
- 对空白内容以自回归方式生成。

可以理解为把 MLM 的“双向上下文”和 CLM 的“自回归生成”结合起来。

---

## 33. In-Context Learning（ICL）

不更新模型参数，只在上下文里提供任务说明或示例让模型完成任务。

---

## 34. Zero-shot / Few-shot

### Zero-shot

只给指令，不给示例。

### Few-shot

给少量输入—输出示例，再要求模型完成新样本。

---

## 35. Instruction Tuning

使用大量不同任务的自然语言指令进行 SFT。

目标：

> 不是只学会一个任务，而是学会“按照自然语言指令做任务”。

---

## 36. CoT

**Chain-of-Thought，思维链。**

让复杂任务通过中间推理步骤得到最终答案。

面试中可理解为：

```text
Problem
  ↓
Reasoning Steps
  ↓
Final Answer
```

---

# 最后一页：一眼记住

## 架构

```text
BERT
= Encoder-Only
= 双向 Attention
= MLM
= NLU

T5
= Encoder-Decoder
= Encoder 理解 + Decoder 生成
= Cross-Attention
= Span Corruption
= Seq2Seq

GPT / LLaMA
= Decoder-Only
= Causal Attention
= CLM
= Next Token Prediction
= 主流 LLM
```

## 训练

```text
Pretrain
海量无标注数据
↓
学语言 + 知识

SFT
指令-回答数据
↓
学会遵循指令

PPO-RLHF
人类偏好 → RM → PPO
↓
学会偏好对齐
```

## 偏好优化

```text
PPO-RLHF
Reward Model ✅
Critic ✅
Online Rollout ✅

DPO
Reward Model ❌
Critic ❌
Preference Pair ✅
Online Rollout ❌

GRPO
Reward Signal ✅
Critic ❌
Online Rollout ✅
Group Sampling ✅
```

## Attention

```text
MHA：Q 多，K/V 也多
MQA：Q 多，K/V 只有 1 组
GQA：Q 多，K/V 分组共享

MHA → 质量
MQA → 极致 KV Cache 优化
GQA → 二者折中
```
