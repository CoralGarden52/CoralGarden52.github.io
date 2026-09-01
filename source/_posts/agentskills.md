---
title: Agent Skills阅读笔记
cover: /assets/posts/agentskills/agentskills-cover.png
categories: agent
---
## 面试速记

这篇文章可以按“是什么、为什么、怎么设计、怎么评估、怎么落地”来记。Agent Skills 本质上是一种给 AI Agent 扩展能力的开放格式，把领域知识、项目规范、固定流程、脚本、模板和参考资料封装成可复用的技能包，让 Agent 在需要时按需加载，而不是每次都重新推理。

1. 网站/项目核心：Agent Skills 解决的是 Agent 缺少项目上下文、团队规范、领域知识和工具调用方式的问题。面试时可以说，它把一次性的 prompt 经验沉淀成可版本化、可复用、可协作的工程资产。

2. Progressive Disclosure：这是核心机制。启动时只暴露 `name` 和 `description`，命中后加载 `SKILL.md`，执行时再读取 `scripts/`、`references/`、`assets/` 等资源。优点是节省上下文、降低噪音、提高技能触发精度。

3. 标准结构：最小结构是一个目录加 `SKILL.md`。`scripts/` 放可执行脚本，`references/` 放长文档，`assets/` 放模板、图片、数据等资源。`description` 很关键，因为 Agent 主要靠它判断什么时候加载技能。

4. Quickstart：创建第一个 Skill 的重点不是复杂代码，而是清楚描述技能何时使用、如何执行。一个很轻量的 `SKILL.md` 也能工作，说明 Skills 的门槛低，适合把小而稳定的流程先沉淀下来。

5. 最佳实践：Skill 的粒度要像设计函数一样，既不能太窄导致一次任务加载很多技能，也不能太宽导致误触发。内容应写项目特定规则、易错点、边界条件和真实失败案例，而不是解释通用概念。

6. 有效指令写法：高质量 Skill 通常包含 gotchas、输出模板、checklist、validation loop、plan-validate-execute。面试时可以强调，给 Agent 写指令不是堆说明，而是把流程、格式、验证和异常处理结构化。

7. 优化 `description`：`description` 决定技能是否会被正确激活。好的描述要说明“什么时候用”和“什么时候不用”，避免太宽导致误触发，太窄导致漏触发。可以用正负样本测试触发率。

8. 评估 Skill：评估不能只看主观感觉，要设计 benchmark、assertion 和对照实验。把使用 Skill 与不使用 Skill 的输出对比，结合 failed assertions、人类反馈和执行日志迭代改进。

9. 使用脚本：脚本适合处理可重复、可验证、容易出错的操作。脚本要面向非交互式 Agent 环境，避免密码框、确认菜单、TTY prompt；复杂命令应封装进 `scripts/` 并尽量固定工具版本。

10. 客户端接入：客户端支持 Skills 的流程包括发现技能目录、解析 `SKILL.md` frontmatter、构造 skill catalog、激活技能、管理技能上下文。安全上要注意项目级 Skills 可能来自不可信仓库，加载前应有信任机制。


11. 总体价值：Agent Skills 的价值不只是“让模型知道更多”，而是把 Agent 的行为约束、知识、工具和验证方式工程化。它让 Agent 更稳定、更可复用、更容易团队协作，也更容易评估质量。
下面是我对 **agentskills.io** 的阅读总结，原站链接：https://agentskills.io/
## 1. 这个网站/项目在讲什么？

**Agent Skills** 是一种给 AI Agent 扩展能力的开放格式。它的核心思想很简单：把某一类任务所需的专业知识、流程、脚本、模板、参考资料，打包成一个文件夹，让 Agent 在需要时加载使用。官方给出的最小结构是一个目录里包含 `SKILL.md`，可选地包含 `scripts/`、`references/`、`assets/` 等目录。

可以把它理解为：

> Agent 的“可复用技能包”或“任务插件规范”。

它解决的问题是：Agent 本身虽然有通用能力，但在实际项目中经常缺少**项目上下文、团队规范、固定流程、领域知识、工具调用方式**。Agent Skills 通过标准化目录和说明文件，把这些知识封装起来，使 Agent 能够按需调用，而不是每次都从零开始推理。

------

