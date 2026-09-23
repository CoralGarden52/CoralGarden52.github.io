---
title: happy-llm第八章大模型强化学习的阅读笔记
cover: /assets/posts/happy-llm/happy-llm-8.png 
categories: llm
tags:
  - happy-llm
---

# 面试速记

> 基于 Datawhale《Happy-LLM》第八章 **大模型强化学习 / Agentic RL** 整理，并整合本次粘贴文本中关于 PyTRIO、Verl、DAPO、FSDP、GRPO、OPD、Search-R1、ReTool、importance sampling、token 对齐、on-policy 等补充讨论。
>
> **说明：**第八章正文包含大量代码。本文对正文中的 **86 个代码/命令/轨迹示例块逐一解释**；为了避免笔记变成源码复制，正文只保留关键逻辑的**简化版代码/伪代码**，完整源码应以 Happy-LLM 仓库为准。

## 背诵版总结

1. 第八章的主题是 **Agentic RL**：不仅让 LLM“回答得更好”，还让模型在搜索、代码执行等环境中通过 **rollout → feedback → advantage → policy update** 学会行动。
2. 全章四条主线：

```text
GRPO
→ OPD
→ Search-R1
→ ReTool
```

3. 四个案例统一使用 **PyTRIO**。其核心闭环可以背成：

```text
当前策略生成 rollout
→ 环境 / Teacher 提供反馈
→ 构造 token-level advantage
→ 构造 Datum
→ forward_backward()
→ optim_step()
→ 刷新 sampler
→ 下一轮 rollout
```

4. **GRPO**：同一个 prompt 采样一组回答，使用组内 reward 的均值/标准差得到相对 advantage，不需要单独 Value/Critic Model。
5. GRPO 组内优势：

$$
A_i =
\frac{
r_i-\mu_r
}{
\sigma_r+\epsilon
}
$$

6. **importance sampling ratio**：

$$
\rho_t(\theta) =
\frac{
\pi_\theta(y_t\mid s_t)
}{
\pi_{\theta_{\mathrm{old}}}(y_t\mid s_t)
}
$$

它是策略更新中的一个“纠偏/重加权工具”，而 **GRPO 是完整 RL 算法**；两者不是同一层级概念。
7. **PPO** 在 importance sampling 基础上加入 clipping，限制单次策略变化过大。
8. GRPO 的 `run_rollout_group()` 做四件事：

```text
同题采样多条 completion
→ 保存旧策略 logprob
→ 规则判题得到 reward
→ 组内标准化得到 advantage
```

9. **completion** 就是 prompt 之后由模型生成的那段 token 序列，不一定等于“最终答案”；它可能包含推理过程、工具调用和最终答案。
10. `build_grpo_datum()` 的关键是**右移一位**：模型每个输入位置预测下一个 token；prompt 只是条件，completion 才是策略动作，因此 prompt 区域的 policy loss 输入使用 0 占位。
11. **on-policy 约束**：第 \(k\) 次更新后，第 \(k+1\) 次 rollout 应尽量使用更新后的当前策略生成。Sampler 必须定期刷新，否则数据逐渐变成 off-policy。
12. **OPD = On-Policy Distillation**：Student 自己生成轨迹，Teacher 不重新生成答案，而是在 Student 实际走过的每一个 token 状态上给出概率反馈。
13. OPD 使用 reverse KL 的单样本信号：

$$
d_t =
\log \pi_{\theta_{\mathrm{old}}}(y_t\mid s_t) -
\log \pi_T(y_t\mid s_t)
$$

$$
A_t =
-\beta d_t
$$

14. GRPO 是**轨迹级稀疏 reward → advantage**；OPD 是 **Teacher 的逐 token 稠密反馈 → advantage**。
15. **Search-R1**：模型自己决定“何时搜索、搜什么”，搜索结果作为 observation 加入上下文，然后继续生成；最终答案 reward 反向强化前面的搜索策略。
16. Search-R1 与传统 RAG 的核心区别：

```text
RAG：
系统预先决定检索流程

Search-R1：
模型通过 RL 学习是否搜索、搜索什么、是否继续搜索
```

17. **observation mask**：搜索结果/工具返回会进入上下文，但不是模型动作，因此：
   - 可以被后续 token 看到；
   - 不参与 policy loss；
   - old logprob / advantage 填 0。
18. **ReTool** 把 Search-R1 的“搜索环境”替换成“Python 代码解释器”。模型可以写代码、看到 stdout/stderr、根据错误重试，再输出最终答案。
19. ReTool 只需要最终 outcome reward；成功轨迹中的代码调用、推理和答案共享轨迹 advantage，因此模型能学习“怎样使用代码工具有助于成功”。
20. **PyTRIO vs Verl**：
   - PyTRIO：强调低工程门槛、托管式 API、适合快速研究 Agentic RL；
   - Verl：面向大规模高性能 RL 训练，集成 FSDP/Megatron-LM、vLLM/SGLang 等，工程复杂度更高。
21. 第八章正文之外的扩展算法：
   - DAPO：GRPO/PPO 路线的稳定性与采样效率改进；
   - GSPO：把策略比率/裁剪提升到 sequence 级；
   - OPSD：On-Policy Self-Distillation；
   - ALFWorld：文本化具身家务环境；
   - FSDP：PyTorch 原生全分片数据并行，属于系统层而非 RL 算法。

一句话：

> **第八章的核心不是死记某个算法，而是掌握一条统一数据流：当前模型自己产生轨迹，环境或 Teacher 在真实轨迹上给反馈，把反馈对齐到模型自己生成的 token，再更新策略并用新策略重新采样。**

---

## 面试表达模板

### 1. 请概括第八章

