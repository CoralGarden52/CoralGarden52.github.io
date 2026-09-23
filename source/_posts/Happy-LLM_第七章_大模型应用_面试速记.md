---
title: happy-llm第七章：大模型应用的阅读笔记
cover: /assets/posts/happy-llm/happy-llm-7.png 
categories: llm
tags:
  - happy-llm
---

# 面试速记

> 基于 Datawhale《Happy-LLM》第七章 **大模型应用** 整理。第七章主要包含三部分：**LLM Evaluation、RAG、Agent**。

## 背诵版总结

1. **LLM Evaluation**：用标准化数据集和指标衡量模型在知识、数学、推理、长文本、多语言、工具调用等方面的能力。
2. 典型评测集：
   - **MMLU**：多学科知识与理解；
   - **BFCL V2**：工具调用；
   - **GSM8K / MATH**：数学；
   - **ARC Challenge / GPQA / HellaSwag**：推理；
   - **InfiniteBench / Multi-needle**：长文本；
   - **MGSM**：多语言数学。
3. 典型榜单：
   - Open LLM Leaderboard；
   - LMSYS Chatbot Arena；
   - OpenCompass；
   - 金融、安全、法律、医疗等垂直榜单。
4. **RAG = Retrieval-Augmented Generation**，核心是先检索外部知识，再让 LLM 基于检索结果生成答案。
5. RAG 流程：

```text
Document
→ Chunk
→ Embedding
→ Vector Store
→ Query Embedding
→ Similarity Search
→ Top-k Context
→ Prompt
→ LLM
→ Answer
```

6. RAG 主要用于缓解：
   - 幻觉；
   - 知识过时；
   - 私有知识无法直接注入；
   - 回答缺少依据。
7. **Embedding** 把文本映射成向量，语义相近文本在向量空间中更接近。
8. Tiny-RAG 使用余弦相似度：

$$
\cos(\theta) =
\frac{
\mathbf{a}\cdot\mathbf{b}
}{
\|\mathbf{a}\|
\|\mathbf{b}\|
}
$$

9. **Agent = LLM + Planning + Memory + Tool Use + Reflection**。
10. Agent 不仅回答问题，还会：
    - 理解目标；
    - 拆解任务；
    - 调用工具；
    - 根据工具结果继续推理；
    - 反思并迭代。
11. 第七章把 Agent 分为：
    - Task-Oriented Agent；
    - Planning & Reasoning Agent；
    - Multi-Agent System；
    - Exploration & Learning Agent。
12. **ReAct = Reason + Act**：

```text
Reason
→ Action
→ Observation
→ Reason
→ ...
```

13. Tiny-Agent 使用 OpenAI API `tool_calls`：
    - Python 函数转 JSON Schema；
    - 传给 LLM；
    - LLM 返回 tool_calls；
    - 程序执行函数；
    - Tool Result 回传；
    - LLM 生成最终答案。

一句话：

> **第七章讲的是 LLM 如何真正落地：Evaluation 负责“测能力”，RAG 负责“补知识”，Agent 负责“做事情”。**

---

## 面试表达模板

### 1. 请介绍 Happy-LLM 第七章

> 第七章主要介绍大模型应用，包含评测、RAG 和 Agent。评测部分通过 MMLU、GSM8K、BFCL 等标准数据集衡量模型能力；RAG 通过文档切分、Embedding、向量检索和大模型生成，将外部知识引入上下文；Agent 则在 LLM 基础上加入规划、记忆和工具调用，使模型能够完成复杂、多步骤任务。

### 2. 什么是 RAG？

> RAG 是检索增强生成。它不会只依赖模型参数中的知识，而是在回答问题前先从外部知识库中检索相关文档，把检索到的内容作为 Context 与问题一起交给大模型，再生成答案。这样可以降低知识过时和幻觉问题，并提升回答的可追溯性。

### 3. 一个完整 RAG 系统如何工作？

> 离线阶段先加载文档、切 Chunk，并用 Embedding 模型将每个 Chunk 转成向量保存到向量数据库。在线问答时，把用户 Query 也转换成向量，根据余弦相似度检索最相关的 Top-k 文档，将检索结果和问题拼接为 Prompt，再交给 LLM 生成答案。

### 4. 什么是 LLM Agent？

> LLM Agent 是以大语言模型为核心，并增加 Planning、Memory、Tool Use 和 Reflection 能力的系统。普通 LLM 通常直接回答，而 Agent 可以先判断任务、选择工具、执行工具、观察结果，再继续推理，直到完成目标。

