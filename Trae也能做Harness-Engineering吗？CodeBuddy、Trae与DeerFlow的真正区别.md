# Trae 也能做 Harness Engineering 吗？CodeBuddy、Trae 与 DeerFlow 的真正区别

> 很多人把 `CodeBuddy`、`Trae`、`DeerFlow` 放在一起聊，是因为它们都在“管 AI”。但别急着画等号：`CodeBuddy` 和 `Trae` 更像是在 IDE 里给 AI 上纪律，`DeerFlow` 则是一套独立的 Agent 运行底座。

聊到 `Harness Engineering`，很多人接着会问三件事：

- `Trae` 也能不能做这套事？
- `DeerFlow` 算不算一种 `Harness Engineering`？
- 如果都能叫 harness，那 `CodeBuddy` 跟 `DeerFlow` 到底还差什么？

这三个问题放在一起答，很容易越答越乱。原因不复杂：它们根本不在一个层级上。`CodeBuddy` 和 `Trae` 讨论的是怎样在 IDE 里把 AI 管住；`DeerFlow` 讨论的，则是怎样把这套东西做成独立系统。看起来都在“管 AI”，分量其实不一样。这篇就把这层差别拆开。

## 先把结论放前面

如果赶时间，先看这四句：

- **`Trae` 也能实践 `Harness Engineering`。** 但它主要覆盖前馈约束、规划和验收，不是完整闭环。
- **`DeerFlow` 不只是“也能做 harness”。** 它从定位上就是 `Super Agent Harness`。
- **`CodeBuddy` 和 `Trae` 更像 IDE 级 harness 实践。** 它们擅长让 AI 在本地研发流里守规矩，不是自己提供一整套 Agent 运行时。
- **`DeerFlow` 更像运行时级 harness。** 它不是陪你在编辑器里写代码，而是给 Agent 提供独立运行、调度、记忆和隔离执行的底座。

如果只记一句，那就是：**`Trae` 和 `CodeBuddy` 是在 IDE 里把 harness 练起来，`DeerFlow` 是把 harness 做成系统。**

## 第一章：先别急着比产品，先把“在比什么”说清

`Harness Engineering` 这个词，最近被讨论得很多。Martin Fowler 网站在 2026 年 4 月刊出的 [Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html) 里，把这套思路讲得很清楚。

里面有两个特别关键的词：

- **`Feedforward Guides`**：前馈式引导。也就是 AI 动手前，先把目标、边界、规则、限制说清楚。
- **`Feedback Sensors`**：反馈式传感。也就是 AI 动手后，不听它口头说完成了，而是用报告、测试、检查、评审把它拉回现实。

再说白一点：

> **`Harness Engineering` 不是让模型突然变聪明，而是让系统更不容易失控。**

所以这里真正要比的，不是“谁也有个规则文件”，而是下面三层：

| 层级 | 它在解决什么 | 典型代表 |
|------|-------------|---------|
| **方法论层** | 怎样把 AI 放进约束、反馈、控制回路里 | `Harness Engineering` |
| **IDE 实践层** | 怎样在本地研发流里把这套纪律落地 | `CodeBuddy`、`Trae` |
| **运行时层** | 怎样给 Agent 提供可长期运行的基础设施 | `DeerFlow` |

把这三层拆开后，很多混淆就没那么难解释了。三者都在“管 AI”，但管的是不同位置的事。

## 第二章：为什么说 `Trae` 也能实践 `Harness Engineering`

结论是：**能，但范围别说大了。** `Trae` 能实践 `Harness Engineering`，但它实践的主要是 harness 的一个轻量子集，不是完整闭环。

它最强的两块其实很清楚。

一块是 `Rules`。

`Trae` 支持项目级规则目录 `.trae/rules`，规则文件用 Markdown 编写，可以设置始终生效、按文件匹配生效、智能生效，甚至支持手动 `#Rule` 触发。这个能力，本质上就是把“前馈约束”显式写进项目里。

另一块是计划能力。

`Trae` 的 `/plan` 会先生成一份计划文档，把目标、范围、步骤先对齐；`/spec` 更重，直接生成 `spec.md`、`task.md`、`checklist.md` 三份文档，把需求、任务拆解和验收清单一起写出来。

这套东西当然像 harness。因为它干的就是三件事：

- 先把边界写死
- 再把任务拆开
- 最后用清单或验收项做反馈

如果把 `CodeBuddy` 和 `Trae` 摆在一起看，会更清楚：