> 第八章讲 Agentic RL，核心包含 GRPO、OPD、Search-R1 和 ReTool。GRPO 用同题多采样得到组内相对优势；OPD 让 Student 在自己的状态分布上生成轨迹，再由 Teacher 对这些 token 提供逐 token 概率反馈；Search-R1 把 RL 扩展到搜索环境，让模型学习何时搜索和搜索什么；ReTool 则把环境换成代码解释器，让模型学会通过代码执行和错误反馈完成任务。四者都统一成 rollout、反馈、advantage、Datum、策略更新和 sampler 刷新的闭环。

### 2. GRPO 和 PPO 的主要区别是什么？

> PPO 通常需要 Critic/Value Model 来估计 advantage；GRPO 对同一个 prompt 采样一组回答，使用组内 reward 做相对标准化，直接得到 advantage，从而省掉单独的 Value Model。策略更新仍可使用 importance sampling ratio，并可以进一步使用 PPO clipping 控制策略变化。

### 3. OPD 为什么叫 On-Policy？

> 因为监督样本不是固定离线数据，而是 Student 当前策略自己生成的轨迹。Teacher 沿着 Student 实际访问的状态逐 token 打分，所以 Student 每次学到的是“自己当前真的会走到的状态”上的 Teacher 信号，减少离线蒸馏中的 distribution shift。

### 4. Search-R1 与 RAG 有什么区别？

> RAG 通常由系统固定执行一次或若干次检索，然后把结果交给 LLM；Search-R1 把搜索本身作为模型动作，让模型通过强化学习学会是否搜索、如何组织查询、何时继续搜索以及什么时候回答。搜索结果是 environment observation，而搜索词和最终答案都是模型策略产生的动作。

### 5. ReTool 的核心是什么？

> ReTool 将代码解释器视为 RL 环境。模型生成代码工具调用，执行器返回 stdout、stderr 或 timeout，这些结果进入下一轮上下文，模型可以继续推理和修正。最终答案是否正确形成 outcome reward，再通过轨迹 advantage 对此前的代码调用、推理和最终回答共同进行信用分配。

---

## 高频面试问题

### Q1：rollout、reward、advantage 分别是什么？

- **Rollout**：当前策略从 prompt 出发生成的一条完整轨迹。
- **Reward**：环境对轨迹结果的评价，例如数学答案是否正确。
- **Advantage**：某条动作/轨迹相对基线“好多少”，决定提高还是降低对应动作概率。

在 LLM 中可以对应为：

| RL 概念 | LLM 中的含义 |
|---|---|
| State | prompt + 已生成 token + observation |
| Action | 下一个模型生成 token |
| Policy | LLM next-token 分布 |
| Trajectory | 一段回答或多轮工具轨迹 |
| Environment | 判题器、搜索引擎、代码解释器 |
| Reward | 正确性、格式、任务是否成功 |

### Q2：importance sampling 与 GRPO 的区别？

`importance_sampling` 是**统计/策略优化技术**，GRPO 是**完整算法**。

$$
\rho_t =
\frac{
\pi_\theta(a_t\mid s_t)
}{
\pi_{\mathrm{old}}(a_t\mid s_t)
}
$$

它用来处理“rollout 是旧策略生成、更新时模型已经是新策略”的概率分布差异。

GRPO 还包括：

```text
分组采样
+ reward
+ group-relative advantage
+ policy update
```

所以：

```text
importance sampling
⊂ GRPO 的更新工具之一

GRPO
= 一整套 RL 训练算法
```

### Q3：completion 是什么？

假设：

```text
Prompt:
Solve 2 + 3.

Model:
First calculate ... \boxed{5}
```

则：

```text
Prompt
= 已知输入

Completion
= 模型从 prompt 后开始生成的全部 token
```

它可能同时包含：

```text
Reasoning
Tool Call
Final Answer
```

### Q4：为什么 `build_grpo_datum()` 要右移？

自回归模型的目标是：

$$
x_t
\rightarrow
x_{t+1}
$$

例如：

```text
prompt     = [x1, x2, x3]
completion = [y1, y2]

model_input = [x1, x2, x3, y1]
target      = [ 0,  0, y1, y2]
```

关键位置：

```text
输入 x3
→ 预测 y1

输入 y1
→ 预测 y2
```

所以不是“prompt 第一个 token 不需要预测”这么简单，而是：

> **policy loss 只关心模型生成的 completion token。prompt 内部即使存在 next-token 关系，也不是 RL 要优化的动作，因此对应 loss 输入被 mask/占位。**

### Q5：为什么 prompt loss 区长度是 `len(prompt)-1`？

因为第一个要训练的动作是 completion 的第一个 token \(y_1\)。

在 causal LM 对齐中：

```text
prompt 最后一个 token
```

会作为输入状态的一部分，用于预测：

```text
completion 第一个 token
```

因此在 target 序列中，需要为 prompt 内部预测位置保留：

$$
m-1
$$

个占位，再接上 completion 的 \(T\) 个真实 target。

### Q6：为什么 old logprob 必须在 rollout 时保存？

因为策略更新使用：

$$
\frac{
\pi_{\theta}(a_t\mid s_t)
}{
\pi_{\theta_{\mathrm{old}}}(a_t\mid s_t)
}
$$

如果训练后再重新计算 denominator，它已经不是 rollout 时的旧策略概率，importance ratio 就失去意义。

### Q7：退化组是什么？

若同一题采样 \(G\) 条回答：

```text
[1,1,1,1]
```

或：

```text
[0,0,0,0]
```

则组内 reward 没有差异，标准化后的 advantage 都接近 0。

这类 group 无法告诉模型：

> “同一道题中哪种生成方式更好”。

因此通常跳过或动态重采样。

### Q8：为什么 sampler 刷新是 on-policy 约束？

假设第 \(k\) 步更新了参数：

$$
\theta_k
\rightarrow
\theta_{k+1}
$$

下一批数据最好来自：

$$
\pi_{\theta_{k+1}}
$$

而不是继续使用旧的：

$$
\pi_{\theta_k}
$$