## 2. 核心机制：Progressive Disclosure 渐进式披露

Agent Skills 最重要的设计是 **渐进式披露**。也就是不是一开始把所有技能的完整内容都塞进上下文，而是分三层加载：

| 阶段       | 加载内容                       | 触发时机               | 作用                    |
| ---------- | ------------------------------ | ---------------------- | ----------------------- |
| Discovery  | 只加载 `name` 和 `description` | Agent 启动时           | 让 Agent 知道有哪些技能 |
| Activation | 加载完整 `SKILL.md`            | 用户任务匹配技能描述时 | 给 Agent 具体执行说明   |
| Execution  | 按需读取脚本、参考文件、模板等 | 执行任务时             | 完成复杂任务            |

这种设计可以让 Agent 同时拥有很多技能，但不会一开始消耗大量上下文。官方建议每个技能的 `SKILL.md` 主体控制在 5000 tokens 以内，主文件最好少于 500 行；更长的内容应该拆到 `references/` 或其他资源文件中。

------

## 3. Agent Skill 的标准结构

一个 Skill 本质上是一个目录，最小结构如下：

```text
skill-name/
├── SKILL.md
├── scripts/
├── references/
├── assets/
└── ...
```

其中只有 `SKILL.md` 是必需的。`scripts/` 用于放可执行代码，`references/` 用于放额外文档，`assets/` 用于放模板、图片、数据文件等资源。

`SKILL.md` 必须由两部分组成：

```markdown
---
name: skill-name
description: A description of what this skill does and when to use it.
---

这里写具体的执行说明。
```

Frontmatter 中：

| 字段            | 是否必需 | 说明                                                 |
| --------------- | -------- | ---------------------------------------------------- |
| `name`          | 必需     | 技能名，必须和目录名一致，只能小写字母、数字、连字符 |
| `description`   | 必需     | 描述技能做什么、什么时候用，是触发技能的关键         |
| `license`       | 可选     | 技能许可证                                           |
| `compatibility` | 可选     | 环境要求，例如 Python 版本、依赖工具、网络要求       |
| `metadata`      | 可选     | 自定义元数据                                         |
| `allowed-tools` | 可选     | 实验字段，表示预批准工具                             |

其中 `description` 非常关键，因为 Agent 在启动阶段只看 `name` 和 `description` 来判断什么时候加载技能。

------

## 4. Quickstart：如何创建第一个 Skill？

Quickstart 页面用“掷骰子”作为示例。它要求在项目中创建：

```text
.agents/skills/roll-dice/SKILL.md
```

示例内容大致是：

```markdown
---
name: roll-dice
description: Roll dice using a random number generator. Use when asked to roll a die...
---

To roll a die, use the following command...
```

然后在 VS Code + GitHub Copilot Agent mode 中输入 `/skills` 检查技能是否被识别，再让 Agent 执行 “Roll a d20”。这个例子说明，一个 Skill 可以非常轻量，甚至不到 20 行，只要描述清楚，Agent 就能在合适的时候激活它。

Quickstart 还强调：同一个 Skill 不只适用于 VS Code，Agent Skills 是开放格式，理论上可以被 Claude Code、OpenAI Codex 等兼容 Agent 使用。

------

## 5. Skill 创作者最佳实践

Best practices 页面强调：不要让 LLM 凭空写一个泛泛的 Skill，而应该从真实任务、真实项目资料、真实错误中提炼技能。好的 Skill 应该来自：

1. 真实执行过的任务流程；
2. 你对 Agent 的纠正记录；
3. 项目文档、API 规范、Schema、配置文件；
4. Code review 评论、Issue、修复补丁；
5. 真实失败案例及解决方法。

### 写 Skill 的核心原则

**第一，只写 Agent 不知道、容易做错的内容。**
不要解释“什么是 PDF”“什么是 HTTP”这种通用知识，而要写项目特定规则、工具选择、非显然边界条件。例如 PDF 技能里直接写“文本提取默认用 pdfplumber，扫描件再用 OCR”，比解释 PDF 格式更有价值。

**第二，技能粒度要适中。**
过窄会导致一个任务要加载多个技能，过宽会导致触发不精准。官方类比为“像设计函数一样设计 Skill”：一个 Skill 应该封装一个连贯的工作单元。

