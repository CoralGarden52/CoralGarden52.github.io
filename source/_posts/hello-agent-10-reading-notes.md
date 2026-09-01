---
title: Hello-Agent第十章智能体通信协议的阅读笔记
cover: /assets/posts/hello-agents/hello-agent-10.png
categories: agent
tags:
  - hello-agents
---

# 面试速记

## 一句话概括

第十章主要讲 **智能体通信协议**：当 Agent 不再只是单体问答系统，而是需要连接外部工具、调用其他 Agent、加入大规模智能体网络时，就需要标准化通信协议。本章重点介绍了 **MCP、A2A、ANP** 三类协议，其中 **MCP 是最重要、生态最成熟、最适合当前项目落地的部分**。

## 背诵版总结

本章的核心是从“单体智能体”走向“可连接、可协作、可扩展的智能体系统”。前面章节中的 ReAct Agent 已经可以推理、调用工具和使用记忆，但它的问题是工具接入成本高、服务之间难复用、多个智能体之间缺少标准协作方式。因此，智能体通信协议的价值在于把 Agent 与外部世界的交互标准化。

本章介绍了三种协议：**MCP、A2A 和 ANP**。MCP，即 Model Context Protocol，主要解决“智能体如何标准化访问工具和资源”的问题，可以把文件系统、数据库、GitHub、Slack、本地脚本等外部能力封装成统一的 MCP Server，让 Agent 通过 MCP Client 动态发现工具并调用。A2A，即 Agent-to-Agent Protocol，主要解决“智能体之间如何点对点通信和协作”的问题，适合研究员、撰写员、编辑等多个 Agent 分工协作。ANP，即 Agent Network Protocol，主要面向大规模智能体网络，解决服务发现、智能路由、身份认证和网络扩展问题。

其中 MCP 是当前最值得重点掌握的协议。它可以理解为 Agent 工具生态的 “USB-C”：不同工具不再需要重复手写适配器，而是通过统一协议提供 Tools、Resources 和 Prompts。HelloAgents 中基于 FastMCP 实现 MCPClient 和 MCPServer，并通过 MCPTool 把 MCP Server 自动展开成普通 Agent 工具，使 Agent 能像调用普通工具一样调用外部能力。MCP 和 Function Calling 不是竞争关系，Function Calling 是模型判断“何时调用工具、生成什么参数”的能力，而 MCP 是工程层面连接工具、发现工具和标准化调用工具的协议基础设施。

## 面试表达模板

如果面试官问“你怎么理解 MCP”，可以这样回答：

> MCP 是 Model Context Protocol，可以理解为连接大模型智能体和外部工具的标准协议。传统做法中，如果 Agent 要访问 GitHub、数据库、文件系统或企业内部 API，开发者通常要为每个服务单独写 Tool 类、处理认证、参数格式、错误处理和返回格式，这会导致工具难复用、难维护。MCP 把这些外部能力封装成标准化的 MCP Server，Agent 侧通过 MCP Client 连接服务器，先通过 list_tools 发现工具，再通过 call_tool 按统一格式调用工具。MCP 的核心能力包括 Tools、Resources 和 Prompts，其中 Tools 负责执行操作，Resources 负责提供数据，Prompts 负责提供可复用提示模板。它和 Function Calling 是互补关系：Function Calling 是模型的能力，负责判断是否需要调用工具并生成参数；MCP 是工程协议，负责让工具以统一方式被发现、描述和调用。

如果面试官问“第十章三种协议怎么区分”，可以这样回答：

> MCP 解决的是 Agent 和工具之间的通信，适合文件系统、数据库、GitHub、API 等外部服务接入；A2A 解决的是 Agent 和 Agent 之间的点对点协作，适合多智能体分工、委托任务和协商；ANP 解决的是大规模智能体网络中的服务发现、路由和身份信任问题，适合开放网络和分布式 Agent 生态。因此可以概括为：MCP 连接工具，A2A 连接智能体，ANP 连接智能体网络。

---

# Hello-Agent 第十章：智能体通信协议阅读笔记

## 1. 本章核心内容

第十章《智能体通信协议》关注的是智能体系统从“单体运行”走向“外部连接”和“多智能体协作”时遇到的问题。

前面的章节已经构建了具备推理、工具调用、记忆和上下文管理能力的 Agent，但这些 Agent 仍然面临三个限制：