否则训练数据与当前 policy 的分布差越来越大，逐渐变成 off-policy。

### Q9：OPD 的 Teacher 是怎么“逐 token 打分”的？

Teacher 不自己生成另一条答案，而是读取：

```text
prompt
+
Student completion
```

然后在每一个 Student token 位置计算：

$$
\log \pi_T(y_t\mid x,y_{<t})
$$

因此 Student 的每个动作都有 Teacher 信号。

### Q10：为什么 Teacher 和 Student tokenizer 要兼容？

OPD 是逐 token 对齐：

```text
Student 第 t 个 token
↔ Teacher 对同一个 token 的概率
```

如果同一文本被两个 tokenizer 切成不同 token 序列，位置 \(t\) 就不再表示同一个动作，逐 token KL/advantage 会失效。

### Q11：reverse KL 在 OPD 里怎么变成 advantage？

$$
d_t =
\log\pi_{\mathrm{old}}(y_t\mid s_t) -
\log\pi_T(y_t\mid s_t)
$$

再设：

$$
A_t =
-\beta d_t
$$

当 Teacher 比 Student 更偏好该 token 时，Student 获得正向调整；反之被压低。

### Q12：OPD 与传统离线知识蒸馏有什么不同？

| 项目 | 离线蒸馏 | OPD |
|---|---|---|
| 轨迹来源 | 固定数据集 | Student 当前策略 |
| Teacher 评分状态 | 数据集中状态 | Student 实际访问状态 |
| Distribution Shift | 较明显 | 更小 |
| 信号 | Teacher 分布 | Teacher 分布 |
| 是否 on-policy | ❌ | ✅ |

### Q13：Search-R1 为什么不是普通 RAG？

RAG：

```text
系统决定：
先检索 → 把结果给模型
```

Search-R1：

```text
模型决定：
要不要搜？
搜什么？
拿到结果后还要不要再搜？
什么时候最终回答？
```

所以 Search-R1 学习的是：

> **检索策略本身。**

### Q14：什么是 observation mask？

工具返回内容不是模型自己生成的 action。

例如：

```text
Assistant: search("...")
Tool: 搜索结果文本
Assistant: Answer: ...
```

其中：

```text
search(...)       → policy action，要训练
Tool observation  → 环境输入，不训练
Answer            → policy action，要训练
```

因此 observation token：

```text
old_logprob = 0
advantage   = 0
```

但仍保留在上下文，供后续 Assistant token 使用。

### Q15：ReTool 如何让模型学会“代码报错后修正”？

轨迹可能是：

```text
生成代码
→ NameError
→ stderr 进入上下文
→ 再生成修正版代码
→ 得到正确结果
→ 最终答案正确
```

最终 reward 为正后，整条成功动作链中的：

```text
第一次工具选择
第二次修正代码
最终答案
```

共享正 advantage，因此模型逐渐学会利用错误反馈自我修正。

### Q16：PyTRIO 和 Verl 的区别？

| 项目 | PyTRIO | Verl |
|---|---|---|
| 定位 | 低门槛 Agentic-RL 实验 | 大规模生产级 RL |
| 基础设施 | 服务端/托管抽象较多 | 自己管理 GPU 集群与后端 |
| API | 简洁 Python SDK | 完整分布式 RL 工程栈 |
| 适合 | 快速验证算法 | 大规模、高吞吐训练 |
| 工程复杂度 | 较低 | 较高 |

### Q17：FSDP 与 ZeRO 的关系？

两者都解决模型状态冗余。

- **ZeRO** 是 DeepSpeed 的分阶段体系：
  - Stage 1：优化器状态；
  - Stage 2：+ 梯度；
  - Stage 3：+ 参数。
- **FSDP** 是 PyTorch 原生的全分片数据并行，核心目标接近 ZeRO-3。

它们属于**系统/分布式训练层**，不是 RL 算法。

---

# 第八章整体结构

```text
8.1 GRPO
    ↓
学会 rollout / reward / advantage / policy update

8.2 OPD
    ↓
把 reward 换成 Teacher 逐 token 反馈

8.3 Search-R1
    ↓
把单轮回答扩展到多轮搜索环境

8.4 ReTool
    ↓
把搜索环境换成代码解释器
```

统一主线：

$$
\text{Current Policy}
\rightarrow
\text{Rollout}
\rightarrow
\text{Feedback}
\rightarrow
\text{Advantage}
\rightarrow
\text{Policy Update}
\rightarrow
\text{New Policy}
$$

---

# 1. PyTRIO 与 Verl

## 1.1 PyTRIO

第八章全部示例使用 PyTRIO。

可以把它理解成：

> **把“本地算法控制”与“远端模型采样、前向反向、优化和权重管理”分离的 RL 后训练 SDK。**

典型 API：

```python
# 简化示例
client = ServiceClient()
trainer = client.create_lora_training_client(...)
sampler = trainer.save_weights_and_get_sampling_client()

future = trainer.forward_backward(datums, loss_fn="importance_sampling")
trainer.optim_step(adam_params)
```

重点不是自己实现 GPU 集群调度，而是研究：

```text
怎样 rollout
怎样设计 reward
怎样构造 advantage
怎样对齐 token
```

## 1.2 Verl

Verl 更偏向生产级大规模 RL 训练：

```text
PPO / GRPO / DAPO 等算法
+
FSDP / Megatron-LM 等训练后端
+
vLLM / SGLang 等推理后端
```

适合有 GPU 集群并追求吞吐、扩展性和复杂并行策略的场景。

---

# 2. GRPO

## 2.1 从语言模型到 RL

LLM 策略：

$$
\pi_\theta(y\mid x) =
\prod_{t=1}^{T}
\pi_\theta(y_t\mid x,y_{<t})
$$

LLM 中的“动作”就是下一个 token。

GRPO 的关键：

