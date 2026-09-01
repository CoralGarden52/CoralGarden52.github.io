---
title: Learn Claude Code 系统加固阅读笔记：s15–s19
cover: /assets/posts/Learn_Claude_Code/s15–s19.png
categories: agent
tags:
  - Learn_Claude_Code
---

# Learn Claude Code 多 Agent 平台阅读笔记：s15–s19

## 一、总结

`s15–s19` 的主线是：**把单个 coding agent 扩展成一个可协作、可自治、可隔离、可接入外部能力的多 Agent 平台**。

如果说前面章节已经完成了：

```text
核心闭环 → 系统加固 → 任务运行时
```

那么 `s15–s19` 进一步补齐的是：

```text
长期队友 → 团队协议 → 自主认领 → Worktree 隔离 → MCP 插件
```

一句话概括：

> 多 Agent 平台的重点不是“多开几个模型”，而是让多个 agent 有身份、有协议、有任务边界、有独立工作区，并且能接入外部工具生态。

---

## 二、面试速记

### 1. s15 Agent 团队解决什么？

`s15` 解决的是“长期队友”问题。

`s04` 的 subagent 是一次性委派，执行完就消失；而 teammate 是有名字、角色、邮箱和独立循环的持久 agent。

团队系统至少需要三件事：

```text
名册
邮箱
独立循环
```

### 2. s16 团队协议解决什么？

`s16` 解决自由文本协作不可追踪的问题。

多个 agent 之间不能只靠“随便说一句”，而要用 `request_id` 把请求、批准、拒绝、超时对齐。

核心是：

```text
协议消息 + 请求追踪表 + 状态机
```

### 3. s17 自主代理解决什么？

`s17` 解决 lead 手动分配任务的瓶颈。

空闲队友可以扫描任务板，按角色、任务状态、阻塞关系判断是否可认领，然后把任务标记为 `in_progress`。

自治不是乱抢任务，而是在规则内自动找活。

### 4. s18 Worktree 隔离解决什么？

`s18` 解决多个 agent 并行改代码时互相污染的问题。

任务板回答“做什么”，worktree 回答“在哪做”。

每个任务绑定独立工作目录和分支，方便并行开发、回看变更和收尾清理。

### 5. s19 MCP 与插件解决什么？

`s19` 解决工具不能都写死在主程序里的问题。

MCP 把工具来源从“本地硬编码”升级为“外部可插拔”。

但外部工具仍然必须进入同一个工具路由、权限管道和结果格式，不能绕过主控制面。

---

## 三、s15：Agent 团队

`s15` 的核心思想是：

```text
subagent 是一次性执行单元，teammate 是长期存在的协作成员。
```

`s04` 的 subagent 适合临时探索：

```text
创建 → 执行 → 返回摘要 → 消失
```

但多 Agent 平台需要的是长期队友：

```text
有名字
有角色
有邮箱
有自己的 messages
有自己的 agent loop
```

最小团队系统包含三层：

```text
1. 名册：记录团队里有谁
2. 邮箱：队友之间如何发消息
3. 独立循环：每个队友有自己的上下文和工作循环
```

最小数据结构：

```json
{
  "name": "alice",
  "role": "coder",
  "status": "working"
}
```

邮箱可以先用 JSONL 文件实现：

```text
.team/inbox/alice.jsonl
.team/inbox/bob.jsonl
```

队友每一轮先读取 inbox，再进入自己的 agent loop。这样它不是靠“重新创建”获得新任务，而是通过“下一轮检查邮箱”持续接收工作。

核心结论：

> teammate 的关键不是多一次模型调用，而是多一个长期存在、可反复接活的执行者。

---

## 四、s16：团队协议

`s16` 的核心思想是：

```text
有了邮箱，团队能说话；有了协议，团队才会按规矩协作。
```

如果所有协作都靠自由文本，会有两个问题：

```text
某些动作必须明确批准或拒绝
多个请求并发时，很难知道回复对应哪件事
```

所以需要结构化协议。

最小协议消息：

```json
{
  "type": "shutdown_request",
  "from": "lead",
  "to": "alice",
  "request_id": "req_001",
  "payload": {},
  "timestamp": 1710000000.0
}
```

同时还要有请求追踪表：

```json
{
  "request_id": "req_001",
  "kind": "shutdown",
  "from": "lead",
  "to": "alice",
  "status": "pending"
}
```

最小状态机：

```text
pending → approved
pending → rejected
pending → expired
```

它可以用于优雅关机，也可以用于高风险计划审批。本质都是：

```text
发起请求 → 等待响应 → 更新状态 → 继续流程
```

核心结论：

> 团队协议不是增加聊天格式，而是让协作流程可追踪、可恢复、可调试。

---

## 五、s17：自主代理

`s17` 的核心思想是：

```text
一个团队真正开始自己运转，不是因为 agent 数量变多，而是因为空闲队友会自己找下一份工作。
```