**第三，控制详细程度。**
过于全面的规则反而会让 Agent 难以判断当前任务该用哪部分说明。更推荐“简洁步骤 + 工作示例 + 关键边界条件”。

**第四，脆弱任务要更严格，开放任务要更灵活。**
比如数据库迁移这种 fragile task，可以要求 Agent 精确执行某个命令；代码审查这种任务，则可以给检查点而不是规定每一步。

------

## 6. 有效 Skill 指令的写法

官方推荐几种结构：

### 1）Gotchas：易错点

这是很重要的部分。把 Agent 容易犯错的非显然事实放在 `SKILL.md` 中，例如：

- 某个表使用软删除，查询必须加 `deleted_at IS NULL`；
- 不同系统中的用户 ID 字段名不一样；
- `/health` 只代表 Web 服务活着，不代表数据库连接正常。

这类内容比泛泛的“注意错误处理”更有价值。

### 2）输出模板

如果需要固定输出格式，直接给模板，而不是用自然语言描述格式。Agent 对具体结构的模仿通常比抽象描述更稳定。

### 3）Checklist

对于多步骤任务，用 checklist 防止 Agent 跳步。例如表单处理可以写成：

```markdown
- [ ] 分析表单
- [ ] 创建字段映射
- [ ] 验证映射
- [ ] 填充表单
- [ ] 验证输出
```

这种结构适合有依赖关系和验证门槛的任务。

### 4）Validation loop

让 Agent 做完一步后运行验证脚本，如果失败就根据错误修复，再重复验证，直到通过。这个思想对代码生成、表单填写、数据处理都很有价值。

### 5）Plan-validate-execute

对于批量操作或有破坏性的操作，先生成计划，再验证计划，最后执行。例如先提取 PDF 表单字段，再生成 `field_values.json`，再用脚本验证字段是否存在，最后填表。

------

## 7. 如何优化 `description`

`description` 是技能能否被正确触发的关键。官方明确说：技能只有被激活才有用，而 `description` 是 Agent 判断是否加载技能的主要机制。写得太窄会漏触发，写得太宽会误触发。

推荐写法：

- 用命令式表达，例如 “Use this skill when…”；
- 关注用户意图，而不是内部实现；
- 适当“pushy”，把适用场景说清楚；
- 保持简洁，不能超过 1024 字符。

官方建议准备约 20 条触发评估样本：

- 8–10 条 should-trigger；
- 8–10 条 should-not-trigger；
- 包含正式表达、口语表达、错别字、缩写、明确/不明确任务、短提示/长提示等。

负样本要选择“近似但不该触发”的例子，而不是明显无关的例子。例如 CSV 分析技能的负样本不应该是“今天天气怎么样”，而应该是“写一个 Python 脚本读取 CSV 并上传数据库”，因为它包含 CSV 但任务本质是 ETL，不是分析。

还建议多次运行同一个 query，因为模型行为有随机性。可以用触发率判断是否通过，例如运行 3 次，触发率超过 0.5 才算 should-trigger 通过。

------

## 8. 如何评估 Skill 输出质量

Evaluating skills 页面强调：不要只试一次觉得能用就结束，而要做结构化评测。一个测试用例包含：

1. 用户 prompt；
2. 期望输出；
3. 可选输入文件。

评估时建议每个测试都跑两遍：

- with skill：加载技能；
- without skill：不加载技能，作为 baseline。

这样可以判断 Skill 是否真正提升了质量，而不是模型本来就能完成。

评估结果要记录：

- 输出文件；
- token 消耗；
- 耗时；
- assertion 评分结果；
- 汇总 benchmark。

### Assertion 设计

好的 assertion 应该可观察、可验证，例如：

- 输出文件是合法 JSON；
- 图表有坐标轴标签；
- 报告包含至少 3 条建议。

弱 assertion 包括：

- “输出很好”；
- “必须使用某个固定短语”。

对于机械检查，比如 JSON 合法性、行数、文件尺寸，官方建议用脚本验证，而不是让 LLM 判断。

### 结果分析

不要只看总分，还要分析模式：

- with skill 和 without skill 都通过的断言，说明 Skill 没带来增量；
- 两者都失败，可能是测试太难或断言不合理；
- 只有 with skill 通过，说明 Skill 真正有价值；
- 结果波动大，说明指令可能有歧义；
- token 或时间异常，要看执行日志找瓶颈。