```text
同一道题
↓
采样 G 条回答
↓
每条得到 reward
↓
组内相对标准化
↓
得到 advantage
↓
更新策略
```

## 2.2 Group Relative Advantage

$$
A_i =
\frac{
r_i-\mathrm{mean}(r_1,\ldots,r_G)
}{
\mathrm{std}(r_1,\ldots,r_G)+\varepsilon
}
$$

它把“题目难度”抵消掉，重点比较：

> 同一道题里，哪种生成方式相对更好。

## 2.3 Importance Sampling 与 PPO

简化 importance sampling loss：

$$
L_{\mathrm{IS}}
\propto
-\rho_t A_t
$$

PPO：

$$
L_{\mathrm{PPO}} =
-\min
\left(
\rho_tA_t,
\mathrm{clip}(\rho_t,1-\epsilon,1+\epsilon)A_t
\right)
$$

区别：

```text
Importance Sampling
→ 直接用 ratio * advantage

PPO
→ 再给 ratio 加“限速器”
```

### 粘贴文本中的 CISPO 补充

CISPO 属于第八章正文之外的扩展讨论。可简单理解为：

> 对 importance ratio 做约束/裁剪处理，但避免 PPO 在某些越界 token 上直接失去有效梯度，从而改善 RL 训练稳定性。

正文代码只保留：

```text
importance_sampling
ppo
```

两个 PyTRIO 内置 loss。

---

# 3. GRPO 关键代码解释

## 3.1 Rollout 数据结构

```python
# 简化版
@dataclass
class RolloutSample:
    tokens: list[int]
    logprobs: list[float]   # rollout 时旧策略概率
    text: str
    reward: float
    advantage: float
```

核心：

> **必须保存 rollout 时的 old logprob。**

## 3.2 `run_rollout_group()` 核心逻辑

```python
# 简化伪代码
result = sampler.sample(
    prompt=prompt_tokens,
    num_samples=group_size,
)

samples = []
for seq in result:
    reward = grade(seq.text, ground_truth)
    samples.append({
        "tokens": seq.tokens,
        "old_logprobs": seq.logprobs,
        "reward": reward,
    })

rewards = [x["reward"] for x in samples]
mean = average(rewards)
std = standard_deviation(rewards)

for sample in samples:
    sample["advantage"] = (sample["reward"] - mean) / (std + 1e-8)
```

这是 GRPO 的最核心代码。

## 3.3 `build_grpo_datum()` 的右移

```python
# 简化版
obs_len = len(prompt_tokens) - 1

model_input = prompt_tokens + completion[:-1]
target = [0] * obs_len + completion
old_logprob = [0.0] * obs_len + completion_old_logprobs
advantage = [0.0] * obs_len + [A] * len(completion)
```

核心理解：

```text
Prompt
= State / Observation

Completion
= Policy Action
```

RL loss 只训练模型自己的动作。

## 3.4 每 step 刷新 sampler

```python
# 简化版
for step in range(steps):
    sampler = trainer.save_weights_and_get_sampling_client()
    trajectories = rollout(sampler)
    train(trajectories)
```

这是 on-policy 的关键工程动作。

---

# 4. OPD：On-Policy Distillation

## 4.1 为什么需要 OPD？

传统离线蒸馏：

```text
固定数据
→ Teacher 标注
→ Student 学
```

问题：

Student 真正生成时可能进入训练数据没有覆盖的错误状态。

OPD：

```text
Student 自己 rollout
↓
Teacher 沿 Student 真实轨迹打分
↓
Student 更新
```

Teacher 能覆盖 Student 当前实际访问的状态。

## 4.2 Teacher 必须评价同一条 Student 轨迹

关键代码逻辑：

```python
# 简化版
all_ids = prompt_ids + student_completion_ids
all_teacher_logprobs = teacher.compute_logprobs(all_ids)

teacher_completion_logprobs = all_teacher_logprobs[len(prompt_ids):]
```

Teacher 不生成自己的答案。

它做的是：

> “如果我处在 Student 当前这个上下文，我会给 Student 刚生成的 token 多大概率？”

## 4.3 Reverse KL Advantage

$$
d_t =
\log\pi_{\mathrm{Student}}(y_t\mid s_t) -
\log\pi_{\mathrm{Teacher}}(y_t\mid s_t)
$$

$$
A_t=-\beta d_t
$$

因此 OPD 的 advantage 是：

```text
token 1 → A1
token 2 → A2
token 3 → A3
...
```

而 GRPO：

```text
整条 completion
→ 同一个轨迹 advantage A
→ 复制到所有 completion token
```

## 4.4 Tokenizer 对齐

```python
student_ids = student_tokenizer.encode(text)
teacher_ids = teacher_tokenizer.encode(text)

assert student_ids == teacher_ids
```

这是逐 token Teacher 反馈成立的前提之一。

---

# 5. Search-R1

## 5.1 Search-R1 与 RAG

传统 RAG：

```text
Query
→ 固定 Retriever
→ Context
→ LLM
```

Search-R1：

```text
LLM
→ 判断是否搜索
→ 生成 search query
→ Search Environment
→ Observation
→ LLM
→ 可能再次搜索
→ Final Answer
```

它训练的是“搜索策略”。

## 5.2 搜索工具协议

```python
# 简化版
SEARCH_TOOL = {
    "name": "search",
    "parameters": {
        "query": "string"
    }
}
```

模型的动作可能是：

```text
search("The Little Prince author")
```

环境返回：

```text
Antoine de Saint-Exupéry ...
```

然后模型再决定下一步。

## 5.3 多轮状态机

```python
# 简化伪代码
while not trajectory.done:
    assistant = sample(current_prompt)

    if assistant.is_search_call:
        result = search(assistant.query)
        append_tool_observation(result)

    elif assistant.is_final_answer:
        trajectory.done = True

    else:
        trajectory.invalid = True
```

这就是 Agentic RL 的 rollout。

## 5.4 Observation Mask

