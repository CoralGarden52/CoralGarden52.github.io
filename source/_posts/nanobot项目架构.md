---
title: nanobot的项目架构
cover: /assets/posts/nanobot/项目架构.png
categories: agent
tags:
  - nanobot
---

## 总体判断

当前 `nanobot` 是一个**以 Python `asyncio` 为核心的模块化单体（modular monolith）AI Agent Runtime**，而不是微服务系统。它通过清晰的适配层支持多种入口（WebUI、TUI、CLI、聊天平台、OpenAI 兼容 API、Python SDK），但核心推理、会话、工具调用、定时任务仍在同一个 Gateway 进程内协作完成。

本地检出的版本为 **`nanobot-ai 0.3.0`**，提交为 `5b44ebdf`。规模上，Python 生产代码约 **110k LOC / 329 文件**，WebUI 生产代码约 **54k LOC / 167 文件**；因此它已不是“小脚本型 Agent”，而是一个功能较完整的个人 Agent 平台。

---

## 1. 架构全景

```text
                ┌──────────────────────────────────────────┐
                │              接入层 / Clients              │
                │ WebUI · TUI · CLI · Chat Apps · API · SDK  │
                └───────────────┬──────────────────────────┘
                                │
      Chat channel path         │             API / SDK path
                                │
┌────────────────────┐          │       ┌──────────────────────┐
│ Channels           │          │       │ AgentLoop.process_direct│
│ Telegram/WebSocket │          │       └───────────┬──────────┘
│ Discord/Slack/...  │          ▼                   │
└─────────┬──────────┘  ┌──────────────┐            │
          │             │ MessageBus   │◄───────────┘
          │ Inbound     │ inbound/out  │
          ▼             └──────┬───────┘
                    ┌──────────▼──────────┐
                    │      AgentLoop       │
                    │ session / context    │
                    │ command / hooks      │
                    │ turn orchestration   │
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │    AgentRunner       │
                    │ provider-tool loop   │
                    │ stream / retry /     │
                    │ context governance   │
                    └──────┬────────┬─────┘
                           │        │
           ┌───────────────▼──┐  ┌──▼─────────────────────┐
           │ LLM Providers     │  │ ToolRegistry           │
           │ OpenAI/Anthropic  │  │ FS / Shell / Web / MCP │
           │ Bedrock/Codex/... │  │ Cron / Image/Subagent  │
           └───────────────────┘  └────────────────────────┘
```

两条关键原则：

1. **聊天渠道经 `MessageBus` 解耦**：Channel 把 `InboundMessage` 放入队列；`AgentLoop` 处理后把 `OutboundMessage` 放回队列，由 `ChannelManager` 路由发送。
2. **API / SDK 直连 AgentLoop**：OpenAI 兼容 API 与 SDK 使用 `AgentLoop.process_direct()`，以便将 SSE/流式回调直接映射到 HTTP 或 SDK 的事件流，不必经过聊天通道队列。

---

## 2. 分层与核心职责

| 层级 | 主要目录 / 文件 | 职责 |
|---|---|---|
| 启动与组合根 | `nanobot/cli/*`、`cli/gateway_runtime.py` | 装配所有运行时组件，启动 Gateway、后台模式、健康检查、配置监听 |
| 领域核心 | `nanobot/agent/loop.py`、`runner.py` | 处理一个 Agent turn；组织上下文、模型调用、工具循环、持久化和响应 |
| 消息与事件 | `nanobot/bus/*` | `MessageBus` 负责用户可见消息；`RuntimeEventBus` 负责进程内状态事件 |
| 渠道适配 | `nanobot/channels/*` | Telegram、Discord、Slack、微信、WebSocket 等平台接入与回发 |
| 模型适配 | `nanobot/providers/*` | Provider 注册、自动选择、兼容协议、回退、模型运行时快照 |
| 工具执行 | `nanobot/agent/tools/*` | 工具发现、JSON Schema 校验、并发/互斥、MCP、文件、Shell、Web 等 |
| 状态与记忆 | `nanobot/session/*`、`agent/memory.py` | 会话历史、摘要压缩、长期记忆、Dream consolidation、provider 私有状态 |
| 自动化 | `nanobot/cron/*`、`triggers/*` | 定时任务、心跳、Dream、外部本地触发器 |
| 配置与安全 | `nanobot/config/*`、`security/*` | Pydantic 配置、环境变量解析、路径与网络安全边界 |
| UI 与协议 | `webui/`、`tui/`、`channels/websocket/` | React WebUI、OpenTUI 原生终端 UI、WebSocket/HTTP 协议 |
| 对外编程接口 | `nanobot/api/*`、`nanobot/sdk/*`、`nanobot/nanobot.py` | OpenAI 兼容 API、Python SDK、高层 Facade |