| Harness 四层 | `CodeBuddy` 常见落点 | `Trae` 常见落点 | 本质作用 |
|------|--------------------|----------------|---------|
| **前馈约束** | `.codebuddy/rules` | `.trae/rules` | 先把范围、禁区、规范写死 |
| **执行编排** | `Plan Mode` | `/plan` | 先规划，再执行 |
| **强规格对齐** | Plan 文档审阅、任务确认 | `/spec` 产出 `spec.md`、`task.md`、`checklist.md` | 减少 AI 自由发挥 |
| **反馈验证** | 执行前审阅、执行后检查、计划归档 | `checklist.md` + 文档化验收 | 不听口头总结，只看产物 |

但也正因为如此，`CodeBuddy` 和 `Trae` 覆盖得更多还是 harness 的“上半身”：前馈约束、执行编排、规格对齐、验收反馈。再往下，一套更完整的 harness 还得处理执行隔离、长期记忆、多 Agent 调度、可脱离 IDE 运行这些事。

| harness 完整闭环的关键层 | `CodeBuddy` / `Trae` | `DeerFlow` |
|------|-------------------|-----------|
| **前馈约束** | ✅ 强项 | ✅ |
| **执行编排** | ✅ 强项 | ✅ |
| **反馈验证** | ⚠️ 能做，但更依赖人在环上复核 | ✅ 可以做成系统内闭环 |
| **执行隔离** | ❌ 基本没有独立沙箱 | ✅ `sandbox` |
| **长期记忆** | ❌ 不主打长期任务记忆 | ✅ `memory` |
| **多 Agent 调度与自治运行** | ⚠️ 能做协作，但更依附 IDE、没有独立运行时 | ✅ `subagents` + 独立运行时 |

所以如果问题是：**“`使用CodeBuddy实践Harness-Engineering.md` 里那套思路，能不能搬到 `Trae` 上？”**答案是：**可以，而且做法基本能平移。** 你只需要把：

- `.codebuddy/rules` 换成 `.trae/rules`
- `Plan Mode` 换成 `/plan`
- 更复杂的需求，直接上 `/spec`

方法论没有变。变的是界面、命令名和交互细节。真正要补的一句是：**`CodeBuddy` 和 `Trae` 能把多角色协作做起来，但这层能力更像 IDE 内编排；`DeerFlow` 则把它做成了运行时级的自治基础设施。**

## 第三章：为什么说 `DeerFlow` 不是“也能做”，而是“它就是”

到 `DeerFlow` 这里，层级就变了。它不是在一个编辑器里给 AI 加一点规则、加一点计划。它自己就是那套运行底座。