```python
# 简化版
full_tokens += tool_observation_tokens
old_logprobs += [0.0] * len(tool_observation_tokens)
advantages += [0.0] * len(tool_observation_tokens)

full_tokens += assistant_tokens
old_logprobs += assistant_old_logprobs
advantages += [trajectory_advantage] * len(assistant_tokens)
```

工具 observation：

```text
进入 Context
但不进入 Policy Loss
```

## 5.5 Outcome Reward

Search-R1 只看最终短答案：

```text
格式正确 + 精确匹配 → 1
格式正确 + 答案错误 → 0
格式错误              → 负奖励
```

然后同题 group 做相对 advantage。

因此一次正确答案可以强化：

```text
什么时候搜索
搜索词怎么写
什么时候继续搜
什么时候停止搜索
最终答案怎么输出
```

---

# 6. ReTool

## 6.1 ReTool 是什么？

ReTool 把 RL 环境变成：

> **Python Code Interpreter**

轨迹：

```text
Question
↓
LLM 生成代码
↓
Python Sandbox
↓
stdout / stderr
↓
LLM 继续推理或修正
↓
最终 \boxed{answer}
↓
Reward
```

## 6.2 Code Tool Schema

```python
# 简化版
CODE_TOOL = {
    "name": "code_interpreter",
    "parameters": {
        "code": "string"
    }
}
```

## 6.3 自我修正

```text
第一次代码：
print(sum(values))

Tool:
NameError

第二次代码：
values = [...]
print(sum(values))

Tool:
128

Assistant:
\boxed{128}
```

RL 不需要人工告诉模型：

```text
“遇到 NameError 时应该重写变量”
```

只需要最终任务成功 reward，成功轨迹就会被强化。

## 6.4 Sandbox

核心职责：

```text
执行模型代码
限制运行时间
捕获 stdout
捕获 stderr
捕获 timeout
把结果变成 observation
```

真实生产中必须使用严格隔离环境，不能直接在宿主机执行不可信模型代码。

## 6.5 ReTool 的 Feedback Mask

与 Search-R1 相同：

```text
模型生成代码
→ policy token，训练

stdout / stderr
→ environment observation，不训练

模型后续推理和答案
→ policy token，训练
```

---

# 7. 粘贴文本中的扩展概念整理

## 7.1 DAPO

DAPO 属于 GRPO/PPO 路线的改进，重点包括：

```text
Clip-Higher
Dynamic Sampling
Token-level Policy Gradient
Overlong Reward Shaping
```

特别是 Dynamic Sampling：

> 跳过/重采样 reward 全相同的无效组，提高有效 rollout 的比例。

## 7.2 GSPO

GSPO 将策略优化的关键比率从：

```text
token level
```

提升到：

```text
sequence level
```

目标是减小长序列 RL 中 token-level ratio 的高方差问题。

## 7.3 OPSD

OPSD：

> **On-Policy Self-Distillation**

和 OPD 的区别是：

```text
OPD：
外部 Teacher

OPSD：
固定/特权信息增强的 Self Teacher
```

## 7.4 ALFWorld

ALFWorld 是文本化具身环境。

Agent 需要完成：

```text
寻找物体
移动物体
使用物体
完成家庭任务
```

常用于研究：

```text
长轨迹
稀疏 reward
Agent credit assignment
```

## 7.5 FSDP

FSDP：

> **Fully Sharded Data Parallel**

属于 PyTorch 分布式训练系统。

核心：

```text
参数
梯度
优化器状态
```

在数据并行组中分片。

### FSDP vs ZeRO

| 项目 | FSDP | ZeRO |
|---|---|---|
| 生态 | PyTorch 原生 | DeepSpeed |
| 核心 | 全分片 DP | Stage 1/2/3 分片体系 |
| 与 ZeRO-3 | 目标接近 | ZeRO-3 本身 |
| 是否 RL 算法 | ❌ | ❌ |

## 7.6 关于 SAR-OPD / IDT-OPD

粘贴文本中提到了 **Medical OPD 的 SAR-OPD 与 IDT-OPD**，但所附文本本身也指出没有可靠公开定义并进行了推测。

因此本文不把推测当成事实。

> 如果后续需要学习 Medical OPD，应该直接以对应论文/仓库对 SAR-OPD、IDT-OPD 的正式定义为准。

---

# 8. 第八章 86 个代码片段逐段解释

> 下面按 Happy-LLM 第八章源码出现顺序列出。这里解释的是**每个代码块在完整训练系统中承担什么职责**。