1. **工具集成成本高**：每接入一个新服务，都要手写一个 Tool 类。
2. **能力扩展受限**：Agent 只能使用预先写好的工具，无法动态发现新工具。
3. **协作能力不足**：多个 Agent 之间缺少标准通信方式，通常只能手动编排。

智能体通信协议要解决的就是：

> 如何让 Agent 以标准化方式访问工具、调用其他 Agent，并在更大的智能体网络中完成服务发现和协作？

本章介绍三种协议：

| 协议 | 全称 | 主要解决问题 | 一句话理解 |
|---|---|---|---|
| MCP | Model Context Protocol | Agent 与工具 / 资源通信 | 让 Agent 标准化调用外部工具 |
| A2A | Agent-to-Agent Protocol | Agent 与 Agent 协作 | 让多个 Agent 点对点通信 |
| ANP | Agent Network Protocol | 大规模 Agent 网络发现与路由 | 让 Agent 在网络中发现和连接服务 |

---

# 2. 为什么需要智能体通信协议？

## 2.1 传统工具接入方式的问题

在传统 Agent 开发中，如果要让 Agent 使用外部服务，通常需要手写工具适配器。

例如：

```python
class GitHubTool(BaseTool):
    def run(self, repo_url):
        # GitHub API 调用逻辑
        pass

class DatabaseTool(BaseTool):
    def run(self, query):
        # 数据库连接和查询逻辑
        pass

class WeatherTool(BaseTool):
    def run(self, location):
        # 天气 API 调用逻辑
        pass
```

这种方式的问题很明显：

- 每个服务都要重复写 HTTP 请求、认证、错误处理；
- 不同开发者写的工具接口不统一；
- 工具难复用；
- API 变更后维护成本高；
- Agent 无法自动发现新工具。

## 2.2 通信协议的价值

通信协议的价值是把工具、Agent、服务之间的交互方式标准化。

这类似互联网中的 TCP/IP 协议。不同设备只要遵循同一协议，就可以互相通信。  
在 Agent 系统中，不同工具或智能体只要遵循同一协议，就可以被 Agent 统一调用。

通信协议带来的变化包括：

- **标准化接口**：不同服务使用统一访问方式；
- **互操作性**：不同开发者的工具可以被复用；
- **动态发现**：Agent 可以在运行时发现工具和能力；
- **可扩展性**：系统可以更容易添加新功能；
- **工程解耦**：Agent 不需要关心底层服务细节。

---

# 3. 三种协议设计理念

## 3.1 MCP：智能体与工具的桥梁

MCP 的核心目标是：

```text
标准化 Agent 与外部工具 / 资源之间的通信
```

它适合解决：

- 访问文件系统；
- 访问数据库；
- 查询 GitHub；
- 调用企业内部 API；
- 读取知识库；
- 连接 Slack、Google Drive 等外部服务。

MCP 的设计哲学是“上下文共享”。  
它不仅仅是简单的远程过程调用，还强调让 Agent 和工具之间共享丰富的上下文信息。例如访问代码仓库时，MCP Server 不只返回文件内容，还可以提供目录结构、依赖关系、提交历史等上下文。

一句话理解：

> MCP 是 Agent 工具生态的 USB-C。

## 3.2 A2A：智能体之间的对话

A2A 的核心目标是：

```text
实现 Agent 与 Agent 之间的点对点通信
```

它关注的是多个智能体如何协作，例如：

- 研究员 Agent 搜集资料；
- 撰写员 Agent 生成文章；
- 编辑 Agent 审查质量；
- 协调员 Agent 分派任务。

A2A 的设计哲学是“对等通信”。  
每个 Agent 既可以是服务提供者，也可以是服务消费者。它们可以互相请求、响应、协商任务，而不一定依赖中央协调器。

一句话理解：

> A2A 让 Agent 像团队成员一样互相委托和协商。

## 3.3 ANP：智能体网络基础设施

ANP 的核心目标是：

```text
构建大规模、开放式智能体网络中的服务发现和路由机制
```

它要解决的问题包括：

- 新 Agent 如何加入网络；
- 一个 Agent 如何发现其他 Agent；
- 多个 Agent 都能完成任务时，如何选择最佳服务；
- 如何建立可信身份；
- 如何进行动态路由。

ANP 的设计哲学是“去中心化服务发现”。  
它更适合未来大规模 Agent 网络，而不是普通单体应用。

一句话理解：

> ANP 让 Agent 在网络中自动发现和连接其他服务。

