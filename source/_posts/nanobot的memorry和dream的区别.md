---
title: nanobot的Memory和Dream的区别
cover: /assets/posts/nanobot/项目架构.png
categories: agent
tags:
  - nanobot
---


这里的Dream指的是什么？
这里的 **Dream** 是 nanobot 的“长期记忆整理”后台任务，不是某个模型或第三方服务。

它会定期读取对话历史 `memory/history.jsonl`，结合当前的 `SOUL.md`、`USER.md` 和 `memory/MEMORY.md`，提炼并增量更新长期有效的信息，例如用户偏好、项目决策和稳定上下文。

默认每 2 小时运行一次，可在配置 `agents.defaults.dream` 中调整或关闭；也可手动运行 `/dream`。它的更改可通过 `/dream-log` 查看，并用 `/dream-restore` 恢复。

Memory / Dream 如何把长对话沉淀成长期记忆

nanobot 把“长对话沉淀为长期记忆”拆成两个阶段：

```text
正在进行的对话
      │
      │ 上下文接近上限
      ▼
Consolidator：压缩旧消息
      │
      ▼
memory/history.jsonl
      │
      │ Dream 定期或手动运行
      ▼
Dream：提炼、去重、纠错、分类
      │
      ▼
SOUL.md / USER.md / memory/MEMORY.md
```

### 1. 对话先保存在当前 Session

当前对话消息保存在 `session.messages` 中，属于短期记忆。它会优先保证当前对话连贯，不会每说一句话就修改长期文件。

当上下文接近模型限制时，`Consolidator` 会选择较早的、完整的用户回合进行压缩，避免从工具调用或一轮回复中间截断。

### 2. Consolidator 生成历史摘要

压缩后的内容追加到：

```text
memory/history.jsonl
```

例如：

```json
{
  "cursor": 42,
  "timestamp": "2026-04-03 00:02",
  "content": "- 用户偏好深色模式\n- 项目决定使用 PostgreSQL"
}
```

这个文件是“发生过什么”的归档，不是最终长期记忆。

如果摘要模型失败、超时或返回无效结果，nanobot 会把受限长度的原始消息以 `[RAW]` 形式保存，避免信息丢失。

### 3. Dream 提炼长期信息

Dream 定期运行，默认每 2 小时一次，也可以用 `/dream` 手动触发。

它读取：

- `history.jsonl` 中尚未处理的新记录
- 当前的 `SOUL.md`
- 当前的 `USER.md`
- 当前的 `memory/MEMORY.md`

然后根据内置规则进行：

- 提取原子事实
- 删除重复内容
- 用新事实替换过时或冲突的事实
- 判断哪些内容应该长期保留
- 将内容放入正确的文件
- 删除已经失效的临时信息

文件职责大致是：

| 文件 | 保存内容 |
|---|---|
| `USER.md` | 用户身份、偏好、语言、沟通习惯 |
| `SOUL.md` | Agent 的行为规则、交互方式、工具策略 |
| `memory/MEMORY.md` | 项目目标、架构、决策和稳定上下文 |
| `skills/<name>/SKILL.md` | 可重复执行的具体工作流程 |

例如：

```text
用户说：“以后请用中文回答，尽量简洁。”
```

可能沉淀为：

```text
USER.md:
- 用户偏好使用中文交流。
- 用户偏好简洁回答。
```

而：

```text
我们决定项目使用 PostgreSQL。
```

会进入：

```text
memory/MEMORY.md:
- 项目数据库采用 PostgreSQL。
```

### 4. 使用游标避免重复处理

系统使用两个游标：

- `.cursor`：历史文件最新写入位置
- `.dream_cursor`：Dream 已经处理到的位置

Dream 只读取 `cursor > .dream_cursor` 的记录。只有在本轮模型执行完成、工具调用没有失败、文件修改成功后，才推进 `.dream_cursor`。

因此，如果 Dream 中途失败，这批历史会在下一次继续处理，不会被静默丢弃。

### 5. 长期记忆会被重新注入对话

之后的新对话构建 Prompt 时，会读取：

```text
SOUL.md
USER.md
memory/MEMORY.md
```

所以 Agent 获得的是整理后的稳定上下文，而不是把全部历史对话重新塞进上下文窗口。

### 6. 修改可审计、可恢复

Dream 修改长期文件后，`GitStore` 会记录实际文件差异：

```text
/dream-log
/dream-restore
```

这样可以查看 Dream 改了什么，必要时恢复到之前的版本。

需要注意：如果对话还没有触发 Consolidator，或者 `history.jsonl` 中没有新的未处理记录，Dream 就没有可整理的输入。这也是短对话执行 `/dream` 后可能提示“没有可处理历史”的原因。

Dream 是 **Agent 实例/用户级 Agent workspace 级别**，不是 WebUI 选中项目级别。

选中项目后，`ContextBuilder` 只从项目根读取 `AGENTS.md`，仍从 Agent workspace 读取全局 `SOUL.md`、`USER.md` 和 `memory/MEMORY.md`。

```text
Agent workspace（长期、跨项目）
├── SOUL.md
├── USER.md
├── memory/MEMORY.md
├── memory/history.jsonl
├── skills/
└── sessions/

当前项目（随 WebUI 选择变化）
├── AGENTS.md       <- 会进入本次 Prompt
├── 源码与产物
├── SOUL.md         <- 不自动读取
└── USER.md         <- 不自动读取
```

所以 Dream 处理的是同一个 Agent workspace 内跨项目累积的历史，并只维护：

- `SOUL.md`
- `USER.md`
- `memory/MEMORY.md`
- 在满足条件时创建/更新 Agent workspace 的 `skills/<name>/SKILL.md`

它不修改 `AGENTS.md`，也不管理项目目录的同名 `SOUL.md`、`USER.md` 或项目 `memory/`。



| 状态 | 位置 | 作用 |
|---|---|---|
| `SOUL.md`、`USER.md`、`memory/`、`skills/`、`sessions/` | Agent workspace | Agent 级持久状态；Dream 管理长期记忆 |
| `AGENTS.md` | 当前项目根目录 | 项目级指令；每次选中项目时注入 Prompt，Dream 不管理 |
| 项目源码与产物 | 当前项目根目录 | 工具操作对象，不属于 Dream 长期记忆 |