最后用 failed assertions、人类反馈、执行日志一起改进 `SKILL.md`，反复迭代。

------

## 9. Skill 中如何使用脚本

Using scripts 页面讲的是：Skill 可以直接要求 Agent 运行命令，也可以把可复用脚本打包到 `scripts/` 中。

### 一次性命令

如果现有工具已经能完成任务，可以直接在 `SKILL.md` 写命令，比如：

- `uvx`
- `pipx`
- `npx`
- `bunx`
- `deno run`
- `go run`

官方建议固定版本，例如 `npx eslint@9.0.0`，避免未来工具更新导致行为变化。复杂命令则应该改成 `scripts/` 中的测试脚本。

### 脚本设计原则

脚本要面向 Agent 使用，因此要避免交互式输入。Agent 运行在非交互 shell 中，不能处理密码框、确认菜单、TTY prompt。输入应该通过命令行参数、环境变量或 stdin。

还应做到：

- 支持 `--help`，说明参数和示例；
- 错误信息要具体，告诉 Agent 错在哪里、允许值是什么、下一步怎么修；
- 输出尽量结构化，例如 JSON、CSV、TSV；
- 数据输出走 stdout，日志和警告走 stderr；
- 支持幂等、dry-run、安全默认值；
- 大输出要分页或写文件，避免被工具截断。

------

## 10. Client implementors：如何让自己的 Agent 支持 Skills

面向客户端实现者的页面讲的是，如果你要开发一个 Agent 系统，如何接入 Agent Skills。核心流程是 5 步：

### Step 1：发现 Skills

本地 Agent 通常扫描两个层级：

- 项目级：当前项目里的技能；
- 用户级：用户全局技能。

推荐扫描 `.agents/skills/`，因为这是跨客户端共享的常见约定。也可以扫描客户端自己的目录，例如 `.<your-client>/skills/`。项目级技能优先级通常高于用户级技能。

同时要考虑安全：项目级技能可能来自不可信仓库，因此可以要求用户先信任该目录，再加载其中的技能，避免陌生仓库静默注入指令。

### Step 2：解析 `SKILL.md`

需要解析 YAML frontmatter，提取 `name`、`description`、可选字段和正文。官方建议实现上可以适度宽松：比如 name 不匹配目录名时可以警告但仍加载；description 缺失则应跳过，因为没有 description 就无法进行技能披露。

### Step 3：把技能目录告诉模型

构造一个 skill catalog，包含每个技能的 `name`、`description`，可选包含 `location`。这个 catalog 可以放在 system prompt 中，也可以放在专门的 skill activation tool 描述里。

### Step 4：激活 Skills

有两种常见方式：

1. **文件读取激活**：模型根据 catalog 自己读取对应 `SKILL.md`；
2. **专用工具激活**：提供 `activate_skill` 工具，输入 skill name，返回技能正文。

专用工具的好处是可以控制返回内容、加结构化标签、列出资源文件、做权限控制、记录 analytics。官方建议如果使用专用工具，应把可选 skill name 限制成 enum，避免模型编造不存在的技能名。

用户也应该能显式激活技能，例如通过 `/skill-name` 或 `$skill-name`。

### Step 5：管理技能上下文

技能一旦加载，应尽量保护它不被上下文压缩删除。因为 Skill 指令是持续有效的行为指导，如果中途丢失，Agent 的表现会隐性退化。还应该去重，避免同一个 Skill 被反复注入上下文。复杂任务也可以用 subagent 执行技能，把主对话保持简洁。

------

## 11. 总体评价：Agent Skills 的设计价值

这个项目的价值在于，它把“提示词工程”升级成了一个更工程化、可复用、可版本管理的机制。

以前我们常见做法是：

```text
把一大段 prompt 粘到 system prompt 或用户消息里。
```

Agent Skills 的做法是：

```text
把某类任务的知识、流程、脚本、模板、评估方法封装成标准目录。
Agent 只在需要时加载。
```

它更适合真实项目，因为它支持：

- 项目级知识沉淀；
- 团队规范复用；
- Agent 行为可审计；
- 技能版本管理；
- 脚本和模板打包；
- 评估驱动迭代；
- 跨 Agent 客户端复用。



