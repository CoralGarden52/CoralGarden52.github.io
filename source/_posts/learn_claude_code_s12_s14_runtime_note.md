---
title: Learn Claude Code 系统加固阅读笔记：s12–s14
cover: /assets/posts/Learn_Claude_Code/s12–s14.png
categories: agent
tags:
  - Learn_Claude_Code
---

# Learn Claude Code 任务运行时阅读笔记：s12–s14

## 一、总结

`s12–s14` 的主线是：**把 coding agent 从“会执行当前工具调用”，升级成“能长期组织任务、异步等待结果、按时间触发工作”的任务运行时系统**。

如果说前面章节解决的是：

```text
模型推理 → 工具调用 → 工具结果 → 继续推理
```

那么 `s12–s14` 进一步解决：

```text
任务图 → 后台执行 → 定时触发
```

一句话概括：

> 任务运行时的重点，不是让 agent 多会几个工具，而是让它能管理“要做什么、谁在跑、什么时候开始”。

---

## 二、面试速记

### 1. s12 任务系统解决什么？

`s12` 解决的是 todo 不够表达复杂工作关系的问题。

todo 适合当前会话内的临时步骤，而 task 适合跨阶段、跨轮次、多人协作的持久工作图。

它通过：

```text
TaskRecord
blockedBy
blocks
is_ready()
```

判断哪些任务已经可以开始。

### 2. s13 后台任务解决什么？

`s13` 解决慢命令阻塞主循环的问题。

例如：

```text
npm install
pytest
docker build
```

这类命令不应该让主循环一直等待，而应该通过 `background_run` 立即返回 `task_id`，后台执行完成后再通过通知队列把摘要带回模型。

### 3. s14 定时调度解决什么？

`s14` 解决“未来某个时间再开始做事”的问题。

后台任务是在等结果，定时调度是在等开始。调度器记录：

```text
cron
prompt
last_fired_at
```

时间匹配后把 prompt 放入通知队列，再由主循环统一处理。

### 4. task、background task、schedule 有什么区别？

```text
task：工作目标，关心要做什么、谁依赖谁、进度如何

background task：运行中的执行单元，关心命令是否还在跑、结果何时回来

schedule：未来触发入口，关心什么时候把一条意图重新送回主循环
```

这三者相关，但不能混成一个对象。

---

## 三、s12：任务系统

`s12` 的核心思想是：

```text
todo 是当前会话计划，task 是持久化工作图。
```

todo 只能表达“接下来做什么”，但不擅长表达：

```text
谁依赖谁
谁完成后会解锁谁
哪些任务已经 ready
哪些任务仍然 blocked
```

所以任务系统的核心不是保存清单，而是维护一张任务依赖图。

最关键的数据结构是 `TaskRecord`：

```json
{
  "id": 1,
  "subject": "Write parser",
  "description": "",
  "status": "pending",
  "blockedBy": [],
  "blocks": [],
  "owner": ""
}
```

其中：

```text
blockedBy：当前任务还在等谁
blocks：当前任务完成后会解锁谁
status：pending / in_progress / completed / deleted
```

最核心规则是：

```python
def is_ready(task):
    return task["status"] == "pending" and not task["blockedBy"]
```

这条规则回答的是：

> 哪条任务现在已经满足开工条件。

`s12` 的重点可以压缩成四句话：

```text
任务要落盘，不要只存在 messages 里
依赖关系要双向维护：blockedBy + blocks
完成任务后要自动解锁后续任务
task 是长期工作板，不是当前运行中的命令
```

---

## 四、s13：后台任务

`s13` 的核心思想是：

```text
慢命令可以在后台跑，主循环不必陪着等待。
```

前面的工具调用大多是同步的：

```text
模型发起工具调用
  → 立刻执行
  → 立刻返回结果
```

但对于 `pytest`、`npm install`、`docker build` 这类慢任务，同步等待会阻塞整个 agent。

因此，`s13` 引入后台任务系统。

最小结构是：

```text
主循环
  → background_run("pytest")
  → 立刻返回 task_id
  → 主循环继续工作

后台执行线
  → 真正运行 pytest
  → 完成后写入通知队列
  → 下一轮模型调用前注入摘要
```

最关键的数据结构是 `RuntimeTaskRecord`：

```json
{
  "id": "a1b2c3d4",
  "command": "pytest",
  "status": "running",
  "started_at": 1710000000.0,
  "result_preview": "",
  "output_file": ""
}
```

这里要注意两点：

```text
通知只放摘要
完整输出写入文件
```

因为后台任务可能输出几万行日志，不能直接全部塞进上下文。更合理的方式是：

```text
完整日志 → output_file
简短摘要 → notification
模型需要全文时 → 再 read_file
```

`s13` 最重要的一句话是：

> 主循环仍然只有一条，并行的是等待和执行槽位，不是主循环本身。

---

## 五、s14：定时调度

`s14` 的核心思想是：

```text
时间也可以成为触发 agent 工作的入口。
```

`s13` 解决的是“慢命令已经开始，结果什么时候回来”；  
`s14` 解决的是“某件事应该在未来什么时候开始”。

典型需求包括：

```text
每天晚上跑一次测试
每周一早上生成报告
30 分钟后提醒继续检查结果
```

定时调度的最小链路是：

```text
schedule_create(...)
  → 保存调度记录
  → 定时检查器每分钟检查
  → 时间匹配后写入通知队列
  → 主循环下一轮把 prompt 当作用户消息交给模型
```

关键数据结构是 `ScheduleRecord`：

```json
{
  "id": "job_001",
  "cron": "0 9 * * 1",
  "prompt": "Run the weekly status report.",
  "recurring": true,
  "durable": true,
  "created_at": 1710000000.0,
  "last_fired_at": null
}
```

其中 `last_fired_at` 很重要。没有它，系统可能在短时间内重复触发同一条任务。

`s14` 的重点不是 cron 语法，而是：

```text
如何把未来意图持久化
如何在时间到达时生成通知
如何让通知重新回到同一条主循环
```

所以定时调度不是新的 agent，也不是新的主循环，而是多了一种“未来启动入口”。

---

## 六、任务运行时总图

```text
s12 任务系统
  把 todo 升级为持久任务图
  解决“要做什么、谁依赖谁、谁可以开始”

s13 后台任务
  把慢命令移到后台执行
  解决“谁正在跑、结果什么时候回来”

s14 定时调度
  把未来时间变成触发入口
  解决“什么时候开始做这件事”
```

三者组合起来后，agent 不再只是一个即时问答工具，而是有了更完整的任务运行时：

```text
任务图负责组织目标
后台任务负责异步执行
定时调度负责未来触发
通知队列负责统一回到主循环
```

---

## 七、一句话结论

`Learn Claude Code s12–s14` 的核心不是复杂调度算法，而是把 agent 的工作拆成三层：**任务目标层、运行执行层、时间触发层**。

真正稳定的 coding agent，必须能区分“要做什么”“谁正在跑”“什么时候开始”，并让所有结果最终通过通知队列回到同一条主循环。
