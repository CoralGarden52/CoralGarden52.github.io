---
title: Learn Claude Code 系统加固阅读笔记：s07–s11
cover: /assets/posts/Learn_Claude_Code/s07–s11.png
categories: agent
tags:
  - Learn_Claude_Code
---

# Learn Claude Code 系统加固阅读笔记：s07–s11

## 一、总结

`s07–s11` 的主线是：**把一个能跑的 coding agent，加固成一个更安全、更可维护、更可恢复的系统**。

如果说 `s01–s06` 解决的是核心闭环：

```text
模型推理 → 工具调用 → 工具结果 → 继续推理
```

那么 `s07–s11` 解决的是系统加固：

```text
权限控制 → Hook 扩展 → 长期记忆 → Prompt 组装 → 错误恢复
```

一句话概括：

> 系统加固的重点不是让 agent 做更多事，而是让 agent 在“能做事”的基础上，做得更安全、更可控、更稳定。

---

## 二、面试速记

### 1. s07 权限系统解决什么？

解决“模型产生的工具调用不能直接执行”的问题。

任何工具调用都应该先经过权限管道，再决定是：

```text
allow / ask / deny
```

最小流程是：先检查 deny rules，再看当前模式，再看 allow rules，最后剩余灰区交给用户确认。

### 2. s08 Hook 系统解决什么？

解决“不改主循环也能扩展系统行为”的问题。

Hook 本质是主循环在固定时机暴露出来的扩展点，例如：

```text
SessionStart
PreToolUse
PostToolUse
```

主循环只负责发事件，具体附加逻辑交给 hook runner。

### 3. s09 记忆系统解决什么？

解决“哪些信息应该跨会话留下”的问题。

Memory 不是把所有有用信息都存下来，而是只保存：

```text
跨会话仍然有价值
且不能轻易从当前仓库重新推导的信息
```

用户偏好、用户纠错、项目隐含约定、外部资源指针适合保存；文件结构、当前任务进度、临时分支名、凭证不适合保存。

### 4. s10 系统提示词解决什么？

解决 system prompt 不可维护的问题。

系统提示词不应该是一大段硬编码字符串，而应该是一条输入组装流水线：

```text
core + tools + skills + memory + CLAUDE.md + dynamic_context
```

这样才能区分稳定规则、动态提醒、长期记忆和技能信息。

### 5. s11 错误恢复解决什么？

解决“报错就崩”的问题。

错误恢复不是掩盖错误，而是先判断错误类型，再选择恢复路径：

```text
输出被截断 → 续写
上下文太长 → 压缩后重试
网络或限流问题 → 退避重试
```

每条恢复路径都要有重试预算，防止无限循环。

---

## 三、s07：权限系统

`s07` 的核心思想是：

```text
模型可以提出行动意图，但真正执行前必须经过权限检查。
```

最小权限系统不需要一开始就做得很复杂，只需要四步：

```text
tool_call
  → deny rules
  → mode check
  → allow rules
  → ask user
```

推荐先实现三种模式：

```text
default：未命中规则时问用户
plan：只允许读，不允许写
auto：简单安全操作自动放行，危险操作再问
```

其中 `bash` 要被特殊对待，因为它不是普通文本，而是可执行动作描述。

至少要先拦截：

```text
sudo
rm -rf
命令替换
可疑重定向
高风险网络命令
```

核心结论：

> 权限系统不是为了让 agent 更笨，而是为了让 agent 的行动先经过可靠的安全判断。

---

## 四、s08：Hook 系统

`s08` 的核心思想是：

```text
主循环只负责暴露时机，额外行为交给 hook。
```

Hook 适合处理那些不应该硬塞进主循环的扩展逻辑，例如：

```text
会话开始时做初始化
工具执行前做额外检查
工具执行后写审计日志
工具失败后补充提示
```

教学版先掌握三个事件就够了：

```text
SessionStart
PreToolUse
PostToolUse
```