### 5. Tool Calling 怎么实现？

> 先把工具函数的名称、描述和参数转换成 JSON Schema，并传给 LLM。模型如果决定调用工具，会返回 tool_calls，其中包含函数名和参数。Agent Runtime 执行对应 Python 函数，再把 Tool Result 作为 tool message 写回上下文，最后再次调用 LLM 生成最终回答。

---

## 高频面试问题

### Q1：为什么要做 LLM Evaluation？

参数量和训练 Loss 不能完整说明模型能力。评测可以量化：

```text
知识
数学
推理
工具使用
长文本
多语言
鲁棒性
资源消耗
```

所以 Evaluation 的作用是把“模型看起来很强”变成可比较的客观结果。

### Q2：MMLU 是什么？

MMLU 全称：

> **Massive Multitask Language Understanding**

覆盖历史、数学、物理、生物、法律等多个学科，用于衡量模型的跨学科知识储备和理解能力。

### Q3：GSM8K 和 MATH 有什么区别？

```text
GSM8K
→ 基础数学文字题
→ 多步骤算术推理

MATH
→ 更复杂的代数、几何、竞赛数学
→ 整体难度更高
```

### Q4：BFCL 是什么？

BFCL 主要评估：

> **Function Calling / Tool Use**

也就是模型能否根据用户意图正确选择工具并构造调用参数。

### Q5：Open LLM Leaderboard、Chatbot Arena、OpenCompass 有什么区别？

| 榜单 | 主要特点 |
|---|---|
| Open LLM Leaderboard | 标准 Benchmark 评测开源模型 |
| LMSYS Chatbot Arena | 更强调真实用户对话偏好 |
| OpenCompass | 多任务评测，强调中文与国内模型生态 |

---

# 第七章：大模型应用

## 1. 第七章整体结构

```text
7.1 LLM 的评测
7.2 RAG
7.3 Agent
```

可以记成：

```text
Evaluation
→ 模型到底强不强？

RAG
→ 模型缺外部知识怎么办？

Agent
→ 模型怎样真正执行任务？
```

---

# 2. LLM 的评测

大模型评测是：

> **通过标准化的方法和数据集，对模型在不同任务上的表现进行量化和比较。**

评测不只看 Accuracy，还可能关注：

```text
泛化能力
推理能力
工具使用
推理速度
资源消耗
鲁棒性
安全性
```

---

# 3. 第七章提到的评测集

| 类别 | Benchmark | 主要能力 |
|---|---|---|
| 通用 | MMLU | 多学科知识与理解 |
| 工具使用 | BFCL V2 | Tool / Function Calling |
| 数学 | GSM8K | 基础数学推理 |
| 数学 | MATH | 高难数学 |
| 推理 | ARC Challenge | 科学与常识推理 |
| 推理 | GPQA | 高难问题回答与推理 |
| 推理 | HellaSwag | 语境理解与合理续写 |
| 长文本 | InfiniteBench / En.MC | 长上下文理解 |
| 长文本 | Multi-needle | 长文档多信息检索 |
| 多语言 | MGSM | 多语言数学推理 |

---

# 4. 主流评测榜单

## Open LLM Leaderboard

由 Hugging Face 提供，主要使用标准 Benchmark 对开源模型进行统一对比。

## LMSYS Chatbot Arena

通过真实用户与模型交互并进行偏好比较，重点关注实际对话体验。

## OpenCompass

面向多任务、多语言和中文场景的大模型评测平台。

---

# 5. 垂直领域榜单

第七章还提到：

| 领域 | Benchmark |
|---|---|
| 金融 | CFBenchmark |
| 安全 | Flames |
| 通识对话 | BotChat |
| 法律 | LawBench |
| 医疗 | MedBench |

这说明：

> **通用模型能力高，不代表在所有专业领域都同样可靠。**

---

# 6. 什么是 RAG？

RAG 全称：

> **Retrieval-Augmented Generation**

普通 LLM：

```text
Question
↓
LLM 参数知识
↓
Answer
```

RAG：

```text
Question
↓
Knowledge Retrieval
↓
Relevant Context
↓
Question + Context
↓
LLM
↓
Answer
```

---

# 7. 为什么需要 RAG？

主要原因：

### 7.1 幻觉

LLM 可能生成看起来合理、实际上错误的信息。

### 7.2 知识过时

模型参数中的知识来自训练数据，新政策、新论文、新产品等可能不在参数中。