---

## 3. Gateway 是实际的组合根

`nanobot/cli/gateway_runtime.py::_run_gateway()` 是最值得优先阅读的文件。它构造并连接：

- `MessageBus`：聊天消息入站/出站队列；
- `RuntimeEventBus`：UI/运行状态订阅；
- `SessionManager`：会话持久化；
- `CronService` 与 `LocalTriggerStore`：定时与本地事件；
- `ToolRegistry` 与 `MCPProvider`：工具和 MCP 生命周期；
- `AgentLoop.from_config(...)`：核心 Agent；
- `ChannelManager`：启用的聊天渠道和 WebUI；
- 配置文件 watcher：配置变动后使模型运行时缓存失效。

Gateway 随后并发启动 Agent Loop、Channel Manager、cron、trigger queue、健康检查、WebUI 开发服务器监听等任务。

**架构含义**：这是一个“运行时宿主（runtime host）”设计。核心逻辑并不由 CLI 自己承担，CLI 的主要作用是做装配、生命周期管理和进程管理。

---

## 4. Agent 核心：`AgentLoop` 与 `AgentRunner` 的分工

这是项目最关键、也最成熟的边界。

### `AgentLoop`：面向用户 turn 的编排器

`nanobot/agent/loop.py` 当前约 **2411 行**。它处理：

- 计算有效 session key；
- 每个 session 的串行锁与运行中消息注入；
- slash command、自动化任务、取消控制；
- workspace 与请求上下文绑定；
- 会话加载、上下文构造、session summary 恢复；
- 流式输出、进度事件、最终消息投递；
- turn 记录、token 用量、会话压缩和保存；
- Dream、cron、trigger、subagent 等与 turn 的协作。

它的设计重点是：**不同 session 可并发，不同消息不能同时破坏同一 session 的上下文**。通过 session lock 与 pending queue，用户可以在 Agent 运行中继续发送信息，后续消息被作为注入而不是开启竞争的 turn。

### `AgentRunner`：面向模型的 provider/tool 循环

`nanobot/agent/runner.py` 当前约 **1710 行**，负责：

```text
构建 model messages
  → 调 Provider
  → 接收 stream / reasoning / tool calls
  → 校验并执行工具
  → 将 tool result 回填 messages
  → 再次调模型
  → 直到最终答复、出错或达到最大迭代数
```

它还承担：

- provider 重试与错误归类；
- 工具调用 Schema 校验；
- 工具执行并发批次划分；
- workspace / SSRF 违规的软失败或硬失败；
- 流式文本、推理内容、工具活动的 Hook；
- context governance：避免工具结果过大、上下文溢出或非法 tool-call 边界。

这种拆分是合理的：

- **渠道、session、命令问题**：优先查 `AgentLoop`；
- **模型、流式、工具、多轮调用问题**：优先查 `AgentRunner`。

---

## 5. 消息模型：两条 Event 通路

### A. `MessageBus`：用户可见的传输通路

`nanobot/bus/queue.py` 是一个非常轻量的内存队列：

