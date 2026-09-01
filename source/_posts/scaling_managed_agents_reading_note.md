---
title: "Scaling Managed Agents: Decoupling the brain from the hands 阅读笔记"
cover: "/assets/posts/anthropic_engineering/Scaling Managed Agents.png"
categories: agent
tags:
  - anthropic_engineering
---

# Scaling Managed Agents 阅读笔记

## 一、总结

Anthropic 这篇工程博客的核心观点是：**长周期智能体系统应该将“脑子”和“手”解耦**。

其中，“脑子”指 Claude 与 agent harness，负责推理、规划和工具调度；“手”指 sandbox、MCP Server、外部工具和执行环境；session 则是完整、持久、可恢复的事件日志。

早期 agent 架构通常把 session、harness 和 sandbox 放在同一个容器中，虽然实现简单，但会带来状态丢失、调试困难、安全边界混乱、容器不可替换等问题。Managed Agents 的关键改进是：**把推理控制、执行环境和会话状态拆成稳定、可替换、可恢复的基础设施组件**。

一句话概括：

> Managed Agents 不是简单的 agent 框架，而是一种面向长周期任务的 agent runtime 设计。

---

## 二、面试速记

### 1. 这篇文章解决什么问题？

解决长周期 agent 运行中的可靠性、可恢复性、安全性和扩展性问题。

早期设计把 harness、sandbox、session 都放在同一个容器里，一旦容器崩溃，状态可能丢失；同时，用户数据、执行环境和调试入口混在一起，也会带来安全和维护问题。

### 2. Managed Agents 的核心架构是什么？

核心拆成三部分：

| 组件 | 含义 | 作用 |
|---|---|---|
| Brain | Claude + harness | 推理、规划、调用工具 |
| Hands | sandbox、MCP Server、外部工具 | 执行动作 |
| Session | 持久事件日志 | 保存完整历史，支持恢复和回看 |

### 3. “Decouple the brain from the hands”是什么意思？

意思是将模型推理层和工具执行层分离。

harness 不再和 sandbox 强绑定，而是把 sandbox 当成普通工具调用：

```text
execute(name, input) → string
```

如果 sandbox 失败，只是一次工具调用失败；系统可以重新创建 sandbox，然后继续执行任务。

### 4. 为什么 session 不能等同于上下文窗口？

因为长周期任务会超过模型上下文长度。

如果只靠摘要、裁剪或压缩上下文，会不可逆地丢失信息。Managed Agents 把 session 设计成独立于模型上下文的持久事件日志，模型可以根据需要回看历史事件，而不是把所有内容一次性塞进上下文窗口。

### 5. 这个架构带来了什么收益？

主要收益包括：

- **可靠性更强**：harness 和 sandbox 都可以失败后恢复。
- **安全性更好**：凭证不直接暴露给 sandbox 中运行的代码。
- **扩展性更强**：可以同时连接多个 sandbox、MCP Server 和外部工具。
- **启动更快**：不需要一开始就启动完整容器，只有真正需要执行时才创建 sandbox。

---

## 三、为什么需要 Managed Agents？

Anthropic 指出，agent harness 往往会编码当前模型的缺陷。例如某一代模型在接近上下文限制时可能提前结束任务，于是 harness 会加入 context reset 逻辑。

但随着模型能力提升，这些补丁可能很快过时，甚至变成额外负担。

因此，agent 系统不应该强绑定某一代模型的局限，而应该设计成可替换、可演进的架构。Managed Agents 的目标就是让底层 harness、sandbox、工具和上下文管理策略都可以随着模型能力提升而更新。

---

## 四、早期架构的问题

早期做法是把所有内容放进一个容器：

```text
session + harness + sandbox
```

这种方式适合快速实现 demo，但不适合长周期任务。

主要问题有三个：

### 1. 容器变成“宠物”

容器一旦承载 session、文件、执行状态和 harness，它就不能轻易丢弃。  
如果容器卡死或崩溃，任务状态也会受影响。

### 2. 调试和安全冲突

工程师想进入容器调试，但容器里可能包含用户文件、代码和数据。  
这会让调试权限和用户数据安全产生冲突。

### 3. 执行环境被写死

如果 Claude 只能操作当前容器中的资源，那么当企业用户希望 agent 访问自己 VPC 内部的资源时，系统就会变得复杂。  
这说明早期架构把“工具在哪里”这个假设写死了。

---

## 五、核心设计：Brain、Hands、Session 解耦

Managed Agents 的核心是将三个部分独立出来：

```text
Brain: Claude + harness
Hands: sandbox / MCP Server / tools
Session: persistent event log
```

harness 运行在 sandbox 外部，sandbox 只是一个可调用、可替换的执行端。

当 Claude 需要执行代码或操作文件时，harness 调用 sandbox：

```text
execute(name, input) → string
```

如果 sandbox 失败，harness 可以把错误返回给 Claude，也可以重新 provision 一个新的 sandbox：

```text
provision({resources})
```

这样，sandbox 不再是必须长期维护的“宠物容器”，而是可以销毁、重建、替换的基础设施。

---

## 六、Session：不是上下文窗口，而是事件日志

文章强调：**session 不等于模型上下文窗口**。

上下文窗口是模型一次推理能看到的内容，而 session 是完整、持久的历史记录。  
它应该记录用户输入、模型输出、工具调用、工具结果、错误信息和中间状态。

这样做有两个好处：

第一，任务失败后可以恢复；  
第二，模型需要历史信息时，可以回看特定事件，而不是依赖一次性摘要。

这和 event sourcing 的思想很接近：系统状态不是只保存最终结果，而是保存完整事件流。

---

## 七、安全设计：凭证不进入 sandbox

Managed Agents 还强调凭证隔离。

如果 API Key、OAuth token 或数据库密码直接暴露在 sandbox 中，Claude 生成的代码就可能通过 prompt injection 或恶意脚本读取这些凭证。

更合理的设计是：

```text
Claude → harness → proxy / vault → external service
```

凭证存储在 sandbox 外部，Claude 和 sandbox 中运行的代码都不能直接读取 token。  
这样既能让 agent 调用外部服务，也能降低凭证泄露风险。

---

## 八、Many Brains 与 Many Hands

解耦之后，系统可以支持更灵活的扩展。

### Many Brains

不同 harness 可以服务不同任务。  
当模型能力提升时，harness 可以替换，而不需要重写整个 agent 系统。

### Many Hands

Claude 可以同时调用多个执行环境，例如：

- 本地 sandbox
- 远程 sandbox
- MCP Server
- 数据库工具
- 浏览器工具
- 企业内部服务

这些工具都可以统一抽象成：

```text
execute(name, input) → string
```

这让 agent 不再依赖单一容器，而是可以操作多个外部执行端。

---

## 九、精简结论

这篇文章最重要的价值不是介绍某个具体工具，而是提出了长周期 agent 的工程化原则：

> agent 系统要可靠运行，关键不是让某个容器永不失败，而是让 brain、hands、session 都可以独立失败、独立恢复、独立替换。

Managed Agents 的本质是一个面向未来模型能力演进的 agent runtime。它通过解耦推理层、执行层和状态层，让智能体系统具备更好的可恢复性、安全性和扩展性。
