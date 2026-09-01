---
title: nanobot的AgentLoop和AgentRunner的区别
cover: /assets/posts/nanobot/项目架构.png
categories: agent
tags:
  - nanobot
---

| `AgentRunner` | 接收 `AgentRunSpec`；请求 Provider；汇集流式文本、reasoning 和工具调用；执行工具；追加工具结果；控制迭代、超时与最终结果                                   | 不选择 Channel，不创建业务 Session，不决定回复发到哪里 |
| :------------ | :-------------------------------------------------------------------------------------------------------------- | :---------------------------------- |
| `AgentLoop`   | 从 MessageBus 接收入站消息；解析会话键和工作区；构建 Bootstrap、Memory、Skills 与历史上下文；把配置装进 `AgentRunSpec`；保存新消息；组装 `OutboundMessage` | 不再自己实现 Provider/工具的底层迭代算法           |


```text
Channel / WebUI / CLI
        ↓
MessageBus
        ↓
AgentLoop                 业务编排层
        ↓ AgentRunSpec
AgentRunner               模型执行层
        ↓
Provider + Tools
```

`AgentLoop` 负责“这次对话在什么业务上下文中运行”。

- 从 `MessageBus` 消费入站消息。
- 决定会话键、所属 Channel、工作区和权限上下文。
- 加载 Agent 身份、Memory、Skills、历史消息、项目指令等，构造 Prompt。
- 根据配置选择模型、工具、超时等，并封装为 `AgentRunSpec`。
- 调用 `AgentRunner`。
- 将用户消息、模型回复保存回 Session。
- 将最终回复组装成 `OutboundMessage`，发送回 WebUI、CLI、Telegram 等来源。
- 处理命令、自动压缩、Dream/cron 等系统级流程。

`AgentRunner` 负责“给定完整输入后，如何稳定地跑完一次模型 Agent 回合”。

- 接收已经准备好的 `AgentRunSpec`。
- 调用 Provider，并处理流式文本、reasoning、工具调用。
- 执行模型请求的工具，并把工具结果追加回消息链。
- 继续请求模型，形成 ReAct：`模型 -> 工具 -> 工具结果 -> 模型`。
- 控制最大迭代次数、取消、超时、错误处理和最终结束原因。
- 返回模型输出、使用量、工具结果和运行元数据。

关键边界是：

```text
AgentLoop 决定「为什么跑、以谁的身份跑、上下文从哪里来、结果发到哪里」
AgentRunner 决定「这次模型和工具如何循环运行直到结束」
```

因此 `AgentRunner` 不应该知道 Telegram、WebUI、用户 Session 的业务含义；它只运行一个规范化的 `AgentRunSpec`。`AgentLoop` 也不应重新实现 Provider 流式处理或工具循环，否则同一套 ReAct 逻辑会散落到多个入口。

问题：AgentLoop就是负责完整的输入给到`AgentRunSpec`  吗？  然后`AgentRunner`   负责请求模型并回复？

基本正确，但 `AgentLoop` 不只是组装输入，`AgentRunner` 也不只是“请求一次模型”。

```text
入站消息
  -> AgentLoop：准备业务上下文与运行规格 AgentRunSpec
  -> AgentRunner：执行多轮模型/工具循环
  -> AgentLoop：持久化并投递最终回复
```

`AgentLoop`：

- 接收消息，确定 Channel、Session、工作区和权限。
- 加载历史、Memory、Skills、系统提示词等上下文。
- 按配置构造 `AgentRunSpec`。
- 调用 Runner 后保存对话和工具结果。
- 将结果包装为 `OutboundMessage`，发送回原来的 Channel。

`AgentRunner`：

- 用 `AgentRunSpec` 请求模型。
- 模型要调工具时，执行工具并把结果交还给模型。
- 重复这个过程，直到模型最终完成、超时、取消或达到迭代上限。
- 返回最终文本、reasoning、工具执行记录、token 用量和停止原因。

所以更精确地说：

- `AgentLoop` 负责“准备一次对话运行，并处理其外部生命周期”。
- `AgentRunner` 负责“执行这次运行内部的 ReAct 循环”。
- **最终把回复发送给用户的是 `AgentLoop`，不是 `AgentRunner`。**