---

# 4. HelloAgents 中的通信协议架构

HelloAgents 将通信协议设计为三层：

## 4.1 协议实现层

这一层实现具体协议：

```text
hello_agents/protocols/
├── mcp/
├── a2a/
└── anp/
```

其中：

- MCP 基于 FastMCP 实现；
- A2A 基于 Google A2A SDK 的部分能力；
- ANP 是轻量级概念实现，用于模拟服务发现和网络管理。

## 4.2 工具封装层

这一层把协议封装成统一 Tool 接口：

```text
MCPTool
A2ATool
ANPTool
```

它们都继承自 BaseTool，并提供一致的 `run()` 方法。

这样 Agent 不需要关心底层是 MCP、A2A 还是 ANP，只需要像调用普通工具一样调用它们。

## 4.3 智能体集成层

这一层是 Agent 和工具系统的结合点。

例如：

```python
agent.add_tool(MCPTool(...))
agent.add_tool(A2ATool(...))
agent.add_tool(ANPTool(...))
```

Agent 通过工具系统使用不同协议，实现外部能力扩展。

---

# 5. MCP 协议重点详解

## 5.1 MCP 是什么？

MCP 是 **Model Context Protocol**，即模型上下文协议。

它的目标是：

> 用统一协议连接大模型智能体和外部工具、数据源、资源系统。

传统方式下，每接入一个工具都要写适配器；MCP 则让工具服务以标准 MCP Server 的形式存在，Agent 通过 MCP Client 统一访问。

例如：

```python
github_mcp = MCPTool(
    server_command=["npx", "-y", "@modelcontextprotocol/server-github"]
)

fs_mcp = MCPTool(
    server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
)
```

Agent 不需要自己实现 GitHub API 或文件系统 API，只需要连接对应 MCP Server。

## 5.2 MCP 的三层架构

MCP 采用三层架构：

```text
Host -> Client -> Server
```

### 1. Host

Host 是用户直接交互的应用，例如 Claude Desktop、IDE、Agent 平台等。

它负责：

- 接收用户输入；
- 调用模型；
- 管理对话；
- 决定何时使用 MCP Client。

### 2. Client

Client 是协议通信层。

它负责：

- 连接 MCP Server；
- 发现可用工具；
- 发送工具调用请求；
- 接收工具返回结果。

### 3. Server

Server 是具体能力提供者。

它负责：

- 封装工具；
- 访问文件、数据库、API；
- 返回结果；
- 提供资源和提示模板。

一个典型流程是：

```text
用户提问
-> Host 接收问题
-> 模型判断需要工具
-> MCP Client 连接 MCP Server
-> Server 执行具体操作
-> 返回结果
-> 模型整合结果生成回答
```

## 5.3 MCP 的三类核心能力

MCP 提供三类核心能力：

| 能力 | 含义 | 特点 |
|---|---|---|
| Tools | 工具 | 主动执行操作 |
| Resources | 资源 | 被动提供数据 |
| Prompts | 提示模板 | 提供可复用提示词 |

### Tools

Tools 用来执行动作，例如：

- 读取文件；
- 写入文件；
- 搜索 GitHub；
- 查询数据库；
- 发送消息；
- 调用企业 API。

Tools 是 MCP 中最常用的能力。

### Resources

Resources 用来提供数据，例如：

- 文件内容；
- 数据库表；
- 文档片段；
- 配置文件；
- 日志资源。

它更像“可读取的数据源”。

### Prompts

Prompts 用来提供可复用提示模板，例如：

- 代码审查模板；
- 总结模板；
- 调研报告模板；
- SQL 分析模板。

它让工具服务器不仅能提供能力，还能提供“如何使用能力”的提示结构。

## 5.4 MCP 的工具调用流程

MCP 的工具调用通常包含五步：

```text
连接服务器 -> 发现工具 -> 构建上下文 -> 模型决策 -> 工具执行 -> 结果整合
```

具体来说：

1. **工具发现**：Client 调用 `list_tools()` 获取工具名称、描述、参数 schema。
2. **上下文构建**：把工具描述转成模型能理解的形式，加入系统提示或工具列表。
3. **模型推理**：LLM 根据用户问题判断是否需要调用工具。
4. **工具执行**：Client 调用 `call_tool(tool_name, arguments)`。
5. **结果整合**：工具结果返回模型，模型生成最终回答。

这里有一个关键点：