```python
inbound: asyncio.Queue[InboundMessage]
outbound: asyncio.Queue[OutboundMessage]
```

适合将 Channel 与 Agent 解耦：

```text
Channel -> InboundMessage -> AgentLoop -> OutboundMessage -> ChannelManager -> Channel
```

`OutboundMessage` 既可代表最终回复，也可以包装流式 delta、progress、tool hint、reasoning 等事件。

### B. `RuntimeEventBus`：进程内状态同步通路

`nanobot/bus/runtime_events.py` 处理非聊天消息，例如：

- 用户输入已接受；
- turn 即将开始；
- 某个模型运行时已选定；
- turn 运行状态变化；
- 会话已持久化；
- goal 状态变化；
- 当前模型切换。

这是一个不错的分离：**用户输出**与**运行时状态**不混在同一消息协议中。WebUI 的 turn 协调器订阅后者，把状态映射成 UI 所需事件。

---

## 6. 状态、会话与长期记忆

项目的状态并不是单一数据库，而是以本地文件为中心的多层持久化。

### 会话层：`SessionManager`

`nanobot/session/manager.py` 负责：

- per-session 消息历史；
- 会话元数据、标题、模型 preset；
- provider 的私有 continuation state；
- 会话 fork / 清除 / 列表；
- 合法 tool-call 边界；
- 会话 replay 的清洗和截断。

它特别区分：

- **用户可见历史**；
- **运行时上下文**；
- **provider 私有状态**。

这避免将模型厂商私有 reasoning/continuation 信息暴露到普通聊天记录中。

### 长期记忆层：`MemoryStore` + Dream

`nanobot/agent/memory.py` 中：

- `MEMORY.md`：长期压缩记忆；
- `history.jsonl`：跨会话历史归档；
- `SOUL.md`、`USER.md`：人格与用户偏好；
- Dream：周期性读取积累历史，让模型提炼并更新长期记忆。

这是“**短期会话 replay + 长期语义记忆**”的双层记忆模式。

### 自动压缩

`AutoCompact` 与 `Consolidator` 在两类时机工作：

1. 上下文 token 接近预算时；
2. session 空闲达到 TTL 时。

这在长会话 Agent 中很必要，也使会话可持续运行，而不是历史无限增长。

---

## 7. 扩展机制

### Channels：自描述的插件包

每个 Channel 是独立包，例如：

```text
nanobot/channels/telegram/
  manifest.py
  runtime.py
  validation.py
```

`ChannelPlugin` 保存 runtime import target、依赖、管理能力、WebUI 元信息等。发现阶段只读取 descriptor，需要时再导入平台 SDK，避免未启用渠道的可选依赖影响启动。

这是一种很实用的“**轻量插件化 + 延迟导入**”设计。

### Tools：内建扫描 + Python entry points

`ToolRegistry` 负责：

- 工具名称管理；
- JSON Schema 输出；
- 参数归一化与验证；
- 调用执行。

`ToolLoader` 通过包扫描发现内建工具，也支持 `nanobot.tools` entry point 外部插件。工具用 `ToolContext` 获取 workspace、bus、session、cron、MCP、runtime event 等运行时依赖。

内建工具覆盖：

- 文件读取、修改、补丁；
- Shell；
- Web search / fetch；
- MCP；
- cron；
- image generation；
- subagent；
- long-running goal；
- CLI Apps 等。

### Skills / MCP / Agent Plugins

三者角色不同：

- **Skill**：Markdown 工作流指导，偏“知识和行为约束”；
- **MCP**：运行时工具服务，偏“外部能力”；
- **Agent Plugin**：安装与启用边界，可打包 Skill、MCP、CLI App。

这个区分是正确的：不应该把所有新能力都塞进 `AgentLoop`。

---

## 8. WebUI、TUI、API、SDK

### WebUI