| # | 所属部分 | 类型 | 代码片段作用与解释 |
|---:|---|---|---|
| 1 | 章首环境 | `bash` | 创建 Python 3.13 虚拟环境、安装第八章依赖并执行 `trio login`；作用是准备 PyTRIO 实验环境。 |
| 2 | 章首环境 | `python` | 创建 `ServiceClient` 并查询当前 PyTRIO 服务支持的模型列表；正式实验前确认 Student/Teacher 模型是否可用。 |
| 3 | GRPO：奖励 | `python` | 实现 `\boxed{}` 抽取、答案归一化和规则判题；把数学题最终答案转成可验证的 0/1 reward。 |
| 4 | GRPO：配置 | `python` | 限定本章 GRPO 示例支持的策略更新损失为 `importance_sampling` 和 `ppo`。 |
| 5 | GRPO：RolloutSample | `python` | 定义单条 rollout 的训练记录：completion tokens、旧策略 logprobs、文本、reward、advantage。 |
| 6 | GRPO：奖励函数 | `python` | 带注释地重述答案抽取、GSM8K 标准答案解析和判题逻辑；说明 reward 如何从文本变成标量。 |
| 7 | GRPO：Prompt | `python` | 使用 Chat Template 与 few-shot 前缀构造 prompt，并编码成 token IDs；保证训练和采样使用统一格式。 |
| 8 | GRPO：run_rollout_group | `python` | 对同一个 prompt 一次采样 `group_size` 个 completion，保存旧 logprob、计算规则 reward，再做组内 reward 标准化得到 advantage。 |
| 9 | GRPO：build_grpo_datum | `python` | 把 prompt 与 completion 右移对齐成 PyTRIO `Datum`，并把 prompt 区域的 target/logprob/advantage 用 0 占位，只让 completion 参与策略损失。 |
| 10 | GRPO：Token 对齐示例 | `text` | 用极小 token 序列展示 input、target、old_logprob、advantage 如何错位一位并保持等长。 |
| 11 | GRPO：策略损失 | `python` | 调用 `forward_backward(datums, loss_fn=...)`；同一批 GRPO 数据可切换 importance sampling 或 PPO clipping。 |
| 12 | GRPO：客户端初始化 | `python` | 创建 LoRA training client、Tokenizer、采样参数和 Adam 参数；把训练端、采样端和优化器参数准备好。 |
| 13 | GRPO：主循环 | `python` | 串联 batch 选择、sampler 刷新、rollout、退化组过滤、Datum 构造和训练提交，是同步版 GRPO 的核心训练循环。 |
| 14 | GRPO：优化 | `python` | 提交 `optim_step` 并等待前向反向与优化 future 完成；体现 PyTRIO 云端异步任务、本地显式同步。 |
| 15 | GRPO：保存 | `python` | 把训练后的权重保存成 sampler 可加载的权重版本，供后续 rollout/评测使用。 |
| 16 | GRPO：同步运行 | `bash` | 最小一步同步 GRPO 试跑命令；用于验证数据、采样、reward、Datum 和更新链路能否跑通。 |
| 17 | GRPO：异步运行 | `bash` | 异步版 GRPO 试跑；算法数据不变，只并发处理多个 prompt 的 rollout 提高吞吐。 |
| 18 | OPD：Teacher logprob | `python` | Teacher 在 Student 的同一条 prompt+completion 轨迹上计算 completion token 的 logprob；为逐 token 蒸馏提供信号。 |
| 19 | OPD：Tokenizer 对齐 | `python` | 将同一文本分别用 Student/Teacher Tokenizer 编码并断言相同；防止 token 对不齐导致逐 token KL 无意义。 |
| 20 | OPD：数据路径 | `python` | 定义 DeepMath-103K 本地目录和默认 shard 数，集中管理数据文件位置。 |
| 21 | OPD：加载 DeepMath | `python` | 按 shard 下载/读取 DeepMath Parquet 数据、拼接并采样训练样本；把大数据集分片流式管理。 |
| 22 | OPD：Prompt | `python` | 把数学题与后缀拼成 user message，再用 Chat Template 构建 Student rollout prompt。 |
| 23 | OPD：Student/Teacher 客户端 | `python` | 创建 Student LoRA training client 与独立 Teacher sampling client；Student 可训练，Teacher 只打分。 |
| 24 | OPD：完整 Teacher 评分 | `python` | 计算整条 prompt+completion 的 logprob，再切出 completion 区间并校验长度与空值。 |
| 25 | OPD：build_opd_datum | `python` | 沿用 GRPO 的右移 token schema，但 advantage 改为每个 token 独立的 reverse-KL advantage。 |
| 26 | OPD：sampler 刷新 | `python` | 按 `sampler_refresh_steps` 刷新 Student sampler；默认每 step 刷新以保持 on-policy。 |
| 27 | OPD：rollout + reverse KL | `python` | Student 生成 completion，记录旧 logprob；Teacher 沿同一轨迹评分，并据此生成逐 token advantage。 |
| 28 | OPD：更新 | `python` | 将 OPD Datums 用 importance sampling 做前向反向，再执行 Adam 更新；实现 Student 参数学习。 |
| 29 | OPD：指标 | `python` | 汇总 completion token 数、吞吐、reverse-KL 等训练指标，用于诊断蒸馏是否有效。 |
| 30 | OPD：保存 | `python` | 保存训练后的 Student 权重，供后续 sampling/evaluation。 |
| 31 | OPD：同步运行 | `bash` | 最小同步 OPD 试跑命令，限制 shard、sample size 和生成长度以降低成本。 |
| 32 | OPD：异步运行 | `bash` | 异步 OPD 试跑；通过更大 batch/group 并发 Student rollout 和 Teacher 评分。 |
| 33 | Search-R1：文件链 | `text` | 展示 `prepare_data → protocol → search → rollout → train → eval` 的模块职责，先建立整体代码地图。 |
| 34 | Search-R1：样本格式 | `json` | 给出统一 JSONL 样本结构：id、question、answers、data_source。 |
| 35 | Search-R1：数据归一化 | `python` | 从原始数据中抽取问题和参考答案，过滤无效样本并统一字段格式。 |
| 36 | Search-R1：SearchExample | `python` | 用不可变 dataclass 表示一条搜索问答样本，明确 rollout 所需字段。 |
| 37 | Search-R1：多跳示例 | `text` | 展示模型先搜作者、再根据 observation 搜出生国家的多轮搜索轨迹。 |
| 38 | Search-R1：SEARCH_TOOL | `python` | 定义 search Function Calling JSON Schema，规定工具名、描述与 `query` 参数。 |
| 39 | Search-R1：解析 Assistant | `python` | 解析模型输出中的 `<tool_call>`，区分 answer、search call 和 invalid 格式。 |
| 40 | Search-R1：搜索结果数据类 | `python` | 统一定义 `SearchItem` 与 `SearchResult`，把不同后端返回转换成共同结构。 |
| 41 | Search-R1：后端工厂 | `python` | 根据 backend 配置创建 DeepSeek/Wikipedia 等搜索客户端，并统一 timeout。 |
| 42 | Search-R1：Wikipedia 请求 | `python` | 构造 Wikipedia API 查询、解析候选页面和摘要，并转成统一 SearchResult；它是真实环境交互层。 |
| 43 | Search-R1：Observation 文本格式 | `text` | 规定搜索结果如何格式化成 `[序号] Title/Content/Source/URL`，再追加回对话上下文。 |
| 44 | Search-R1：轨迹数据结构 | `python` | 定义 AssistantTurn、Trajectory 等对象，保存每轮 prompt/completion/logprob、reward、搜索次数和完成状态。 |
| 45 | Search-R1：异步采样 | `python` | 批量构造多个 SamplingParams 与 sampling future，并发生成多条 Assistant turn。 |
| 46 | Search-R1：consume_assistant | `python` | 把一轮模型输出写入轨迹并解析：若是搜索调用则生成 PendingSearch，若是最终答案则结束轨迹。 |
| 47 | Search-R1：并发搜索 | `python` | 用线程池/并发配置执行所有 PendingSearch，避免搜索后端串行成为 rollout 瓶颈。 |
| 48 | Search-R1：finish_search | `python` | 将 SearchResult 截断/格式化成 tool observation，追加 messages，并检查轨迹 token 预算。 |
| 49 | Search-R1：首轮请求 | `python` | 为每个根问题构造第一次 sampling request；`group_size` 在首轮展开多个同题轨迹。 |
| 50 | Search-R1：首轮 rollout | `python` | 并发采样首轮 Assistant 输出，解析搜索调用并统一执行 pending searches。 |
| 51 | Search-R1：多轮状态机 | `python` | `while` 循环持续执行“生成→解析→搜索→observation→再生成”，直到答案、非法格式或预算耗尽。 |
| 52 | Search-R1：build_next_prompt | `python` | 重新渲染 canonical Chat Template，并处理工具消息，使新 prompt 是旧 token 轨迹的严格前缀扩展。 |
| 53 | Search-R1：连续性断言 | `python` | 验证下一轮 prompt 以已有 full_tokens 为前缀；若失败说明重新 tokenize 后轨迹已错位，应立即报错。 |
| 54 | Search-R1：答案协议 | `text` | 规定最终回答必须是单行 `Answer: <short answer>`，便于规则判题。 |
| 55 | Search-R1：Reward | `python` | 归一化预测/参考短答案，做 exact match，并对格式错误、答案错误、答案正确设置分档 reward。 |
| 56 | Search-R1：组内 Advantage | `python` | 按 question 分组，用组均值/标准差计算相对优势，并识别 reward 全相同的退化组。 |
| 57 | Search-R1：Observation Mask | `python` | 搜索 observation token 的 old_logprob 和 advantage 写 0；Assistant 自己生成的 token 才写真实值。 |
| 58 | Search-R1：build_datum | `python` | 把多轮 prompt、tool observation、assistant completion 合成连续 token 轨迹并构造 PyTRIO Datum，是 Search-R1 训练对齐核心。 |
| 59 | Search-R1：训练 Step | `python` | 从 batch 创建最新 sampler、执行 rollout、算 reward/advantage、构建 datums，再调用训练客户端更新策略。 |
| 60 | Search-R1：Micro-batch 权重 | `python` | 按 micro-batch 样本数/总样本数缩放 advantage，使拆分 micro-batch 后总体 loss 仍等价于全局均值。 |
| 61 | Search-R1：评测配置 | `python` | 构造 group_size=1 等评测 RolloutConfig，固定搜索次数、轨迹长度和后端参数。 |
| 62 | Search-R1：准备数据 | `bash` | 运行 `prepare_data.py` 下载/整理 NQ、HotpotQA 等数据。 |
| 63 | Search-R1：训练命令 | `bash` | 一步最小训练命令，指定问题数、group size、最大搜索次数、搜索后端和并发度。 |
| 64 | Search-R1：Base 评测 | `bash` | 用同一搜索环境评测 Base Model，并把逐题结果写成 JSONL。 |
| 65 | Search-R1：Checkpoint 评测 | `bash` | 通过 `--model-path trio://...` 评测训练后 checkpoint，和 Base Model 做公平对比。 |
| 66 | ReTool：文件链 | `text` | 展示 `prepare_data → protocol → sandbox → rollout → reward → train` 的代码职责。 |
| 67 | ReTool：代码工具示例 | `text` | 展示模型生成 code_interpreter 调用，环境返回数值，模型再输出 `\boxed{}` 最终答案。 |
| 68 | ReTool：自我修正示例 | `text` | 展示首次代码报 NameError 后，模型根据 stderr 重写完整代码并再次调用工具。 |
| 69 | ReTool：数据归一化 | `python` | 从训练数据中解析 prompt 与 reward_model 信息，去掉旧模板并统一成 MathExample。 |
| 70 | ReTool：CODE_TOOL | `python` | 定义 `code_interpreter` Function Calling Schema，并强调每次执行是 fresh process。 |
| 71 | ReTool：解析 Assistant | `python` | 解析 code tool call / final answer / invalid 输出，和 Search-R1 共用类似工具协议。 |
| 72 | ReTool：执行结果接回轨迹 | `python` | 构造下一轮 prompt，把代码 execution result 作为 tool observation 追加，同时保持 token 前缀连续。 |
| 73 | ReTool：Outcome Reward | `python` | 从最后一个 `\boxed{...}` 中正确解析嵌套括号并判题，为整条代码轨迹产生结果奖励。 |
| 74 | ReTool：PPO 配置 | `python` | 配置非对称 PPO clip 阈值，并用 PyTRIO `loss_fn='ppo'` 做策略更新。 |
| 75 | ReTool：LocalPythonSandbox | `python` | 以子进程运行模型生成代码，设置 CPU/超时/环境限制并捕获 stdout、stderr、return code；是代码环境核心。 |
| 76 | ReTool：Tool Result 格式化 | `python` | 把成功 stdout、stderr、异常和 timeout 转成模型可读的 tool message，并做内容清洗。 |
| 77 | ReTool：轨迹数据结构 | `python` | 定义 AssistantTurn、Trajectory，保存多轮代码调用和答案生成状态。 |
| 78 | ReTool：begin_advance | `python` | 解析一轮 Assistant 结果：若请求 code_interpreter，则建立 PendingExecution；若给最终答案则结束。 |
| 79 | ReTool：并发执行代码 | `python` | 异步并发执行多个 PendingExecution，再把每个结果写回对应轨迹，提升 rollout 环境吞吐。 |
| 80 | ReTool：训练 rollout | `python` | 每 step 保存当前权重生成 sampler，再执行带代码工具的 rollout_batch。 |
| 81 | ReTool：Feedback Mask | `python` | 构造多轮 Datum：interpreter observation 的 loss 输入置 0，Assistant 生成的代码/推理/答案共享轨迹 advantage。 |
| 82 | ReTool：PPO 更新 | `python` | 把 Datums 打包成 micro-batches，做全局均值权重修正，逐 micro-batch 调 `forward_backward(..., 'ppo')`，最后 `optim_step`。 |
| 83 | ReTool：评测配置 | `python` | 按 text-only/retool 模式配置 `val_n`、代码调用上限、轨迹预算、temperature/top-p 等。 |
| 84 | ReTool：准备数据 | `bash` | 运行数据准备脚本。 |
| 85 | ReTool：最小训练 | `bash` | 一步最小 ReTool 训练命令，限制题数、组大小、代码调用次数和 sandbox workers。 |
| 86 | ReTool：对照评测 | `bash` | 分别用 text-only 与 retool 模式评测，并固定 val_n、temperature、top-p；用于测量工具环境的净收益。 |

