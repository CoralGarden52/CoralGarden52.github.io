---
title: nanobot的项目实现
cover: /assets/posts/nanobot/项目架构.png
categories: agent
tags:
  - nanobot
---

以 **Agent 应用开发工程师** 的视角从零开始构建 nanobot，正确路径是先构造一个**可验证的 Agent 最小闭环**，再逐步把它演化为一个可长期运行、可扩展、可部署的个人 Agent 平台。整个过程可以理解为从 **“一次 CLI 问答” → “有工具的 Agent” → “多会话 Gateway” → “多端产品” → “可靠的平台”** 的五次架构升级。

---

# 一、先确定产品边界：我到底在做什么

nanobot 的本质不是“调用一次大模型的聊天机器人”，而是：

> 一个能从不同入口接收任务，带着上下文调用模型，按需执行工具，把执行结果持续反馈给用户，并将任务历史、记忆和自动化能力保留下来的本地 Agent Runtime。

因此，项目最开始要明确四个核心对象：

```text
用户输入（Message）
    ↓
Agent 运行一次任务（Turn）
    ↓
模型决定回答或调用工具（LLM + Tool Calls）
    ↓
输出、状态、历史被保存并反馈给用户（Response + Session + Memory）
```

最初的架构目标不是“支持一切”，而是保证这个闭环稳定：

```text
CLI 输入
  → 构建 messages
  → 调用 LLM
  → 如有工具调用则执行
  → 继续调用 LLM
  → 输出最终结果
  → 保存会话
```

只要这个闭环没有测试、没有错误处理、没有清晰数据模型，就不要做 WebUI 和多渠道。

---

# 二、从 0 到 1：先做一个单进程 CLI Agent

## 阶段 0：项目脚手架与工程约束

先建立最小工程结构：

```text
nanobot/
  cli/
  agent/
  providers/
  tools/
  session/
  config/
tests/
pyproject.toml
README.md
```

技术选型应尽量克制：

- Python 3.11+；
- `asyncio`：后续所有模型调用、工具调用、渠道 I/O 都能统一；
- Pydantic：配置和边界输入校验；
- Typer：CLI；
- pytest：测试；
- 一开始只支持一个模型协议，例如 OpenAI-compatible；
- 一开始只支持一个工具，例如 `read_file` 或 `shell`。

此时就应写下几条长期不变的架构约束：

1. **核心 Agent 不直接依赖 Telegram、WebUI 等渠道细节。**
2. **模型 Provider 必须有统一抽象。**
3. **工具必须有明确 Schema、名称、描述、参数校验和执行接口。**
4. **会话持久化必须与模型 Provider 解耦。**
5. **所有不可信输入必须在边界层做校验。**

这就是 nanobot 后来 `.agent/design.md` 中“核心保持小、能力在边缘扩展”的原型。

---

## 阶段 1：定义不可轻易变化的核心契约

最先写的不是复杂实现，而是数据模型和接口。

### 1. 消息模型

```python
@dataclass
class InboundMessage:
    channel: str
    chat_id: str
    sender_id: str
    content: str
    media: list[str]
    metadata: dict[str, Any]

@dataclass
class OutboundMessage:
    channel: str
    chat_id: str
    content: str
    metadata: dict[str, Any]
```

这里即使最初只有 CLI，也要保留 `channel/chat_id`。原因是：未来 Telegram、WebUI、API 都必须能映射到同一套会话机制。

### 2. Provider 抽象

```python
class LLMProvider(ABC):
    @abstractmethod
    async def chat(
        self,
        messages: list[dict[str, Any]],
        tools: list[dict[str, Any]],
        model: str,
    ) -> LLMResponse:
        ...
```

返回统一的：

```python
@dataclass
class LLMResponse:
    content: str | None
    tool_calls: list[ToolCallRequest]
    usage: dict[str, int]
    finish_reason: str
```

这样后续接入 Anthropic、OpenAI Responses、Azure、Bedrock、Codex 等时，Agent 核心不需要改。

### 3. Tool 抽象

```python
class Tool(ABC):
    name: str
    description: str
    parameters: dict[str, Any]  # JSON Schema

    @abstractmethod
    async def execute(self, **kwargs) -> ToolResult:
        ...
```

工具必须先完成三件事：

- 给模型看的 JSON Schema；
- 参数校验；
- 执行结果格式化。

不要把工具调用逻辑散落在 Agent 主循环中，否则工具一多，核心会迅速失控。