- 前端：React 18 + TypeScript + Vite + Tailwind；
- 通信：主要通过 WebSocket 多路复用协议；
- 服务端：`nanobot/channels/websocket/runtime.py` 同时承载 WebSocket 和大量 WebUI HTTP 路由；
- 前端核心客户端：`webui/src/lib/nanobot-client.ts`，约 1500 行，维护连接、请求、事件分发和会话同步；
- 生产构建进入 `nanobot/web/dist/`，随 Python wheel 分发。

WebUI 在架构上不是外部 BFF，而是 Gateway 内嵌的一套强客户端适配层。

### TUI

`tui/` 是独立的 TypeScript/OpenTUI 项目。CLI 在适当条件下快速启动原生 TUI，避免一开始加载完整 CLI 图谱。TUI 与 Python Gateway 通过协议连接，而不是重写 Agent 核心。

### OpenAI 兼容 API

`nanobot/api/server.py` 使用 `aiohttp` 暴露：

- `POST /v1/chat/completions`
- `GET /v1/models`
- `GET /health`

流式请求用 SSE，内部直接调用 `AgentLoop.process_direct()`，带 API session lock 和超时控制。

### Python SDK

`Nanobot` facade 封装 `AgentLoop`，提供：

- `run()`；
- 流式运行；
- session / memory / runtime client；
- hooks；
- MCP 生命周期。

这让项目同时具备“交互型产品”与“可嵌入 Agent 库”的属性。

---

## 9. 安全与部署

安全边界主要集中在 `nanobot/security/` 与工具层：

- **workspace containment**：文件工具限制到 workspace；
- **SSRF guard**：web/MCP HTTP 访问校验私网、环回、metadata endpoint 等；
- **Shell sandbox**：支持 bubblewrap，但并非所有运行环境都具备进程级隔离；
- **WebSocket 暴露保护**：监听所有网卡时要求 token、token issuer 或可信代理认证；
- **Docker 最小权限**：容器运行时降到非 root 用户，capability drop，`no-new-privileges`。

Docker 采用 Node 构建 WebUI + Python/uv 运行时的多阶段镜像。Docker Compose 默认仅将健康检查端口绑定在 localhost，而 WebSocket/WebUI 默认暴露 `8765`。

---

## 10. 架构优点

1. **核心边界清晰**  
   `AgentLoop` 与 `AgentRunner` 的职责切分合理，避免渠道细节污染模型工具循环。

2. **扩展点完整且分层**  
   Channel、Tool、MCP、Skill、Agent Plugin 各有清晰定位。

3. **本地自托管友好**  
   配置、session、memory、cron 都以本地持久化为中心，部署门槛低。

4. **并发模型符合聊天 Agent 场景**  
   不同 session 并行、相同 session 串行、运行中消息注入，能避免上下文竞态。

5. **状态模型成熟**  
   session history、memory、summary、provider private state、runtime context 分层处理，细节较扎实。

6. **多客户端复用核心能力**  
   WebUI、TUI、API、SDK 都复用 AgentLoop，避免各入口出现不同的推理逻辑。

7. **测试投入较高**  
   根目录 `tests/` 有 307 个 Python 测试文件、约 111k LOC，另外 Channel 包内也有测试；CI 覆盖 Python、WebUI、TUI、Docker、类型检查和 lint。

---

## 11. 主要架构风险与改进方向

### 1）核心编排文件偏大

`agent/loop.py` 约 2411 行，`runner.py` 约 1710 行，`session/manager.py` 约 1814 行；WebSocket runtime 约 2125 行。

这不一定是设计错误，但已经是维护风险信号：新需求很容易继续堆积到核心热路径。

**建议：**
- 不进行大重写；
- 按已有 turn stages，逐步抽出独立的协调器，例如：
  - turn persistence；
  - automation turn admission；
  - session checkpoint/recovery；
  - runtime model selection；
- 保持 `AgentLoop` 作为 orchestration façade，而不是继续增加具体业务逻辑。

