---
title: Learn Claude Code 核心闭环阅读笔记：s01–s06
cover: /assets/posts/Learn_Claude_Code/s01–s06.png
categories: agent
tags:
  - Learn_Claude_Code
---

# Learn Claude Code 核心闭环阅读笔记：s01–s06

## 一、总结

`s01–s06` 讲的是 Claude Code 类智能体的**最小核心闭环**：  
从“模型会回答”逐步升级为“模型能使用工具、维护计划、派发子任务、按需加载技能、压缩上下文”的 coding agent。

核心可以概括为：

```text
用户任务
  → 模型推理
  → 调用工具
  → 工具真实执行
  → tool_result 写回 messages
  → 模型基于结果继续推理
  → 计划、子代理、技能、压缩不断增强这个循环
```

最重要的一句话是：

> Agent 的关键不是模型输出了什么，而是系统能不能把真实执行结果重新喂回模型，让它持续推进任务。

---

## 二、面试速记

### 1. Claude Code 的最小 agent loop 是什么？

就是：

```text
messages → model → tool_use → execute tool → tool_result → messages → next turn
```

其中最关键的是：工具结果不能只打印出来，而必须写回消息历史，成为下一轮模型推理的输入。

### 2. 为什么要做工具路由？

因为工具能力不应该写死在循环里。  
正确做法是用 `TOOL_HANDLERS` 这种 dispatch map，把工具名映射到具体处理函数。

这样新增工具时只需要新增 handler 和 schema，主循环不用改。

### 3. todo 解决什么问题？

todo 解决多步骤任务容易跑偏的问题。

它不是完整任务系统，而是当前会话内的轻量计划状态，用：

```text
pending / in_progress / completed
```

帮助模型明确当前正在做什么。

### 4. 子代理有什么价值？

子代理的价值不是“多一个模型”，而是“多一个干净上下文”。

父 agent 把局部探索任务交给子 agent，子 agent 在独立 messages 中完成任务，最后只把摘要带回父上下文，避免主上下文被噪声污染。

### 5. skill 技能系统是什么？

skill 是某类任务的可复用说明书。

系统平时只把 skill 名称和描述放进 prompt，让模型知道有哪些技能；真正需要时再通过 `load_skill` 加载完整正文。

这样可以避免把所有知识一次性塞进 system prompt。

### 6. 上下文压缩解决什么问题？

解决工具输出、文件内容、多轮历史不断膨胀的问题。

压缩不是简单删除历史，而是保留继续工作所需的信息：  
大输出写磁盘只留预览，旧工具结果替换成占位，历史过长时生成连续性摘要。

---

## 三、s01：Agent 循环

`s01` 是整个系统的起点。

语言模型本身只会生成文本，不会自动打开文件、运行命令、观察报错，也不会把工具结果用于下一步推理。要让它变成 agent，就必须在外层写一个循环。

最小循环包含五步：

```text
1. 用户请求写入 messages
2. 把 messages、system、tools 发给模型
3. assistant 回复写回 messages
4. 如果模型调用工具，就执行工具
5. tool_result 作为 user message 写回 messages
```

这里最关键的是两点：

```text
assistant 回复要写回历史
tool_result 也要写回历史
```

如果工具结果没有回到 `messages`，模型下一轮就看不到真实执行结果，agent loop 就断了。

---

## 四、s02：工具使用

`s02` 把单一 `bash` 工具升级为多个专用工具，例如：

```text
read_file
write_file
edit_file
bash
```

这样做的原因是：所有操作都走 shell 不稳定，也不安全；专用工具可以做路径沙箱、输出限制和输入校验。

核心代码思想是：

```python
TOOL_HANDLERS = {
    "bash": run_bash,
    "read_file": run_read,
    "write_file": run_write,
    "edit_file": run_edit,
}
```

主循环不需要关心每个工具怎么实现，只需要根据 `tool_use.name` 找到对应 handler。