### 4. Session 抽象

```python
@dataclass
class Session:
    key: str
    messages: list[dict[str, Any]]
    metadata: dict[str, Any]
```

最初只需用 JSON 文件保存，重点是先稳定 session key：

```text
cli:default
telegram:<chat_id>
websocket:<chat_id>
api:<session_id>
```

这就是 nanobot 后来能够实现多渠道复用、WebUI 多会话和 unified session 的基础。

---

## 阶段 2：实现最小 Agent Runner

这一阶段只做一件事：让模型可以连续调用工具。

核心伪代码：

```python
messages = build_initial_messages(user_input)

for _ in range(max_iterations):
    response = await provider.chat(messages, tools=registry.schemas())

    if response.content:
        stream_or_collect(response.content)

    if not response.tool_calls:
        return response.content

    for tool_call in response.tool_calls:
        result = await tool_registry.execute(
            tool_call.name,
            tool_call.arguments,
        )
        messages.append(tool_result_message(tool_call, result))
```

这个类后来演化为 nanobot 的：

```text
nanobot/agent/runner.py
```

在这一阶段，必须尽早处理：

- 最大工具调用轮数；
- 工具参数非法；
- 模型返回未知工具；
- 工具执行异常；
- 大模型 API 超时、限流、连接失败；
- 流式输出；
- 工具结果过长；
- 用户取消任务。

**关键原则：模型不可信，工具调用也不可信。**  
模型给出的工具名、参数结构、路径、URL 都必须经过本地校验。

---

# 三、从 1 到 10：从一次运行变成可靠的会话 Agent

## 阶段 3：引入 `AgentLoop`，分离“用户任务编排”和“模型循环”

当开始支持会话、命令、流式反馈、任务取消、多个入口时，不能继续把所有逻辑塞进 `AgentRunner`。

应该拆出：

```text
AgentLoop：面向用户和会话
AgentRunner：面向 LLM 和工具调用
```

### `AgentLoop` 应负责

- 收到一条用户消息；
- 决定使用哪个 session；
- 加载会话历史；
- 构造系统提示词和用户上下文；
- 执行 slash command；
- 绑定当前 workspace；
- 调用 `AgentRunner`；
- 处理流式输出；
- 保存最终会话；
- 投递最终回复；
- 调度压缩、记忆、自动化等后台任务。

### `AgentRunner` 应只负责

- 调模型；
- 流式解析；
- 执行工具；
- 将工具结果回填给模型；
- 重试、迭代、结束。

nanobot 当前的核心分层就是沿着这条线形成的：

```text
nanobot/agent/loop.py     # Turn / Session / Channel-facing orchestration
nanobot/agent/runner.py   # Provider / Tool-facing reasoning loop
```

这是整个系统最重要的架构决策。

---

## 阶段 4：会话与上下文治理

Agent 一旦具备多轮对话，就会遇到两个问题：

1. 历史越来越长；
2. 历史里会混入模型不该重复学习的内部数据。

因此，应该把会话管理独立成模块：

```text
nanobot/session/
  manager.py
  keys.py
  summary.py
  history_visibility.py
```

至少实现：

- 按 session key 读取/保存；
- 历史 replay；
- 只保存本轮新消息；
- 清理工具调用回显；
- 清理超长或敏感运行时上下文；
- 截断历史到 token 预算内；
- 保证 replay 时 tool-call / tool-result 的消息边界合法。

接着再增加**会话摘要**：

```text
旧历史 → 模型摘要 → 保存 summary
最近消息 → 保持原样
summary + 最近消息 → 下次请求上下文
```

nanobot 进一步加入了：

- 自动 compact；
- session TTL；
- provider 私有 continuation state；
- 会话 fork；
- WebUI transcript；
- unified session。

这些都应该在核心 session 机制稳定后再加，而不是一开始设计过度复杂的数据库模型。

---

# 四、从 10 到 30：把 Agent 从“会聊天”升级为“能做事”

## 阶段 5：建立 ToolRegistry 与工具生态

工具数量超过 3~5 个时，必须正式引入：

```text
ToolLoader → ToolRegistry → Tool
```

职责为：

```text
ToolLoader
  - 扫描内建工具
  - 加载 Python entry-point 插件

ToolRegistry
  - 注册工具
  - 返回模型可见 Schema
  - 参数归一化与校验
  - 工具查找与执行

Tool
  - 单项能力实现
```