### 7.3 私有知识

企业文档、个人笔记、数据库等不适合每次重新微调模型。

因此 RAG 的核心思想是：

> **知识不一定要写进模型参数，也可以在回答时动态检索。**

---

# 8. RAG 三阶段

```text
Indexing
Retrieval
Generation
```

## Indexing

```text
Document
↓
Chunk
↓
Embedding
↓
Vector Index
```

## Retrieval

```text
Query
↓
Query Embedding
↓
Similarity Search
↓
Top-k Chunks
```

## Generation

```text
Question
+
Retrieved Context
↓
LLM
↓
Answer
```

---

# 9. Tiny-RAG 五个核心模块

第七章 Tiny-RAG 包括：

1. 向量化模块；
2. 文档加载和切分模块；
3. 数据库；
4. 检索模块；
5. 大模型模块。

完整流程：

```text
Documents
↓
ReadFiles
↓
Chunks
↓
Embedding Model
↓
Vectors
↓
VectorStore
↓
Query
↓
Retrieval
↓
Top-k Context
↓
LLM
↓
Answer
```

---

# 10. 文档加载

第七章代码示例支持：

```text
PDF
Markdown
TXT
```

目标是把不同文件统一转换成纯文本。

---

# 11. Chunk

Chunk：

> **把长文档切成较短文本片段。**

原因：

- Embedding 模型有输入长度限制；
- LLM Context 有限制；
- 小片段检索粒度更细；
- 容易定位准确知识。

---

# 12. Chunk Size

Tiny-RAG 示例：

```text
max_token_len = 600
```

Chunk 太大：

```text
上下文丰富
但噪声更多
```

Chunk 太小：

```text
定位更精确
但语义可能不完整
```

---

# 13. Chunk Overlap

第七章代码中的：

```text
cover_content
```

本质就是 Overlap。

例如：

```text
Chunk 1:
ABCDE

Chunk 2:
DEFGH
```

其中：

```text
DE
```

是重复部分。

作用：

> **避免一个完整语义刚好被切断在 Chunk 边界。**

---

# 14. Embedding

Embedding：

> **将文本映射成固定维度向量。**

$$
f: Text \rightarrow \mathbb{R}^{d}
$$

例如：

```text
"什么是 RAG？"
↓
Embedding Model
↓
[0.12, -0.43, ..., 0.81]
```

语义越相近，向量通常也越接近。

---

# 15. BaseEmbeddings

第七章实现 `BaseEmbeddings` 抽象基类。

主要方法：

```text
get_embedding()
cosine_similarity()
```

好处：

> **未来更换 Embedding 模型时，只需继承基类实现对应方法。**

---

# 16. Cosine Similarity

$$
Similarity(A,B) =
\frac{
A \cdot B
}{
\|A\|
\|B\|
}
$$

理论范围：

$$
[-1,1]
$$

通常越接近 1，方向越相似，语义也越接近。

---

# 17. Vector Store

Vector Store 用于：

> **保存文档 Chunk 与其 Embedding 向量。**

Tiny-RAG 中主要接口：

```text
get_vector
persist
load_vector
query
```

---

# 18. Persist

`persist`：

> **把已经构建好的向量数据库保存到磁盘。**

第一次：

```text
Document
→ Chunk
→ Embedding
→ Persist
```

以后：

```text
Load Vector Store
```

不需要重复生成所有 Embedding。

---

# 19. Query Retrieval

用户问题先变成 Query Vector：

```text
Query
↓
Embedding
↓
Query Vector
```

然后计算：

$$
s_i = \cos(q,d_i)
$$

最后排序并取：

```text
Top-k
```

---

# 20. Top-k

Top-k：

> **返回相似度最高的 k 个文档片段。**

k 太小：

```text
可能漏掉信息
```

k 太大：

```text
噪声更多
Token 消耗更多
```

---

# 21. RAG Prompt

检索结果一般不会直接返回给用户，而是组合成：

```text
Instruction
+
Question
+
Retrieved Context
```

再交给 LLM。

常见约束：

```text
请依据 Context 回答。
如果 Context 中没有答案，请明确说不知道。
```

作用：

> **减少模型脱离检索内容自由发挥。**

---

# 22. Tiny-RAG 完整流程

```text
PDF / MD / TXT
       ↓
   ReadFiles
       ↓
    Chunking
       ↓
Embedding Model
       ↓
Vector Store
       ↓
    Persist
       ↓
────────────────
       ↓
User Question
       ↓
Query Embedding
       ↓
Cosine Similarity
       ↓
Top-k Retrieval
       ↓
Retrieved Context
       ↓
RAG Prompt
       ↓
      LLM
       ↓
     Answer
```

