---
title: 学习nanobot的项目实现
cover: /assets/posts/nanobot/项目架构.png
categories: agent
tags:
  - nanobot
---

不要从头逐文件读。把它当作一个“消息驱动的 Agent Runtime”，沿一条真实请求链路学习，边读边断点或加日志验证。

**第一阶段：先跑起来，确认入口**
```bash
cd /home/nyc/nanobot
nanobot agent -m "你好"
# 或
nanobot gateway
```
先看 [pyproject.toml](/home/nyc/nanobot/pyproject.toml:91) 的 CLI 入口，再读 [cli/entry.py](/home/nyc/nanobot/nanobot/cli/entry.py:25)、[cli/gateway_runtime.py](/home/nyc/nanobot/nanobot/cli/gateway_runtime.py:289)。目标是回答：Gateway 启动时创建了哪些对象，谁负责关闭它们。

**第二阶段：追踪一次用户消息**
按这个顺序阅读代码：

```text
Channel/CLI
  -> InboundMessage
  -> MessageBus.inbound
  -> AgentLoop.run()
  -> AgentLoop._process_message()
  -> AgentRunner.run()
  -> Provider.chat()
  -> ToolRegistry.execute()（可选）
  -> OutboundMessage
  -> MessageBus.outbound
  -> ChannelManager
```

对应核心文件：

- [bus/events.py](/home/nyc/nanobot/nanobot/bus/events.py:20)：输入/输出消息的数据结构。
- [bus/queue.py](/home/nyc/nanobot/nanobot/bus/queue.py:8)：`MessageBus`。
- [agent/loop.py](/home/nyc/nanobot/nanobot/agent/loop.py:1230)：处理 session、上下文、投递和持久化。
- [agent/runner.py](/home/nyc/nanobot/nanobot/agent/runner.py:372)：模型与工具的循环。
- [channels/manager.py](/home/nyc/nanobot/nanobot/channels/manager.py:79)：出站消息路由。

建议在这些位置临时加 `logger.debug` 或使用断点，打印 `session_key`、`messages` 数量、工具名、最终 `OutboundMessage`。

**第三阶段：理解 Agent 为什么“记得”上下文**
先读 [session/keys.py](/home/nyc/nanobot/nanobot/session/keys.py:1)，理解不同渠道如何映射 session；再读：

- [session/manager.py](/home/nyc/nanobot/nanobot/session/manager.py:1)：会话读写、历史 replay、摘要。
- [agent/context.py](/home/nyc/nanobot/nanobot/agent/context.py:78)：system prompt 和请求 messages 如何拼装。
- [agent/memory.py](/home/nyc/nanobot/nanobot/agent/memory.py:75)：长期记忆与 Dream consolidation。

练习：新建两个不同 session，比较保存的消息历史；再把历史拉长，观察 compact/summary 如何影响下一次模型请求。

**第四阶段：理解工具和安全边界**
从一个简单工具开始，例如文件工具：

- [agent/tools/base.py](/home/nyc/nanobot/nanobot/agent/tools/base.py:159)：`Tool` 契约。
- [agent/tools/registry.py](/home/nyc/nanobot/nanobot/agent/tools/registry.py:19)：注册、Schema、参数验证与执行。
- [agent/tools/filesystem.py](/home/nyc/nanobot/nanobot/agent/tools/filesystem.py:1)：具体实现。
- [security/workspace_access.py](/home/nyc/nanobot/nanobot/security/workspace_access.py:1)：工作区访问控制。
- [security/network.py](/home/nyc/nanobot/nanobot/security/network.py:1)：SSRF 防护。

练习：自己增加一个只读 `get_current_time` 工具。完成 `name`、`description`、JSON Schema、`execute()` 和单测，再看它如何自动出现在模型 tools 定义中。

**第五阶段：按扩展方向选读**
- 新模型：`providers/base.py` → `providers/factory.py` → 一个具体 provider。
- 新聊天平台：`channels/base.py` → `channels/plugin.py` → `channels/telegram/`。
- WebUI：`channels/websocket/runtime.py` → `webui/src/lib/nanobot-client.ts` → `webui/src/App.tsx`。
- MCP：`agent/tools/mcp.py`，并关注其由 Gateway 持有生命周期。
- 自动化：`cron/service.py`、`triggers/`、`agent/cron_turns.py`。

**第六阶段：用测试反向学习**
不要只读实现。每读一个模块，就找对应测试：

```bash
rg "AgentLoop|MessageBus|ToolRegistry" tests nanobot -g '*test*.py'
```

例如先读 `tests/agent/` 的 loop/runner 测试，再回到实现。测试通常比生产代码更直接地表达“这个模块必须保证什么”。

最终目标不是记住所有文件，而是能独立回答四个问题：一条消息如何流动、上下文如何构建、工具如何被安全执行、结果如何保存并回到原渠道。掌握这四条主线后，再阅读 WebUI、MCP、子 Agent、Cron 等扩展功能会快很多。