这也与项目自身 `.agent/design.md` 中“核心保持小、能力扩展在边缘”的约束一致。

### 2）`MessageBus` 是无界内存队列

当前 `MessageBus` 使用未设置 `maxsize` 的 `asyncio.Queue`。突发消息、慢模型、慢渠道发送、恶意请求都可能造成内存累积。

**建议：**
- 为入站/出站设置上限；
- 按 channel / session 定义背压策略；
- 对 stream/progress 类事件做合并或丢弃策略；
- 对最终消息与控制消息设置更高优先级。

### 3）单进程模型意味着无分布式容错

当前是单 Gateway 进程，内存中的入站队列、运行中 tool call、stream、pending injection 在进程退出时都会丢失。session/memory/cron 是持久化的，但运行中任务不是。

**建议：**
- 对个人自托管场景，这一取舍合理；
- 若要面向团队或高可用部署，需引入显式的 durable job/outbox、幂等键、worker/executor 边界，而非直接把现有 Gateway 横向扩容。

### 4）WebUI 耦合度较高

`ChannelManager` 接收大量 WebUI 专用 callback，WebSocket runtime 也同时承担协议、鉴权、HTTP 路由、会话管理、媒体和设置 API。

**建议：**
- 继续强化已有 `GatewayServices` 这类服务聚合对象；
- 让 `ChannelManager` 依赖一个明确的 WebUI port/interface，而不是持续增加构造参数；
- WebSocket transport 保持 transport 职责，业务路由下沉到 `webui/*` 服务模块。

### 5）Shell 安全依赖部署配置

workspace restriction 是应用层控制；没有 bubblewrap 等后端时，Shell 并不是强隔离。

**建议：**
- 在生产配置中把“允许 Shell”“允许写入”“允许访问额外目录”“sandbox 可用”明确做成可视化风险状态；
- 对公开 WebUI 或多用户渠道，默认采用更严格的工具 profile；
- 对高风险工具增加审计事件、调用限流和更细的 capability policy。

---

## 12. 推荐阅读顺序

如果要继续深入源码，我建议按以下顺序：

1. `nanobot/cli/gateway_runtime.py`  
   先理解所有组件如何被组装和启动。

2. `nanobot/bus/events.py`、`bus/queue.py`、`bus/runtime_events.py`  
   理解消息、流式输出、运行时事件的区别。

3. `nanobot/agent/loop.py`  
   理解一个用户 turn 的 session、context、delivery、persist 生命周期。

4. `nanobot/agent/runner.py`  
   理解 provider-tool 多轮循环和工具执行策略。

5. `nanobot/session/manager.py`、`agent/memory.py`  
   理解持久化、压缩、Dream 和记忆模型。

6. `nanobot/agent/tools/registry.py`、`loader.py`、`mcp.py`  
   理解能力如何暴露给模型。

7. `nanobot/channels/manager.py`、任一具体 channel（建议 `telegram` 或 `websocket`）  
   理解接入适配模式。

8. `webui/src/lib/nanobot-client.ts` 与 `channels/websocket/runtime.py`  
   理解 WebUI 协议和实时事件投影。

---

## 项目架构总结

nanobot 整体上是一个以 Python asyncio 为核心的模块化单体 AI Agent 框架：Gateway 作为运行时组合根，统一装配 MessageBus、AgentLoop、AgentRunner、模型 Provider、ToolRegistry、Session/Memory、Cron 和各类 Channel；WebUI、TUI、CLI、聊天平台、OpenAI 兼容 API 与 Python SDK 都通过不同适配层接入同一套 Agent 核心，其中 AgentLoop 负责会话、上下文、任务调度和消息投递，AgentRunner 负责模型调用、流式响应及工具执行循环，Provider 和 Tool 通过注册与插件机制扩展，Session、长期记忆和 Dream 提供持久化能力，最终形成“多入口接入—消息/事件总线—Agent 编排—模型与工具执行—会话记忆持久化”的完整架构。