---

# 23. RAG vs Fine-Tuning

| 对比项 | RAG | Fine-Tuning |
|---|---|---|
| 修改模型参数 | ❌ | ✅ |
| 更新知识速度 | 快 | 慢 |
| 是否训练 | 通常不需要 | 需要 |
| 私有知识接入 | 方便 | 成本较高 |
| 回答依据 | 更容易追溯 | 较弱 |
| 主要目的 | 补充知识 | 改变能力/行为 |

一句话：

```text
Fine-Tuning
→ 改模型

RAG
→ 改上下文
```

---

# 24. 什么是 Agent？

Agent：

> **以 LLM 为核心，同时具备规划、记忆、工具调用和环境交互能力的系统。**

普通 LLM：

```text
Prompt
↓
Answer
```

Agent：

```text
Goal
↓
Planning
↓
Action
↓
Tool / Environment
↓
Observation
↓
Reasoning
↓
Next Action
↓
...
↓
Final Answer
```

---

# 25. Agent 五个核心能力

第七章总结：

1. Goal Understanding；
2. Planning；
3. Memory；
4. Tool Use；
5. Reflection & Iteration。

可以背成：

```text
理解目标
→ 拆任务
→ 记状态
→ 用工具
→ 看结果再调整
```

---

# 26. Planning

Planning：

> **把复杂目标拆成一系列可执行子任务。**

例如：

```text
规划旅行
↓
查天气
↓
查景点
↓
查航班
↓
找酒店
↓
生成行程
```

---

# 27. Memory

Agent Memory 常分：

## Short-Term Memory

```text
当前任务
对话历史
中间状态
工具结果
```

## Long-Term Memory

```text
历史经验
用户偏好
长期知识
过去任务结果
```

---

# 28. Tool Use

工具可以是：

```text
Search
Calculator
Database
Python
API
Files
...
```

LLM 负责：

> **决定是否需要工具，以及调用哪个工具。**

真正执行工具的是外部程序。

---

# 29. Reflection

Reflection：

```text
执行
↓
检查结果
↓
是否成功？
├── 否 → 修改计划 / 重试
└── 是 → 继续
```

也就是 Agent 根据执行反馈动态修正后续行动。

---

# 30. Agent 四种类型

## Task-Oriented Agent

适合：

```text
客服
预订
代码助手
数据分析
```

## Planning & Reasoning Agent

特点：

```text
复杂任务
多步规划
动态调整
```

常结合 ReAct、CoT 和 Tool Use。

## Multi-Agent System

多个 Agent 分工协作：

```text
Planner
Developer
Reviewer
Tester
```

第七章举例提到 AutoGen、ChatDev。

## Exploration & Learning Agent

特点：

> 能从和环境的交互、成功或失败经验中不断改进策略。

---

# 31. ReAct

ReAct：

> **Reasoning + Acting**

```text
Reason
↓
Action
↓
Observation
↓
Reason
↓
Action
↓
...
```

它让模型可以根据真实工具反馈调整下一步动作。

---

# 32. CoT 与 ReAct

## CoT

```text
Reason
→ Reason
→ Reason
→ Answer
```

重点是推理。

## ReAct

```text
Reason
→ Act
→ Observation
→ Reason
```

重点是：

> **推理 + 执行 + 环境反馈**

简单记：

```text
CoT = Think
ReAct = Think + Do + Observe
```

---

# 33. Tiny-Agent

第七章 Tiny-Agent 使用：

```text
OpenAI Python SDK
+
tool_calls
```

核心组件：

```text
OpenAI Client
LLM
Tools
JSON Schema
Agent Class
Message History
```

---

# 34. Tool Function

例如：

```python
def get_current_datetime():
    ...
```

```python
def count_letter_in_string(a, b):
    ...
```

LLM 本身不能直接执行 Python，所以需要先描述工具。

---

# 35. Docstring

工具函数需要清楚的 docstring。

原因：

> **docstring 会用于构造工具描述，让 LLM 理解工具功能和参数。**

---

# 36. JSON Schema

工具会转换成类似：

```json
{
  "type": "function",
  "function": {
    "name": "add",
    "description": "...",
    "parameters": {
      "type": "object"
    }
  }
}
```

JSON Schema 是：

> **函数的机器可读说明书。**