---

# 9. 四套实践的代码结构对比

| 实践 | Rollout | Feedback | Advantage | 环境 | Loss |
|---|---|---|---|---|---|
| GRPO | 同题多 completion | 规则 reward | Group Relative | 数学判题器 | Importance Sampling / PPO |
| OPD | Student completion | Teacher token logprob | Reverse-KL token advantage | Teacher | Importance Sampling |
| Search-R1 | 多轮搜索轨迹 | 最终答案 reward | Group Relative | Search Engine | Importance Sampling |
| ReTool | 多轮代码轨迹 | 最终答案 reward | Group Relative | Python Sandbox | PPO |

---

# 10. 四者的统一理解

## GRPO

```text
模型生成
→ 判题器反馈
→ 哪条回答相对更好？
```

## OPD

```text
Student 生成
→ Teacher 逐 token 反馈
→ 每个动作应该向 Teacher 靠多少？
```

## Search-R1

```text
模型生成搜索动作
→ 搜索环境返回 observation
→ 多轮交互
→ 最终答案是否正确？
```

## ReTool

```text
模型生成代码动作
→ Python 返回 stdout/stderr
→ 模型自我修正
→ 最终答案是否正确？
```

---

# 11. 一页速记

```text
第八章
= Agentic RL

统一闭环：
Policy
→ Rollout
→ Feedback
→ Advantage
→ Datum
→ forward_backward
→ optim_step
→ Refresh Sampler

GRPO：
同题多采样
Group Reward
Group Relative Advantage
不需要 Value Model

importance_sampling：
旧策略数据
→ 新策略更新
→ ratio 纠偏

PPO：
importance ratio
+ clip 限制更新幅度

completion：
Prompt 后模型生成的全部 Token

build_grpo_datum：
右移一位
Prompt Mask
Completion Train

On-Policy：
更新模型后
用新策略重新 Rollout

OPD：
Student 自己生成
Teacher 沿同一轨迹逐 Token 打分
Reverse KL
稠密 Advantage

Search-R1：
模型自主决定是否搜索
Search Observation 不算 Loss
最终答案 Reward 训练搜索策略

ReTool：
模型自主写代码
stdout/stderr 作为 Observation
可以报错后重试
最终答案 Reward 训练代码使用策略

PyTRIO：
低门槛 Agentic-RL SDK

Verl：
大规模工业 RL 框架

DAPO：
GRPO/PPO 稳定性与采样效率改进

GSPO：
Sequence-level Policy Optimization

OPSD：
On-Policy Self-Distillation

FSDP：
PyTorch 全分片数据并行

ALFWorld：
文本化具身 Agent 环境
```