> 工具描述写得好不好，会直接影响模型会不会正确调用工具。

所以 MCP 工具的 `name`、`description` 和 `inputSchema` 非常重要。

## 5.5 MCP 与 Function Calling 的区别

很多人会把 MCP 和 Function Calling 混在一起。实际上它们不是竞争关系，而是互补关系。

| 对比点 | Function Calling | MCP |
|---|---|---|
| 本质 | 模型能力 | 工程协议 |
| 解决问题 | 模型如何判断调用函数并生成参数 | 工具如何被发现、描述、连接和调用 |
| 关注层面 | LLM 推理与参数生成 | 工具生态和系统集成 |
| 是否跨模型统一 | 不同模型格式不同 | 协议统一 |
| 是否提供工具服务生态 | 不提供 | 可以连接社区 MCP Server |
| 类比 | 会打电话 | 电话通信标准 |

简洁理解：

```text
Function Calling：模型知道什么时候该调用工具
MCP：工具能以统一标准被模型连接和调用
```

因此，一个完整系统通常是两者结合：

```text
LLM 用 Function Calling 判断调用哪个工具
MCP 负责把工具标准化接入和执行
```

## 5.6 HelloAgents 中使用 MCPClient

HelloAgents 基于 FastMCP 实现 MCPClient，支持异步和同步两种 API。

常见流程：

```python
from hello_agents.protocols import MCPClient

client = MCPClient([
    "npx", "-y", "@modelcontextprotocol/server-filesystem", "."
])

async with client:
    tools = await client.list_tools()
    result = await client.call_tool("read_file", {"path": "README.md"})
```

使用 MCPClient 时，常见步骤是：

1. 连接 MCP Server；
2. `list_tools()` 查看工具；
3. `call_tool()` 调用工具；
4. 处理异常；
5. 读取资源或使用 prompt 模板。

## 5.7 MCP 的传输方式

MCP 协议的一个重要特性是 **传输层无关**。

HelloAgents 中提到的传输方式包括：

| 传输方式 | 适合场景 |
|---|---|
| Memory Transport | 单元测试、快速原型 |
| Stdio Transport | 本地开发、本地脚本、社区服务器 |
| HTTP Transport | 远程服务、生产环境、微服务 |
| SSE Transport | 实时通信、长连接 |
| StreamableHTTP | 双向流式通信 |

最常见的是 Stdio：

```python
MCPTool(
    server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
)
```

它通过标准输入输出和本地 MCP Server 进程通信，适合本地开发和工具接入。

## 5.8 MCPTool 自动展开机制

HelloAgents 中非常重要的设计是 **MCPTool 自动展开**。

当你把一个 MCPTool 添加到 Agent 时，它会：

1. 连接 MCP Server；
2. 调用 `list_tools()` 获取所有工具；
3. 为每个 MCP 工具创建包装器；
4. 注册成 Agent 可直接调用的普通工具；
5. 自动处理参数类型转换；
6. 调用 MCP Server 执行真正操作。

例如：

```python
fs_tool = MCPTool(
    name="fs",
    server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
)

agent.add_tool(fs_tool)
```

如果文件系统 MCP Server 提供 `read_file`、`write_file` 等工具，那么它们会自动展开为：

```text
fs_read_file
fs_write_file
```

这样 Agent 可以直接调用展开后的工具。

## 5.9 多 MCP Server 命名冲突问题

如果一个 Agent 同时连接多个 MCP Server，就必须给每个 MCPTool 设置不同的 `name`。

例如：

```python
github_tool = MCPTool(name="gh", ...)
filesystem_tool = MCPTool(name="fs", ...)
```

这样工具会展开为：

```text
gh_search_repositories
fs_read_file
```

避免不同服务器中同名工具冲突。

## 5.10 MCP 社区生态

MCP 的优势之一是社区生态丰富。

本章提到的 MCP 资源包括：

1. **Awesome MCP Servers**：社区维护的 MCP 服务器列表；
2. **MCP Servers Website**：MCP 服务器目录网站；
3. **Official MCP Servers**：官方维护的 MCP 服务器集合。

常见 MCP Server 类型包括：

- 文件系统；
- GitHub；
- 数据库；
- Playwright；
- Notion；
- Slack；
- Obsidian；
- Jira；
- 浏览器自动化工具。

这说明 MCP 的真正价值不只是协议本身，而是它能形成可复用的工具生态。

---

# 6. MCP 实战案例：智能文档助手