所以 s02 的重点是：

```text
加工具 = 加 schema + 加 handler
主循环保持不变
```

---

## 五、s03：待办写入

`s03` 给 agent 增加了会话级计划能力。

多步骤任务中，模型很容易做着做着忘记目标、重复检查，或者从计划执行变回即兴发挥。todo 的作用就是把“当前要做什么”从模型脑内移到系统可观察状态里。

最小 todo 结构是：

```json
[
  {"content": "阅读项目结构", "status": "completed"},
  {"content": "定位报错原因", "status": "in_progress"},
  {"content": "修改并运行测试", "status": "pending"}
]
```

关键约束是：

```text
同一时间最多一个 in_progress
```

这样可以强制模型聚焦当前步骤。

如果连续几轮没有更新计划，系统可以插入 reminder，提醒模型刷新 todo。

---

## 六、s04：子代理

`s04` 解决的是上下文污染问题。

父 agent 在完成一个局部问题时，可能会读很多文件、跑很多命令，但最终有价值的结果只是一句结论。如果所有中间过程都留在父上下文里，后续任务会被大量噪声干扰。

子代理的核心机制是：

```text
父 agent 发现局部任务
  → 调用 task 工具
  → 子 agent 用新的 messages 执行任务
  → 子 agent 返回 summary
  → 父 agent 只接收必要结果
```

这里最重要的是：

```text
子代理不是共享父 messages
而是从干净上下文开始
```

因此，子代理主要用于搜索、阅读、检查、局部分析等探索性任务。

---

## 七、s05：技能系统

`s05` 解决的是“知识如何进入上下文”的问题。

不同任务需要不同规则，例如代码审查、Git 工作流、MCP 集成。如果把所有说明都塞进 system prompt，会浪费 token，也会让主规则变得混乱。

技能系统分成两层：

```text
轻量发现层：
  只放 skill 名称和一句描述

按需加载层：
  需要时再读取完整 SKILL.md
```

最小结构可以是：

```text
skills/
  code-review/
    SKILL.md
  git-workflow/
    SKILL.md
```

skill、memory、CLAUDE.md 的边界也很重要：

```text
skill：某类任务才需要的可选知识包
memory：跨会话长期有用的信息
CLAUDE.md：更稳定的全局规则
```

---

## 八、s06：上下文压缩

`s06` 解决 agent 运行一段时间后上下文膨胀的问题。

工具输出、大文件内容、多轮历史都会不断挤占上下文，导致请求变贵、注意力分散，甚至撞上上下文上限。

它的压缩分三层：

```text
1. 大工具结果不直接塞进上下文
   → 写到磁盘，只保留预览

2. 旧工具结果不一直完整保留
   → 替换成简短占位

3. 整体历史过长
   → 生成连续性摘要
```

压缩的目标不是“删历史”，而是保住继续工作需要的信息，例如：

```text
当前目标是什么
已经做了什么
改过哪些文件
还有什么没完成
哪些决定不能丢
```

---

## 九、核心闭环总图

```text
s01 Agent Loop
  建立 messages → model → tool_result → messages 的最小闭环

s02 Tool Use
  用 dispatch map 扩展工具能力，主循环不变

s03 Todo
  把当前计划外显，防止多步骤任务漂移

s04 Subagent
  用独立上下文处理局部任务，减少父上下文噪声

s05 Skill
  先展示技能目录，需要时再加载完整知识

s06 Compact
  大输出落盘、旧结果占位、长历史摘要，保持上下文可控
```

---

## 十、一句话结论

`Learn Claude Code s01–s06` 的主线不是“写更多工具”，而是围绕同一个 agent loop，不断增强它的**执行能力、计划能力、上下文隔离能力、知识加载能力和上下文控制能力**。

真正的 coding agent，本质上是一个能持续观察真实结果、更新状态、压缩历史并继续推进任务的闭环系统。