---

# 37. Tool Calling

请求模型时：

```python
client.chat.completions.create(
    messages=messages,
    tools=tool_schema
)
```

模型可以：

```text
直接回答
```

或者返回：

```text
tool_calls
```

---

# 38. tool_calls

通常包含：

```text
Function Name
Arguments
Tool Call ID
```

Agent Runtime 根据这些信息执行真实函数。

---

# 39. Tool Message

工具执行结果需要重新加入 messages：

```text
role = tool
```

并带：

```text
tool_call_id
```

这样模型才能知道：

> 这条 Tool Result 对应前面的哪一次调用。

---

# 40. Tiny-Agent 完整流程

```text
User Question
     ↓
加入 Message History
     ↓
LLM + Tool Schema
     ↓
是否需要工具？
   ↙         ↘
 否           是
 ↓            ↓
Answer     Tool Call
              ↓
          Parse Args
              ↓
        Execute Python
              ↓
         Tool Result
              ↓
       加入 messages
              ↓
          再次调用 LLM
              ↓
         Final Answer
```

---

# 41. Message History

第七章中：

```python
self.messages
```

保存：

```text
System
User
Assistant
Tool
```

消息。

它是 Tiny-Agent 最基础的：

> **Short-Term Memory。**

---

# 42. System Prompt

System Prompt 负责定义：

```text
Agent 是谁
行为规则是什么
什么时候可以调用工具
输出风格是什么
```

因此 Agent 能力来自：

```text
LLM
+
System Prompt
+
Tool Schema
+
Runtime
```

---

# 43. Tiny-Agent vs Chatbot

| 能力 | Chatbot | Agent |
|---|---|---|
| 对话 | ✅ | ✅ |
| 历史上下文 | ✅ | ✅ |
| 工具调用 | 不一定 | ✅ |
| 执行动作 | 通常 ❌ | ✅ |
| Planning | 较弱 | ✅ |
| 环境反馈 | 通常 ❌ | ✅ |
| 多步迭代 | 较弱 | ✅ |

---

# 44. RAG 与 Agent 的关系

RAG 和 Agent 可以组合。

例如：

```text
Agent
↓
发现需要查内部知识
↓
调用 RAG Retriever
↓
获取 Context
↓
继续推理
```

所以：

```text
RAG
→ 解决知识问题

Agent
→ 解决任务执行问题
```

RAG 可以成为 Agent 的一个 Tool。

---

# 45. LLM / RAG / Agent 对比

| 项目 | LLM | RAG | Agent |
|---|---|---|---|
| 核心 | 参数知识 | 检索 + LLM | LLM + Planning + Tools |
| 外部知识 | 较弱 | ✅ | ✅ |
| 工具调用 | 非核心 | 非核心 | **核心** |
| 多步骤执行 | 较弱 | 较弱 | **强** |
| 知识更新 | 重新训练 | 更新知识库 | 更新知识源/工具 |
| 典型用途 | 对话生成 | 知识库问答 | 自动化复杂任务 |

---

# 46. 一页速记

```text
第七章
= LLM Application

Evaluation：
MMLU
BFCL
GSM8K
MATH
ARC
GPQA
HellaSwag
InfiniteBench
MGSM

RAG：
Document
→ Chunk
→ Embedding
→ Vector Store
→ Query Embedding
→ Similarity
→ Top-k
→ Context
→ LLM

Agent：
LLM
+ Planning
+ Memory
+ Tool Use
+ Reflection

ReAct：
Reason
→ Act
→ Observation
→ Reason

Tool Calling：
Function
→ JSON Schema
→ LLM
→ tool_calls
→ Execute Tool
→ Tool Result
→ LLM
→ Answer
```

---

# 47. 最后总结

第七章完成了从：

```text
模型
```

到：

```text
应用系统
```

的转变。

三个核心问题：

```text
Evaluation：
模型到底有多强？

RAG：
模型缺外部知识怎么办？

Agent：
模型怎样真正执行复杂任务？
```

最终一句话：

> **Evaluation 负责测能力，RAG 负责补知识，Agent 负责做事情。**

---

# 参考资料

1. Datawhale, Happy-LLM  
   https://datawhalechina.github.io/happy-llm/

2. Datawhale, Happy-LLM 第七章：大模型应用  
   https://github.com/datawhalechina/happy-llm/blob/main/docs/chapter7/%E7%AC%AC%E4%B8%83%E7%AB%A0%20%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%BA%94%E7%94%A8.md