`DeerFlow` 在 GitHub README 里的自我介绍其实很直白：`DeerFlow` 全称是 `Deep Exploration and Efficient Research Flow`，它把自己定义成 [open-source super agent harness](https://github.com/bytedance/deer-flow)。README 里那句核心描述，基本已经把它讲透了：它把 `sub-agents`、`memory` 和 `sandboxes` 组织在一起，再配合可扩展的 `skills`，让 Agent 能处理从研究、编码到内容生成的一整类长任务。照这个方向看，`DeerFlow 2.0` 已经不只是一个“深度研究框架”，而是在往一套完整的 Agent 运行系统走。

这里还有个容易让人疑惑的点：**既然它现在强调的是 harness，为什么名字还叫 `DeerFlow`？**

其实就是项目演进留下来的名字。它早期确实更像一个 deep research workflow framework，名字里的 `flow` 对应的是任务流、步骤编排和研究流程。到了 `2.0`，项目已经不只是在讲 workflow，而是在往 super agent harness 演进。名字留了下来，定位变了。这在开源项目里很常见。

也正因为这样，我会单独强调：**`workflow` 不是 `harness` 的同义词。**

| 概念 | 它主要回答什么 |
|------|----------------|
| **`workflow`** | 步骤怎么串起来跑、状态怎么流转、任务怎么编排 |
| **`harness`** | Agent 在哪跑、怎么约束、怎么验收、怎么隔离、怎么记忆、怎么调度 |

说得再直一点，`workflow` 只是 harness 里的一层，通常对应执行编排；而 harness 还得再往前管约束，往后管反馈，往下管隔离、记忆和运行边界。少了这些，它就还是流程，不是完整 harness。

所以 `DeerFlow` 真正在回答的，不是“我怎么在 IDE 里少跑偏”，而是“我怎么让一个 Agent 真正在独立环境里跑上几十分钟、几个小时，还别把宿主机一起搞乱”。把这层意思摊开，大概就是下面这张表：

| `DeerFlow` 能力 | 对 harness 意味着什么 |
|------|-------------------|
| **`sandbox`** | 执行隔离。Agent 能动手，但不会直接把宿主环境搞坏 |
| **`subagents`** | 原生任务拆分和并行协作，不是单线程对话助手 |
| **`memory`** | 长期状态积累，不只是一轮会话里的上下文 |
| **`skills` / `tools`** | 能力扩展是系统内建能力，不靠临时 prompt 拼装 |
| **`message gateway`** | 任务入口不局限于 IDE，可以来自 IM、Web、接口 |
| **上下文管理** | 长任务里可以卸载、总结、续跑，不容易被窗口长度卡死 |

所以我会把它概括成一句话：

> **`DeerFlow` 不是“把 harness 用起来”，而是“把 harness 做出来”。**

这和 `CodeBuddy`、`Trae` 那种“在编辑器里用规则和计划把 AI 管住”的思路，已经不是同一种产品重心了。

这也不是一句虚话，连启动门槛都直接写在工程现实里。要真把 `DeerFlow` 跑起来，通常至少得先准备这几样东西：

- **一套能承载服务的运行环境。** 官方支持 `Linux`、`macOS`、`Windows`，但如果是在 `Windows` 本地开发，官方明确要求用 `Git Bash`，不是随便开个 `cmd` 或 `PowerShell` 就行。
- **完整的前后端依赖。** 本地开发不是装一个插件就结束了，而是至少要有 `Node.js 22+`、Python 环境，以及 `pnpm`、`uv`、`nginx` 这套依赖链，跑起来前还要先过 `make check`。
- **至少一个可用的大模型 API Key。** `DeerFlow` 不是离线玩具，先得跑 `make config` 生成 `.env` 和 `config.yaml`，再把模型密钥真正配进去；如果你还想让它联网搜索，还得额外准备 `Tavily` 一类搜索服务的 Key。
- **一套明确的启动方式。** 官方推荐 `Docker`，首次要先 `make docker-init`，然后 `make docker-start`；如果走本地开发，则是 `make config`、`make check`、`make install`、`make dev` 这一整套流程。

这几条放在一起，就更容易理解它和 `CodeBuddy`、`Trae` 的差别了：**`DeerFlow` 不需要先依附某个 IDE，也不要求你必须有 `CodeBuddy` 这样的工具；它更像一套自己就能独立启动的服务。CLI 主要是拿来安装、检查和启动，真正的使用入口是它自己的 Web UI。**

所以说它“门槛更高”，不是在夸它高级，而是在陈述一个很朴素的事实：**面对的已经不是一个编辑器里的 AI 助手，而是一套需要部署、配置、启动、维护的 Agent 运行系统。**

## 第四章：`CodeBuddy` 和 `DeerFlow` 到底差在哪

这部分最容易被一句偷懒的话带过去：“哦，一个是 IDE，一个是 Agent 平台。”这话没错，但太轻了。真正的差别不止是形态，而是 **harness 被放在了哪里**。

| 维度 | `CodeBuddy` | `DeerFlow` |
|------|-------------|-----------|
| **本质定位** | AI 编码 IDE / IDE 内协作助手 | 独立的 `Super Agent Harness` |
| **harness 的位置** | 在编辑器工作流里，用规则、计划、工具去约束 AI | 在系统运行时里，把记忆、隔离、调度、工具都做成基础设施 |
| **主要任务** | 本地研发、跨文件修改、代码实现、调试协作 | 长周期、多阶段、跨能力的复合任务 |
| **运行环境** | 依附 IDE 和当前项目目录 | 独立服务与运行环境，可接多入口 |
| **多 Agent 能力** | 已支持协作、成员通信和并行，但仍依附 IDE | 原生 `subagents` 调度和并行执行 |
| **记忆能力** | 更偏用户偏好、项目上下文 | 更偏长期任务状态和用户画像积累 |
| **执行隔离** | 主要在当前工作区内操作 | 有 `sandbox` 级别的隔离执行环境 |
| **门槛** | 低，开 IDE 就能开始 | 高，需要部署、配置、维护 |
| **最适合的第一步** | 给日常编码加纪律 | 给复杂 Agent 系统搭底座 |

如果喜欢用类比，我还是沿用前面那句更好懂的说法：

- **`CodeBuddy` / `Trae`** 像你骑在马上，手里多了一套更靠谱的缰绳、路线卡和检查表。
- **`DeerFlow`** 像你不只是拿着缰绳，而是连马厩、调度系统、补给站和多匹马协同机制都一起搭好了。

所以两者不是谁替代谁。而是一个偏**“人在环上的 IDE 纪律化协作”**，另一个偏**“Agent 可独立运行的基础设施工程”**。

## 第五章：把 `Trae` 也放进来，三者到底该怎么选

如果把 `Trae` 也一起拉进来，最实用的看法不是“谁更强”，而是“你现在处在哪个阶段”。

| 场景 | 更适合先用谁 | 原因 |
|------|-------------|------|
| **日常写业务代码、改多文件逻辑** | `CodeBuddy` 或 `Trae` | 都属于 IDE 级 harness，门槛低，反馈快 |
| **需求还没讲清，但怕 AI 直接抢跑** | `CodeBuddy Plan Mode` / `Trae /plan` | 先规划，再执行，先把方向对齐 |
| **新模块从 0 到 1，想把需求、任务、验收都写重一点** | `Trae /spec` | `spec.md + task.md + checklist.md` 这套规格流更重 |
| **想在本地研发中持续沉淀规则、技能、工具协同** | `CodeBuddy` | `Rules`、`Skills`、`MCP`、计划归档这套组合更像持续工程化 |
| **任务会持续很久，还要多 Agent 并行、沙箱执行、长期记忆** | `DeerFlow` | 这已经不是 IDE 内小闭环了，而是运行时系统问题 |
| **刚开始学 harness，怕一上来太重** | `CodeBuddy` 或 `Trae` | 先在 IDE 里把纪律练熟，成本最低 |

我的判断很直接：

**多数开发者的第一站，依然应该是 `CodeBuddy` 或 `Trae` 这种 IDE 级 harness。**

原因不复杂，先要学会的是“怎么约束 AI”，不是“怎么先搭一套 Agent 平台”。

很多人一上来就奔着多 Agent、沙箱、长期记忆去，结果最后最基础的边界、验收、复检都没立住。

这就有点像还没把单人驾驶练顺，就先买了一整套车队调度系统。

容易热闹。也容易翻车。

## 第六章：最容易混淆的三个误区

这部分值得单独拎出来。

| 误区 | 为什么不对 | 更准确的理解 |
|------|------------|--------------|
| **“`Trae` 也有规则和计划，所以它和 `DeerFlow` 是一类。”** | 规则和计划只覆盖 harness 的前半段，不代表你已经有了隔离、记忆和运行时底座 | `Trae` 更像 IDE 级实践，`DeerFlow` 是运行时级系统 |
| **“`DeerFlow` 更强，所以 `CodeBuddy` 就没意义了。”** | 两者解决的问题不一样，一个偏本地研发协作，一个偏独立 Agent 运行 | 该在 IDE 里做的，还是 IDE 最顺手 |
| **“我只要把 prompt 写细一点，就算做了 harness。”** | 只有 prompt，没有反馈、验收、回滚和控制，不算完整闭环 | 真正的 harness 至少要有前馈约束和反馈传感 |

大家最容易混淆的点，其实都在“把局部能力误认成整个系统”。而 `Harness Engineering` 最忌讳的，恰恰就是这个。它不是一个魔法词。它是把约束、执行、反馈、控制拼成闭环的工程做法。

## 第七章：如果你真要落地，最实用的路线是什么

如果目标不是争论概念，而是真的开始用，建议按这个顺序走：

1. **先在 `CodeBuddy` 或 `Trae` 里练小闭环**
2. **把规则文件、计划、验收清单这套纪律练顺**
3. **确认你真的遇到了长周期、多 Agent、独立执行的瓶颈**
4. **再考虑要不要上 `DeerFlow` 这种运行时级 harness**

这个顺序很重要。前一阶段练的是“怎么管 AI”，后一阶段才是“怎么给 Agent 搭系统”。别反过来。反过来很容易花很多力气去搭底座，结果最基本的行为约束还是空的。

## 结论

回到开头那三个问题，答案可以收得更简单一点。

`Trae` 能实践 `Harness Engineering`，但主要还是 IDE 里的轻量闭环；`DeerFlow` 当然也算，而且它不是“顺手做了 harness”，而是从定位上就在做一套运行时级的 harness。

所以别把它们理解成谁替代谁。`CodeBuddy` / `Trae` 更适合把日常 AI 编码协作管住，`DeerFlow` 更适合承载长周期、多 Agent、可独立运行的任务。

如果你今天就想开始，先在 `CodeBuddy` 或 `Trae` 里把规则、计划、验收、复检这些基本功练顺。真遇到长周期、多 Agent、独立执行的瓶颈，再考虑 `DeerFlow`。

## 参考资料

- [CodeBuddy 官方文档：概览](https://www.codebuddy.ai/docs/zh/ide/User-guide/Overview)
- [CodeBuddy 官方文档：Plan Mode](https://www.codebuddy.ai/docs/zh/ide/Features/Plan-Mode)
- [Trae 规则文档](https://www.w3cschool.cn/traedocs/rules-for-ai.html)
- [Trae 社区：Spec 模式与 Plan 模式](https://forum.trae.cn/t/topic/236)
- [DeerFlow GitHub 仓库](https://github.com/bytedance/deer-flow)
- [Martin Fowler：Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html)

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