工具实现的推荐顺序：

1. `read_file` / `list_dir`
2. `write_file` / `edit_file` / `apply_patch`
3. shell
4. web search / web fetch
5. cron
6. MCP
7. image generation
8. subagent
9. long-running goal

这个顺序有意义：前四项先建立“本地工作能力”，再建立“联网能力”和“自动化能力”，最后才做更复杂的委派和长程执行。

---

## 阶段 6：安全能力要和工具能力同步生长

Agent 工具越强，安全不能最后补。

### 文件系统安全

文件工具必须经过 workspace containment：

```text
用户路径 / 模型路径
  → normalize
  → resolve
  → 验证是否在 workspace 内
  → 根据 read / write capability 决定是否允许
```

而不是：

```python
open(user_supplied_path)
```

nanobot 的 `security/workspace_access.py`、工具路径解析和 workspace policy 就来自这一需求。

### 网络安全

Web fetch、图片下载、HTTP MCP 必须做 SSRF 防护：

- 禁止 loopback；
- 禁止私网；
- 禁止 link-local；
- 禁止云 metadata endpoint；
- 解析重定向后的目标；
- 仅通过显式白名单开放内部地址。

### Shell 安全

需要明确告诉用户：

- workspace restriction 只是应用层限制；
- 它不是容器级隔离；
- 生产环境最好使用 bubblewrap、容器或独立执行器；
- shell、写文件、网络访问必须可独立开关。

这也是 nanobot 后来出现 sandbox、Docker 权限收缩、`no-new-privileges` 等设计的原因。

---

# 五、从 30 到 60：从 CLI Agent 演化为 Gateway

## 阶段 7：引入 `MessageBus`，让渠道和 Agent 解耦

当开始接入 Telegram、Discord、WebUI 时，不应该让每个渠道直接调用 Agent 的内部逻辑。

应该先定义异步消息总线：

```python
class MessageBus:
    inbound: asyncio.Queue[InboundMessage]
    outbound: asyncio.Queue[OutboundMessage]
```

运行模型变成：

```text
Channel
  → publish_inbound()
  → AgentLoop.consume_inbound()
  → AgentLoop.process_turn()
  → publish_outbound()
  → ChannelManager.consume_outbound()
  → Channel.send()
```

这就是 nanobot 的核心数据流。

它带来的收益：

- Channel 不需要理解 Agent 内部；
- Agent 不需要理解 Telegram / Discord API；
- 支持多渠道复用同一套 session、tool 和 provider；
- 便于把进度、流式 delta、工具提示建模成 outbound event；
- 便于未来监控队列、做优先级与背压控制。

---

## 阶段 8：Channel 抽象与插件化

先做一个渠道，例如 Telegram；稳定后再抽象：

```python
class BaseChannel(ABC):
    async def start(self) -> None: ...
    async def stop(self) -> None: ...
    async def send(self, message: OutboundMessage) -> None: ...
```

然后每个渠道保持独立目录：

```text
nanobot/channels/telegram/
  manifest.py
  runtime.py
  validation.py

nanobot/channels/discord/
nanobot/channels/slack/
nanobot/channels/websocket/
```

关键是：**Channel manifest 与运行时实现分开。**

manifest 只包含：

- 名称；
- 显示名；
- optional dependencies；
- runtime import path；
- setup/management metadata；
- capability 信息。

这样可在不导入 Telegram、Discord SDK 的前提下，发现系统支持哪些渠道；只有启用时才加载其运行时依赖。

---

## 阶段 9：Gateway 生命周期

此时才引入：

```text
nanobot gateway
```

Gateway 做的不是模型推理，而是运行时托管：

```text
加载 Config
  → 创建 MessageBus
  → 创建 SessionManager
  → 创建 ToolRegistry
  → 建立 MCP 连接
  → 创建 AgentLoop
  → 创建 ChannelManager
  → 启动 CronService
  → 启动 AgentLoop / Channels / Health Server
  → 监听配置变更
  → 优雅关闭所有资源
```

nanobot 当前 `cli/gateway_runtime.py` 就是这个组合根。

从作者视角，这一步非常重要：  
**不要让 CLI command、WebUI、Channel 各自创建一套 Agent。**  
必须让它们共享同一个 Gateway 和状态空间，否则 session、cron、MCP、模型配置都会产生分裂。

---

# 六、从 60 到 80：加入记忆、自动化和复杂运行时能力