---

# 12. 最后总结

第八章可以用三个层次理解：

### 第一层：先学会 RL 的基本数据流

```text
Rollout
Reward
Advantage
Old Logprob
Policy Update
```

### 第二层：把稀疏 Reward 换成更丰富的 Feedback

```text
GRPO：
最终 Reward

OPD：
Teacher Token Probability
```

### 第三层：把单轮回答变成 Agent 环境轨迹

```text
Search-R1：
Search Tool

ReTool：
Code Interpreter
```

最终一句话：

> **第八章真正想训练的不是“某一道题的答案”，而是模型在自己真实会访问的状态上，怎样选择更好的下一步动作。GRPO 学生成策略，OPD 学 Teacher 行为，Search-R1 学搜索策略，ReTool 学代码工具策略。**

---

# 参考资料

1. Datawhale, Happy-LLM，第八章《大模型强化学习》  
   https://datawhalechina.github.io/happy-llm/

2. Datawhale Happy-LLM GitHub  
   `docs/chapter8/第八章 大模型强化学习.md`

3. 第八章配套目录  
   `docs/chapter8/grpo`  
   `docs/chapter8/opd`  
   `docs/chapter8/search-r1`  
   `docs/chapter8/retool`

4. 本文还整合了本次粘贴文本中的扩展讨论，包括 PyTRIO/Verl、DAPO、FSDP、GSPO、OPSD、ALFWorld、importance sampling、completion、GRPO Datum 对齐、on-policy 刷新、OPD Teacher logprob、Search-R1 与 RAG 的区别等。