本章中 MCP 实战案例是一个多 Agent 智能文档助手。

它包含两个 Agent：

## 6.1 GitHub 搜索专家

负责使用 GitHub MCP Server 搜索仓库。

```python
github_tool = MCPTool(
    name="gh",
    server_command=["npx", "-y", "@modelcontextprotocol/server-github"]
)
```

它的任务是搜索与 “AI agent” 相关的 GitHub 仓库，并返回结构化结果。

## 6.2 文档生成专家

负责根据搜索结果生成 Markdown 报告。

```python
fs_tool = MCPTool(
    name="fs",
    server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
)
```

它可以结合文件系统工具生成和保存报告。

## 6.3 案例体现的意义

这个案例展示了 MCP 在复杂任务中的价值：

```text
GitHub MCP Server 负责外部搜索
文件系统 MCP Server 负责本地文件操作
Agent 负责理解任务、组织结果和生成文档
```

开发者不用重复写 GitHub API 或文件读写适配器，只需要把 MCP Server 接入 Agent 工具链。

---

# 7. 构建自定义 MCP Server

## 7.1 为什么要自定义 MCP Server？

虽然社区已经有很多 MCP Server，但真实项目中仍然需要自定义 MCP Server。

原因包括：

1. **封装业务逻辑**：把企业内部流程包装成标准工具；
2. **访问私有数据**：安全访问内部数据库和私有 API；
3. **性能优化**：针对高频调用做缓存、限流和优化；
4. **功能定制**：接入企业内部模型、算法或硬件。

## 7.2 自定义 MCP Server 的基本结构

本章以天气查询 MCP Server 为例。

基本步骤是：

```text
创建 MCPServer
定义工具函数
注册工具
运行服务器
```

示例结构：

```python
from hello_agents.protocols import MCPServer

weather_server = MCPServer(
    name="weather-server",
    description="真实天气查询服务"
)

def get_weather(city: str) -> str:
    # 查询天气并返回 JSON
    pass

def list_supported_cities() -> str:
    # 返回支持城市列表
    pass

weather_server.add_tool(get_weather)
weather_server.add_tool(list_supported_cities)

if __name__ == "__main__":
    weather_server.run()
```

## 7.3 在 Agent 中使用自定义 MCP Server

创建好 MCP Server 后，可以通过 MCPTool 接入 Agent：

```python
weather_tool = MCPTool(
    server_command=["python", "weather_mcp_server.py"]
)

assistant.add_tool(weather_tool)
```

这样 Agent 就可以通过自然语言调用天气工具，例如：

```text
北京今天天气怎么样？
```

## 7.4 发布 MCP Server

本章还介绍了将自定义 MCP Server 发布到 Smithery 的思路。

基本文件结构包括：

```text
weather-mcp-server/
├── README.md
├── LICENSE
├── Dockerfile
├── pyproject.toml
├── requirements.txt
├── smithery.yaml
└── server.py
```

其中：

- `pyproject.toml` 描述 Python 项目依赖；
- `Dockerfile` 负责容器化部署；
- `smithery.yaml` 描述 MCP Server 元信息、工具列表和运行方式。

发布后，其他开发者就可以发现并安装这个 MCP Server。

---

# 8. A2A 协议详解

## 8.1 A2A 解决什么问题？

A2A 解决的是：

```text
Agent 与 Agent 之间如何直接通信和协作
```

在多智能体任务中，常见角色包括：

- 研究员；
- 撰写员；
- 编辑；
- 技术专家；
- 销售顾问；
- 协调者。

如果所有通信都经过中央协调器，会有：

- 单点故障；
- 性能瓶颈；
- 扩展困难。

A2A 采用点对点通信，让 Agent 之间可以直接交互。

## 8.2 A2A 的核心概念

A2A 的核心抽象包括：

| 概念 | 含义 |
|---|---|
| Agent | 能提供服务或请求服务的智能体 |
| Skill | Agent 对外暴露的能力 |
| Task | 一个可跟踪的协作任务 |
| Artifact | 任务执行产生的结果或中间产物 |

A2A 还定义了任务生命周期，例如：

```text
创建 -> 协商 -> 代理 -> 执行中 -> 完成 / 失败
```

这让智能体之间的协作不只是聊天，而是可以进行任务委托、进度跟踪和异常处理。

## 8.3 A2A 实战案例

本章展示了多个案例：

### 1. 计算器 Agent