最小返回协议可以统一成：

```text
0：正常继续
1：阻止当前动作
2：注入一条补充消息，再继续
```

这套设计让系统具备三种扩展能力：

```text
观察
拦截
补充
```

核心结论：

> Hook 不是主循环的替代品，而是主循环在固定时机对外发出的扩展调用。

---

## 五、s09：记忆系统

`s09` 的核心思想是：

```text
不是所有信息都该进入 memory。
```

Memory 只适合保存两类信息：

```text
跨会话仍然有价值
不能轻易从当前代码或环境重新推导
```

适合保存的 memory 有四类：

```text
user：用户长期偏好
feedback：用户明确纠正过的问题
project：不容易从代码看出的项目约定
reference：外部资源指针
```

不适合保存的内容包括：

```text
文件结构
函数签名
当前任务进度
临时分支名
当前 PR 号
修 bug 的具体代码细节
密钥、密码、凭证
```

最小实现可以用 `.memory/` 目录，每条 memory 一个文件，再用 `MEMORY.md` 做索引。

记忆不是绝对真相。如果 memory 和当前代码状态冲突，应该优先相信当前真实观察。

核心结论：

> Memory 用来提供长期方向，不应该替代当前观察。

---

## 六、s10：系统提示词

`s10` 的核心思想是：

```text
system prompt 不是固定大字符串，而是逐段组装出来的输入流水线。
```

随着 agent 功能增加，输入来源会越来越多：

```text
工具列表会变
skills 会变
memory 会变
当前目录、日期、模式会变
CLAUDE.md 规则会变
```

因此应该用 `SystemPromptBuilder` 管理输入组装：

```text
core
+ tools
+ skills
+ memory
+ CLAUDE.md
+ dynamic_context
= final system prompt
```

这里最重要的边界是：

```text
稳定说明 vs 动态提醒
system prompt vs system reminder
skills vs memory vs CLAUDE.md
```

其中：

```text
skills：可选能力或知识包
memory：跨会话记住的信息
CLAUDE.md：长期规则说明
dynamic_context：本轮动态状态，如日期、cwd、当前模式
```

核心结论：

> Prompt 工程不是写一大段神秘文本，而是把不同来源的信息按职责拼成可维护的输入流水线。

---

## 七、s11：错误恢复

`s11` 的核心思想是：

```text
错误不是例外，而是主循环必须预留出来的一条正常分支。
```

最小错误恢复先区分三类问题：

```text
输出被截断
上下文太长
临时连接失败
```

对应三条恢复路径：

```text
max_tokens
  → 注入续写提示
  → 再试一次

prompt too long
  → 压缩旧上下文
  → 再试一次

timeout / rate limit / transient API error
  → 退避等待
  → 再试一次
```

关键是要维护恢复状态，例如：

```text
continuation_attempts
compact_attempts
transport_attempts
```

它的作用不是记录所有错误，而是防止无限重试，并让每种恢复路径都有自己的预算。

核心结论：

> 稳定的 agent 不是不会出错，而是知道出错后该续写、该压缩、该退避，还是该明确失败。

---

## 八、系统加固总图

```text
s07 权限系统
  工具调用不能直接执行，必须先经过 allow / ask / deny 判断

s08 Hook 系统
  在固定时机插入扩展逻辑，避免主循环越写越重

s09 记忆系统
  只保存跨会话仍然有价值、且不易重新推导的信息

s10 系统提示词
  把 prompt 从固定大字符串升级成输入组装流水线

s11 错误恢复
  把报错就崩升级成分类恢复、有限重试、明确失败
```

---

## 九、一句话结论

`Learn Claude Code s07–s11` 的重点不是继续堆工具，而是给 agent 加上安全边界、扩展点、长期记忆、输入组装和恢复机制。

真正可用的 coding agent，不仅要能调用工具完成任务，还要能判断哪些动作能执行、哪些信息该保留、哪些输入该进入 prompt，以及失败后如何继续推进。