## 阶段 10：长期记忆与 Dream

会话摘要解决的是“当前对话上下文不能无限长”，但不能解决“长期记住用户偏好、项目背景和经验”。

因此再增加 workspace 级记忆：

```text
workspace/
  memory/
    MEMORY.md
    SOUL.md
    USER.md
    history.jsonl
```

其中：

- `MEMORY.md`：稳定事实与长期经验；
- `SOUL.md`：Agent 风格、身份、行为倾向；
- `USER.md`：用户偏好；
- `history.jsonl`：可用于长期整理的历史事件。

然后创建 Dream 任务：

```text
累计历史
  → 定期挑选未处理片段
  → 调模型归纳
  → 更新 MEMORY.md
  → 记录处理 cursor
```

Dream 不应是普通用户消息；它是系统后台任务，应使用独立 session 或受控上下文，以避免污染用户对话。

---

## 阶段 11：Cron、Heartbeat、Trigger

有了 Gateway 后，自动化才有意义：

- Cron：用户创建提醒或定时任务；
- Heartbeat：周期性读取 `HEARTBEAT.md` 中的 active tasks；
- Dream：周期性压缩历史；
- Local Trigger：外部脚本将事件注入指定 session。

关键设计原则：

> 自动化任务不是简单的“定时执行一个 Python 函数”，而是要以受控的 Agent turn 形式进入既有 session。

这样任务才能拥有：

- 原本的会话上下文；
- 原本的模型选择；
- 原本的消息投递渠道；
- 原本的工具权限；
- 可追踪的历史记录。

同时需要处理：

- session 正在运行时的延迟投递；
- at-least-once 语义；
- 幂等；
- 失败状态；
- 不要将无意义的“没有变化”推送给用户。

---

## 阶段 12：MCP、子 Agent、长程目标

### MCP

MCP 要作为**应用级基础设施**管理，而不应由每个 turn 临时连接：

```text
Gateway 启动
  → MCPProvider.connect()
  → 注册 MCP tools 到 ToolRegistry
  → AgentLoop 使用共享 ToolRegistry
Gateway 停止
  → MCPProvider.aclose()
```

原因是 MCP 连接有生命周期、认证、网络安全和重连问题。

### Subagent

子 Agent 不应该复制主 Agent 的所有状态，而应显式定义：

- 可否继承 workspace；
- 可否使用哪些工具；
- 最大并发数；
- 结果如何回传；
- 如何将最终结果注入主会话。

### 长程目标

长程任务应被建模为 session metadata 和 checkpoint，而不是让一个无限循环的 coroutine 永久占着 Agent：

```text
目标状态
  → 当前步骤
  → 运行 checkpoint
  → 可恢复
  → 可取消
  → 可展示给 WebUI
```

---

# 七、从 80 到 100：把运行时做成产品

## 阶段 13：WebUI

WebUI 不应重新实现 Agent，而应成为 Gateway 的一个客户端。

推荐技术路径：

```text
React + TypeScript + Vite
  ↕ WebSocket
Gateway 内嵌 WebSocket channel / HTTP routes
  ↕
MessageBus + RuntimeEventBus + SessionManager
```

WebUI 需要的不是只有“发送文字和显示回复”，还包括：

- 会话列表、标题、搜索、fork；
- 流式回答；
- 推理和工具执行活动；
- 文件预览和 diff；
- 上传附件；
- 模型、Provider、MCP、Skill、Channel 设置；
- cron、goal、自动化状态；
- workspace 选择和访问模式；
- token 用量；
- token / proxy 鉴权。

因此，应该额外建立 `RuntimeEventBus`，把“用户可见回复”与“运行状态变化”分离：

```text
MessageBus：
  文本回复、流式 delta、进度消息

RuntimeEventBus：
  turn started
  runtime admitted
  tool running
  turn completed
  goal changed
  session persisted
```

这是 nanobot 当前能实现较丰富 WebUI 状态投影的关键。

---

## 阶段 14：API 与 SDK

当核心 AgentLoop 稳定后，API 和 SDK 应该是“薄适配层”。

### OpenAI-compatible API

```text
HTTP /v1/chat/completions
  → 验证请求
  → 映射 session_id 到 api:<session_id>
  → AgentLoop.process_direct()
  → 非流式 JSON 或 SSE 输出
```

不要在 API 里再实现一份 provider/tool loop。

### Python SDK