创建一个 A2A Server，暴露加法、乘法和信息查询技能。

### 2. 研究员、撰写员、编辑网络

三个 Agent 分别负责：

```text
研究 -> 写作 -> 编辑
```

通过 A2A Client 依次调用，完成内容生产流程。

### 3. 智能客服系统

由接待员 Agent 判断问题类型，再把问题转发给：

- 技术专家 Agent；
- 销售顾问 Agent。

这体现了 A2A 在多 Agent 分工中的价值。

## 8.4 A2A 与 MCP 的区别

| 对比点 | MCP | A2A |
|---|---|---|
| 通信对象 | Agent 与工具 / 资源 | Agent 与 Agent |
| 核心能力 | 工具发现和调用 | 任务委托和协作 |
| 典型场景 | 文件、数据库、GitHub、API | 研究员、写作者、编辑、专家系统 |
| 抽象重点 | Tools、Resources、Prompts | Task、Artifact、Skill |
| 主要目标 | 扩展工具能力 | 扩展协作能力 |

一句话：

```text
MCP 让 Agent 会用工具，A2A 让 Agent 会找其他 Agent 合作。
```

---

# 9. ANP 协议详解

## 9.1 ANP 解决什么问题？

ANP 解决的是：

```text
大规模开放 Agent 网络中的服务发现、路由和身份信任
```

当网络中有大量 Agent 时，系统要面对：

- 如何发现能完成任务的 Agent；
- 如何选择负载最低、成本最低或能力最强的 Agent；
- 如何让新 Agent 动态加入网络；
- 如何验证通信双方身份；
- 如何跨平台互操作。

## 9.2 ANP 的核心流程

本章介绍的 ANP 流程包括：

1. **服务发现与匹配**  
   Agent 通过公开发现服务，根据语义或能力描述找到目标 Agent。

2. **基于 DID 的身份验证**  
   请求方用私钥签名，服务方通过 DID 获取公钥并验证请求真实性。

3. **标准化服务执行**  
   验证通过后，双方按标准接口和数据格式交换数据。

## 9.3 ANP 实战案例

本章用 ANP 构建了分布式任务调度系统。

流程包括：

1. 创建服务发现中心；
2. 注册多个计算节点；
3. 为每个节点记录负载、CPU、内存、GPU 等元数据；
4. 创建任务调度 Agent；
5. 根据任务需求选择最佳节点。

例如：

- 大型深度学习训练任务优先选择有 GPU 的节点；
- 大量文本处理任务优先选择高内存节点；
- 轻量分析任务可以选择低负载普通节点。

## 9.4 ANP 的定位

ANP 更偏未来大规模智能体网络基础设施，目前生态还不如 MCP 成熟。

因此在实际项目中：

- 短期优先掌握 MCP；
- 多智能体协作时了解 A2A；
- 构建开放式智能体网络时再考虑 ANP。

---

# 10. 三种协议如何选择？

## 10.1 按需求选择

| 需求 | 推荐协议 |
|---|---|
| 访问文件、数据库、GitHub、API | MCP |
| 封装企业内部工具 | MCP |
| 多个 Agent 分工协作 | A2A |
| Agent 之间任务委托和协商 | A2A |
| 大规模 Agent 服务发现 | ANP |
| 去中心化 Agent 网络 | ANP |

## 10.2 按成熟度选择

当前更推荐优先学习和使用 MCP，因为：

- 生态更成熟；
- 官方和社区服务器更多；
- 工具接入价值直接；
- 在真实项目中更容易落地；
- 与 RAG、代码助手、文档助手、自动化工作流结合紧密。

A2A 和 ANP 更适合在理解多智能体系统架构时学习。

---

# 11. 本章对项目开发的启发

## 11.1 MCP 是 Agent 工具生态的关键入口

如果你要做 Agent 项目，不应该每次都从零写工具适配器。  
更好的方式是：

```text
优先查找是否已有 MCP Server
如果没有，再封装自定义 MCP Server
最后通过 MCPTool 接入 Agent
```

这可以显著降低工具接入成本。

## 11.2 工具描述质量决定 Agent 调用效果

MCP 虽然提供标准接口，但 Agent 是否会正确调用工具，仍然依赖工具描述质量。

因此要写清楚：

- 工具做什么；
- 什么时候使用；
- 参数含义；
- 返回结果格式；
- 错误情况。

## 11.3 MCP Server 适合封装企业内部能力