到 `s16` 为止，团队已经有：

```text
持久队友
邮箱
协议
任务板
```

但如果所有任务仍然要 lead 手动分配，lead 就会成为瓶颈。因此 `s17` 增加自主认领。

最重要的是 `is_claimable_task`：

```python
def is_claimable_task(task: dict, role: str | None = None) -> bool:
    return (
        task.get("status") == "pending"
        and not task.get("owner")
        and not task.get("blockedBy")
        and _task_allows_role(task, role)
    )
```

也就是说，一条任务能被认领，必须满足：

```text
任务还没开始
还没人认领
没有前置阻塞
当前队友角色满足要求
```

认领后任务会变成：

```json
{
  "id": 7,
  "owner": "alice",
  "status": "in_progress",
  "claimed_at": 1710000000.0,
  "claim_source": "auto"
}
```

这里 `claim_source="auto"` 很重要，它说明任务是自治认领，而不是人工分配。

系统还会写入 `claim_events.jsonl`，方便回看自治系统做过什么。

自主认领必须是原子操作。否则两个队友可能同时看到同一个空任务，然后都把自己写成 owner。最小实现也要加锁。

核心结论：

> 自治不是让 agent 随便行动，而是让它在角色、状态、依赖和锁的约束下安全认领任务。

---

## 六、s18：Worktree 隔离

`s18` 的核心思想是：

```text
任务板解决“做什么”，worktree 解决“在哪做而不互相踩到”。
```

当多个 agent 并行做代码任务时，如果都在同一个目录里修改文件，很容易出现：

```text
两个任务同时改同一个文件
未完成修改污染其他任务
难以单独回看某个任务的改动范围
```

因此需要给任务绑定独立 worktree。

任务记录示例：

```json
{
  "id": 12,
  "subject": "Refactor auth flow",
  "status": "in_progress",
  "owner": "alice",
  "worktree": "auth-refactor",
  "worktree_state": "active",
  "last_worktree": "auth-refactor",
  "closeout": null
}
```

worktree 注册表示例：

```json
{
  "name": "auth-refactor",
  "path": ".worktrees/auth-refactor",
  "branch": "wt/auth-refactor",
  "task_id": 12,
  "status": "active"
}
```

这里要分清：

```text
TaskRecord：记录任务目标、负责人和状态
WorktreeRecord：记录执行车道、目录、分支和生命周期
```

Worktree 不只是“多开一个目录”，还要记录进入时间、最近命令、收尾动作和事件日志。这样任务结束后可以选择保留目录、回收目录，或者记录删除失败。

核心结论：

> 多 Agent 并行开发时，隔离目录不是可选优化，而是避免互相污染的基础设施。

---

## 七、s19：MCP 与插件

`s19` 的核心思想是：

```text
工具不必都写死在主程序里，外部进程也可以把能力接进 agent。
```

MCP 可以理解成一套让 agent 和外部工具程序对话的统一协议。

最小流程是：

```text
启动外部工具服务进程
查询它有哪些工具
模型需要时转发工具调用
把结果带回 agent 主循环
```

工具路由变成：

```text
LLM
  → tool_use
  → Agent tool router
      → native tool：本地 Python handler
      → MCP tool：外部 MCP server
  → tool_result
  → 回到主循环
```

为了避免命名冲突，MCP 工具通常使用前缀：

```text
mcp__{server}__{tool}
```

例如：

```text
mcp__postgres__query
mcp__browser__open_tab
```

Plugin 则解决“外部工具配置怎么被发现”的问题。最小插件配置可以是：

```text
.claude-plugin/
  plugin.json
```

其中记录插件名、版本、MCP server 以及启动命令。

但最关键的一点是：**MCP 工具不能绕开权限系统**。

无论工具来自本地还是外部，都必须走同一个权限闸门、同一个工具路由、同一个结果标准化流程。

核心结论：

> MCP 不是外挂，而是把外部能力接入同一条控制面和执行面。

---

## 八、多 Agent 平台总图

```text
s15 Agent 团队
  建立长期队友：名册 + 邮箱 + 独立循环

s16 团队协议
  用 request_id 和状态机，让协作可追踪、可审批、可恢复

s17 自主代理
  空闲队友按角色和任务状态自动认领 ready task

s18 Worktree 隔离
  每个任务绑定独立执行车道，避免并行修改互相污染

s19 MCP 与插件
  外部工具通过统一协议接入，同样走权限和路由
```

---

## 九、一句话结论

`Learn Claude Code s15–s19` 的重点不是简单做“多智能体”，而是把多 agent 拆成五个工程问题：**谁长期存在、怎么协作、怎么自己找活、在哪里隔离执行、如何接入外部能力**。

真正可用的多 Agent 平台，必须让身份、协议、任务、worktree 和工具路由都进入同一套可观察、可恢复、可控制的系统。
