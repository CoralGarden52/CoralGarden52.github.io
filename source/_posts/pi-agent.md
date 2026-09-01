---
title: pi-agent
cover: /assets/posts/nanobot/项目架构.png
categories: agent
tags:
  - pi-agent
---


```text
pi-coding-agent / 自定义应用
        │
        ├─ pi-tui：终端交互界面
        ├─ pi-agent-core：ReAct、工具、事件、消息队列
        ├─ pi-ai：模型、Provider、认证、流式请求
        └─ pi-server：多客户端 Session 服务与协议接入
```

| 部分 | 实现与作用 |
|---|---|
| `pi-ai` | 最底层统一模型 SDK。维护模型目录、Provider、API Key/OAuth、重试、token/cost 统计、流式事件、工具 schema 与参数校验。上层无需分别适配 OpenAI、Anthropic、Gemini 等不同协议。 |
| `pi-agent-core` | 通用 Agent 运行时。`Agent` 和 `agentLoop` 实现 `LLM -> tool calls -> tool results -> LLM` 循环；提供流式生命周期事件、并行/串行工具执行、steering/follow-up 队列、上下文转换和 session/compaction 的抽象。它不绑定 CLI、具体模型或具体 UI。 |
| `pi-coding-agent` | Pi 的成品编码 Agent。把 `pi-agent-core` 接到默认的 `read`、`write`、`edit`、`bash` 工具，加载项目指令、技能、扩展、提示模板和本地 JSONL Session；提供 interactive、print/JSON、RPC、SDK 等运行方式。 |
| `pi-tui` | 独立终端 UI 框架，不是 Agent 引擎。实现增量渲染、主/备用终端屏幕、输入编辑器、滚动、选择器、Markdown、图片、快捷键和补全。`pi-coding-agent` 用它呈现聊天、工具进度和设置界面。 |
| `pi-server`（原 `pi-orchestrator`） | 实验性的多客户端 Session 服务。通过 CBOR 协议和 Unix socket 等 listener 接收连接；管理 live session、attach/detach、prompt、steer、abort、模型切换及 snapshot 广播。它要求应用自行提供 Session 存储和 Agent runtime，不是独立的编码 Agent 服务。 |

整体来说，Pi 是一个“**可组合的本地编码 Agent 平台**”：

- `pi-ai` 解决“如何可靠调用各种模型”。
- `pi-agent-core` 解决“模型如何循环调用工具完成任务”。
- `pi-coding-agent` 解决“如何把这个循环变成可直接使用的编码助手”。
- `pi-tui` 解决“如何在终端高质量地交互”。
- `pi-server` 解决“如何让多个客户端连接和管理运行中的 Session”。

它的核心取舍是：保持 Agent core 小而通用，把编码工作流、工具、扩展、界面和服务端能力放在上层组合。相比 nanobot，Pi 更偏向本地编码工作流与可扩展 runtime；nanobot 更偏向多渠道常驻服务、长期记忆、任务调度和权限边界。