例如：

- 私有知识库查询；
- 内部数据库查询；
- 项目管理系统；
- 日志分析服务；
- 模型推理服务；
- 文件存储系统。

这些能力通过 MCP Server 封装后，可以被多个 Agent 和多个模型复用。

## 11.4 A2A 适合多 Agent 分工系统

如果系统中存在多个专业 Agent，例如：

```text
规划 Agent
检索 Agent
代码 Agent
审查 Agent
报告 Agent
```

A2A 可以作为它们之间通信和任务委托的协议基础。

## 11.5 ANP 更适合开放网络和分布式调度

如果未来要做“Agent 市场”或“Agent 网络”，ANP 的服务发现、身份认证和动态路由思想会更重要。

---

# 12. 面试高频问题

## Q1：MCP 是什么？

MCP 是 Model Context Protocol，是一种用于标准化连接大模型智能体和外部工具、资源、提示模板的通信协议。它让 Agent 可以通过统一方式发现工具、调用工具、读取资源，而不需要为每个外部服务单独写适配器。

## Q2：MCP 的 Host、Client、Server 分别是什么？

Host 是用户交互入口和模型运行环境，例如桌面应用或 Agent 平台；Client 负责与 MCP Server 进行协议通信；Server 负责提供具体工具、资源和提示模板，例如文件系统、GitHub 或数据库服务。

## Q3：MCP 的 Tools、Resources、Prompts 有什么区别？

Tools 是主动操作，用来执行任务；Resources 是被动数据源，用来提供信息；Prompts 是提示模板，用来提供可复用的任务指令或上下文模板。

## Q4：MCP 和 Function Calling 有什么区别？

Function Calling 是模型能力，负责判断是否调用工具以及生成工具参数；MCP 是工程协议，负责让外部工具以标准方式被发现、描述、连接和执行。两者是互补关系。

## Q5：为什么说 MCP 像 USB-C？

因为 MCP 统一了 Agent 与外部工具的连接方式。不同工具只要封装成 MCP Server，就可以被支持 MCP 的 Agent 或模型系统统一访问，就像 USB-C 统一了设备连接接口。

## Q6：A2A 和 MCP 最大区别是什么？

MCP 面向 Agent 与工具通信，核心是工具发现和调用；A2A 面向 Agent 与 Agent 通信，核心是任务委托、协商和协作。

## Q7：ANP 适合什么场景？

ANP 适合大规模开放 Agent 网络中的服务发现、动态路由和身份验证。例如很多计算节点、专家 Agent 或服务 Agent 组成一个网络，需要根据任务动态选择最合适的服务。

## Q8：实际项目中应该优先学哪个协议？

优先学 MCP。因为它生态更成熟，能直接解决工具接入问题，并且可以用于文件系统、GitHub、数据库、浏览器自动化、企业 API 等真实场景。A2A 和 ANP 更适合多智能体协作和开放网络架构设计。

---

# 13. 可用于项目表述的一句话

> 在 Agent 系统中引入 MCP 协议，将文件系统、GitHub、数据库和业务 API 封装为标准 MCP Server，并通过 MCPTool 自动展开为 Agent 可调用工具，从而降低工具集成成本，提升工具复用性和系统扩展性。

---

# 14. 总结

第十章的核心是：**Agent 要从单体智能应用走向复杂系统，就必须具备标准化通信能力。**

MCP、A2A 和 ANP 分别对应三个层次：

```text
MCP：Agent 与工具通信
A2A：Agent 与 Agent 通信
ANP：Agent 与智能体网络通信
```

其中 MCP 是当前最值得重点掌握的协议。它通过 Host、Client、Server 架构，把外部工具和资源封装为标准 MCP Server，并提供 Tools、Resources、Prompts 三类能力。HelloAgents 进一步通过 MCPTool 自动展开机制，让 Agent 可以像调用普通工具一样调用 MCP Server 中的能力。

从工程角度看，MCP 的意义不只是“多一种工具调用方式”，而是把工具生态从“手写适配器”推进到“协议化、可发现、可复用”的阶段。它为 Agent 工具生态、企业内部 Agent 平台、代码助手、知识库助手、自动化工作流等场景提供了重要基础。

---

# 15. 参考来源

- Datawhale《Hello-Agents》第十章：智能体通信协议
- Model Context Protocol 官方资料
- A2A Protocol 相关资料
- Agent Network Protocol 相关资料