```python
bot = Nanobot.from_config()
result = await bot.run("分析当前仓库")
```

SDK 应直接复用：

- `AgentLoop`；
- ToolRegistry；
- SessionManager；
- hooks；
- streaming events；
- MCP 生命周期。

这样 CLI、WebUI、API、SDK 的行为才能一致。

---

# 八、作者真正需要持续做的工程工作

一个项目到这个规模，真正难的不是“加一个功能”，而是持续保持架构可维护。

## 1. 每增加一个能力，都先问它属于哪一层

例如新增“GitHub 自动创建 Issue”：

- 如果是模型可调用动作：Tool 或 MCP；
- 如果只是操作指南：Skill；
- 如果是外部聊天平台接入：Channel；
- 如果是 WebUI 设置页面：WebUI adapter；
- 如果只是新模型 API：Provider；
- 不要默认往 `AgentLoop` 里加 `if feature_xxx`。

## 2. 先写边界测试，再接第三方 SDK

典型测试应覆盖：

- Provider response 的解析；
- Tool 参数校验；
- 路径逃逸；
- SSRF；
- session replay 合法性；
- 消息注入；
- 流式分段；
- cron 任务重复投递；
- WebSocket 鉴权；
- 配置迁移。

nanobot 的测试量很大，本质上是因为 Agent 系统的错误常发生在**边界交互处**，而不是普通业务函数内部。

## 3. 设计“可观测的失败”

系统必须让用户、开发者和 WebUI 能区分：

```text
模型超时
工具失败
工具参数错误
权限不足
workspace 越界
SSRF 拦截
session 被取消
配置无效
MCP 断连
```

不能只返回“发生错误”。

## 4. 配置必须显式、可迁移、可验证

配置应由 Pydantic Schema 统一声明：

```text
agents.defaults
providers
channels
tools
gateway
api
mcp_servers
```

并支持：

- camelCase JSON；
- 默认值；
- 环境变量引用；
- 配置迁移；
- 清晰验证错误；
- 配置热更新后的 runtime refresh。

---

# 九、如果你要复刻 nanobot，推荐的实际里程碑

## 第 1 周：最小 Agent

- CLI；
- 单一 OpenAI-compatible Provider；
- 单轮对话；
- JSON session 保存；
- 一个只读文件工具；
- pytest 基础测试。

## 第 2～3 周：可用的工具型 Agent

- `AgentLoop` / `AgentRunner` 分离；
- ToolRegistry；
- 流式输出；
- shell、文件编辑；
- workspace restriction；
- 系统提示词与上下文构造；
- tool error/retry/iteration limit。

## 第 4～5 周：Gateway 与渠道

- MessageBus；
- ChannelManager；
- WebSocket 或 Telegram 先做一个；
- session lock；
- 同 session 消息注入；
- 后台 Gateway；
- 健康检查与优雅关闭。

## 第 6～7 周：可靠性与自动化

- token 上下文控制；
- summary / auto compact；
- 长期 memory；
- cron、heartbeat、trigger；
- 配置文件 watcher；
- Docker 部署。

## 第 8～10 周：产品化接口

- React WebUI；
- WebSocket protocol；
- API / SDK；
- 文件上传与媒体处理；
- Provider 设置、Channel 设置；
- 鉴权、token、审计和错误展示。

## 后续持续演化

- MCP；
- Agent Plugins；
- Skills marketplace；
- 子 Agent；
- goal/checkpoint；
- 多 Provider fallback；
- 云部署；
- 更严格的 sandbox；
- 更完整的可观测性和指标。

---

# 十、最核心的作者思维

如果只记住一句话，应当是：

> **先把“一个会话内可控地完成一次 Agent turn”做正确，再让它通过消息总线接入更多用户入口，再把记忆、自动化、UI、插件和部署逐步附着在稳定边界上。**

nanobot 当前的成熟架构，并不是一开始设计出全部模块，而是通过不断把复杂性推到边缘形成的：

```text
核心：
AgentLoop + AgentRunner + Provider Contract + Tool Contract + Session

边缘：
Channels + WebUI + API + SDK + MCP + Skills + Plugins + Cron

基础设施：
Config + Security + Persistence + Runtime Lifecycle + Tests + Docker
```

这也是构建复杂 Agent 应用最可靠的路径：**核心少而稳，适配层多而可替换，状态可持久，权限可约束，所有入口共享同一个运行时。**