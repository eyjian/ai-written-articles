# Skill 和 Subagent 到底有什么区别：CodeBuddy 的两种扩展机制与一次真实改造

> Skill 是流程指令包，Subagent 是独立 Agent 实例。搞混了就会写出"看起来像多 Agent 协作，其实是一个 Agent 在演多个角色"的架构——五个角色共享同一个上下文窗口、同一套工具、同一个 LLM 实例，和真正的多 Agent 没有关系。

有人用 CodeBuddy 搭了一套"文章编写 Agent Team"：scout 选题、architect 设计大纲、writer 写初稿、reviewer 审稿、polisher 润色终稿。五个角色各有分工，prompt 里还写了 `send_message` 互相传话，看起来就是一个标准的多 Agent 协作系统。

跑了几轮发现不对。所有角色共享同一个上下文窗口，reviewer 审稿时能看到 writer 的全部中间思考过程（上下文互相污染），每个角色用的工具完全一样没法隔离，而 prompt 里写的 `send_message` 和 `team_create`——CodeBuddy 平台当时根本就没提供这些 API。

问题出在哪？**这五个"角色"全部是 Skill，不是 Subagent。** Skill 加载后只是在主 Agent 的上下文里注入了一段流程指令，本质上还是同一个 Agent 在"演"不同角色。角色切换是演出来的，上下文隔离是假的，通信机制是虚构的。

这篇文章从这次真实的改造过程出发，讲清楚 CodeBuddy 的两种扩展机制——Skill 和 Subagent——各自能做什么、不能做什么，以及从前者迁移到后者的完整路径。

---

## 先给判断

赶时间就记这三句：

- **Skill**（`.codebuddy/skills/`）：模块化的流程指令包。通过 `SKILL.md` 定义触发条件和工作指令，由主 Agent 按需加载到自己的上下文里执行。**加载后不产生新的 Agent 实例**，所有操作仍然在主 Agent 的上下文窗口里完成。
- **Subagent**（`.codebuddy/agents/`）：真正独立的 AI 助手。每个 `agentic` 模式的 Subagent 有自己的上下文窗口、自己的工具列表，执行完毕后只返回结果，**不污染主会话**。
- **核心区别**：Skill 是给主 Agent 加流程，Subagent 是给主 Agent 加帮手。前者是"教主 Agent 怎么做"，后者是"派一个独立的 Agent 去做"。

---

## 一、Skill 和 Subagent 各管什么

### Skill：流程指令包

Skill 存放在 `.codebuddy/skills/` 目录下，每个 Skill 有一个 `SKILL.md` 文件定义触发条件和工作指令。用户通过斜杠命令（如 `/article-team`）或自动匹配触发后，主 Agent 把 Skill 的指令加载到自己的上下文里，然后按照指令执行。

关键特征：

- 加载后**不创建新的 Agent 实例**——主 Agent 自己执行
- 所有操作**共享主 Agent 的上下文窗口**——Skill 能看到之前的对话历史，之前的对话也能看到 Skill 的输出
- **工具不隔离**——Skill 用的是主 Agent 的全部工具，不能单独配置
- 适合的场景：流程编排、写作规范、代码检查规则、领域知识注入——任何"教主 Agent 怎么做"的场景

### Subagent：独立 Agent 实例

Subagent 存放在 `.codebuddy/agents/` 目录下，每个 `.md` 文件定义一个 Agent 的角色、工具和行为模式。分两种模式：

- **`agentic` 模式**：主 Agent 自动调用，Subagent 在独立上下文中执行，完成后返回结果。调用方（主 Agent）不需要知道 Subagent 内部的推理过程。
- **`manual` 模式**：用户手动选择，Subagent 替代主 Agent 成为当前对话的 Agent。

关键特征：

- 每个 `agentic` Subagent 有**独立的上下文窗口**——不污染主会话
- 可以配置**独立的工具列表**——比如 scout 只给 WebSearch 和 ReadFile，writer 只给 ReadFile 和 WriteFile
- 可以指定**不同的模型**——比如审稿用推理能力更强的模型，润色用写作风格更好的模型
- 适合的场景：需要隔离上下文、隔离工具权限、并行执行、或者需要独立决策的任务

### 一张表看清区别

| 维度 | Skill | Subagent（agentic 模式） |
|------|-------|------------------------|
| 存放位置 | `.codebuddy/skills/` | `.codebuddy/agents/` |
| 运行方式 | 主 Agent 加载指令后自己执行 | 创建独立 Agent 实例执行 |
| 上下文 | 共享主 Agent 上下文 | 独立上下文窗口 |
| 工具隔离 | 不隔离，用主 Agent 全部工具 | 可配置独立工具列表 |
| 模型 | 和主 Agent 相同 | 可指定不同模型 |
| 执行后 | 输出留在主 Agent 上下文里 | 只返回结果，不污染主会话 |
| 适合场景 | 流程编排、规范注入、知识补充 | 需要隔离、独立决策、工具限制的任务 |

---

## 二、核心矛盾——Skill 模式下的"假多 Agent"

理解了上面的区别，就能看出用 Skill 搭"多 Agent 协作"时的根本问题。

### 同一个窗口里的角色扮演

五个 Skill（scout、architect、writer、reviewer、polisher）加载后，都在主 Agent 的同一个上下文窗口里运行。reviewer "审稿"时，上下文里已经堆满了 scout 的选题过程、architect 的大纲设计、writer 的初稿写作——全部可见。

这带来两个问题：

1. **上下文污染**：reviewer 的判断会受到 writer 的中间推理过程影响。真正的独立审稿应该只看到初稿成品，不应该看到"writer 为什么这么写"的思考过程。
2. **上下文膨胀**：五个角色的全部输入输出累积在同一个窗口里，很容易撞到上下文长度限制。越到后面的角色（reviewer、polisher），可用的上下文空间越少。

### 工具无法隔离

Skill 模式下，所有角色共享主 Agent 的全部工具。scout 可以写文件，writer 可以执行命令，polisher 可以搜索网页——没有任何隔离。

在真正的多 Agent 架构里，工具隔离是基本要求。scout 只需要搜索和阅读能力，不应该有写文件的权限；writer 只需要读写文件，不应该有执行系统命令的权限。权限越小，出错的面越窄。

### 虚构的通信机制

最容易踩的坑：在 Skill 的 prompt 里写了 `send_message`、`team_create` 这些调用，以为 Agent 真的在用这些 API 通信。实际上 CodeBuddy 平台并没有提供这些多 Agent 通信 API——prompt 里的"通信"只是主 Agent 在自说自话。

scout "通知" main、reviewer "退回" writer——这些看起来像 Agent 间通信的行为，本质上是同一个 LLM 实例在不同 prompt 段落间切换角色。消息没有真的发出去，也没有真的被接收。

---

## 三、改造方案——从 Skill 角色扮演到真 Subagent

明确了问题，改造方向就清楚了：**把 Skill 角色升级为真正的 Subagent，用独立上下文替换角色扮演，用结构化输出替换虚构通信。**

### 方案一：纯 Subagent 模式

最直接的改法——把五个角色全部从 `.codebuddy/skills/` 迁移到 `.codebuddy/agents/`，变成真正的 Subagent。

每个 Subagent 的 `.md` 文件结构：

```markdown
---
name: topic-scout
description: 文章选题侦察员。当需要选题、标题优化或定位判断时自动调用。
tools: WebSearch, WebFetch, ReadFile
agentMode: agentic
enabled: true
---

# 选题侦察员（Topic Scout）

（保留原有 Skill 中的核心 prompt 内容，但去掉所有 send_message、
team_create 相关指令，改为：完成后输出结构化 JSON 结果供主 Agent 消费）
```

关键改动：

- 每个 Subagent 独立运行，有自己的上下文窗口和工具权限
- 去掉所有虚构的通信机制（`send_message`、`team_create`、`team_delete`）
- 改为输出**结构化结果**（如 JSON），由主 Agent 消费并决策下一步
- 共享的领域画像配置仍然保留，每个 Subagent 启动时从文件读取

### 方案二：编排 Skill + Subagent 组合模式

更灵活的改法——保留一个 Skill 做编排入口，但它调度的角色变成真正的 Subagent。

架构变成：

- `article-team` 仍然作为 Skill 存在于 `.codebuddy/skills/`，负责触发入口和编排逻辑
- 五个角色迁移到 `.codebuddy/agents/`，变成 `agentic` 模式的 Subagent
- Skill 里的编排逻辑改为：按需调用 Subagent → 解析结构化输出 → 决策下一步 → 调用下一个 Subagent

编排 Skill 的核心变化：

```text
改造前：
  team_create → task(scout) → send_message 通知 → task(architect) → ...

改造后：
  调用 scout Subagent → 拿到结构化输出 →
  请用户确认 →
  调用 architect Subagent → 拿到结构化输出 →
  调用 writer Subagent → ...
```

不再假装有 `team_create` 和 `send_message`。协调逻辑回归本质——**主 Agent 串行调用 Subagent，每次拿到结果后自己决定下一步**。

### 改造前后对比

| 维度 | 改造前（Skill 角色扮演） | 改造后（真 Subagent） |
|------|------------------------|---------------------|
| Agent 运行时 | 同一 LLM 实例角色切换 | 每个 Subagent 有独立上下文窗口 |
| 工具隔离 | 所有角色共享同一套工具 | 每个 Subagent 可配置独立工具集 |
| 上下文污染 | 所有角色共享同一个会话上下文 | Subagent 执行完毕后只返回结果 |
| Agent 间通信 | 虚构的 `send_message` | 通过中间文件 + 主 Agent 编排 |
| 自主决策 | prompt 里写"自主决定"但实际是同一 LLM | 主 Agent 解析结构化输出后真正做分支决策 |
| 模型可差异化 | 不可以 | 每个 Subagent 可指定不同模型 |

### 改造后的目录结构

```text
.codebuddy/
├── skills/
│   └── article-team/                     ← 编排 Skill（保留）
│       ├── SKILL.md                      ← 触发入口
│       ├── commands/article-team.md      ← 编排逻辑（改造）
│       └── shared-writing-resources/     ← 领域画像配置（不变）
│
└── agents/                               ← 真正的 Subagents（新增）
    ├── topic-scout.md                    ← agentic 模式
    ├── outline-architect.md              ← agentic 模式
    ├── draft-writer.md                   ← agentic 模式
    ├── tech-reviewer.md                  ← agentic 模式
    └── final-polisher.md                 ← agentic 模式
```

---

## 四、Subagent 之间怎么"通信"

改造后面临一个现实问题：CodeBuddy 的 Subagent 只能被主 Agent 调用，**Subagent 之间不能直接通信**。这意味着 reviewer 不能直接把审稿意见发给 writer——必须经过主 Agent 中转。

替代方案是**中间文件 + 结构化输出**：

```text
.codebuddy/article-subagent/
├── profile-context.json         ← 画像解析结果
├── scout-output.json            ← 选题方案
├── architect-output.json        ← 大纲
├── writer-output.md             ← 初稿
├── reviewer-output.json         ← 审稿报告
└── polisher-output.md           ← 终稿
```

每个 Subagent 的 prompt 中指定：

- **输入**：读取上游 Subagent 的输出文件
- **输出**：写入自己的结构化输出文件

主 Agent 负责的事变得很明确：解析输出 → 做分支决策（比如 reviewer 给了"不通过"，就重新调用 writer）→ 把上下文注入下一个 Subagent 的调用 prompt。

这套方案和真正的 Agent Team 通信（`send_message`）相比，有一个本质区别：**所有协调逻辑都集中在主 Agent 身上**。reviewer 不能自主决定退回 writer，它只能输出审稿结论，由主 Agent 判断是否需要退回。自主决策权从 Subagent 收回到了主 Agent。

这不一定是坏事。对于大多数场景来说，协调逻辑集中在一处反而更容易调试和控制。但如果需要 Agent 之间真正的自主协作——比如 reviewer 直接和 writer 对话讨论修改方案——这在当前的 Subagent 架构下做不到。

---

## 五、改造后仍然做不到的事

即使完成了从 Skill 到 Subagent 的改造，以下能力在当前架构下仍然受限：

| 局限 | 说明 |
|------|------|
| Subagent 之间直接通信 | 必须经过主 Agent 中转，不能点对点通信 |
| 真正的并行执行 | `agentic` Subagent 是串行调用的，不能同时启动两个 |
| Subagent 执行中途干预 | 调用后只能等完成或中断，不能在执行过程中注入新指令 |
| Agent 自主退回和协商 | Subagent 不能主动调用其他 Subagent，退回逻辑由主 Agent 控制 |

这些局限和真正的 Agent Team 架构（比如用 `team_create` + `send_message` 搭建的网状协作）之间存在差距。差距的根源在于：**Subagent 架构是星形拓扑（主 Agent 在中心），Agent Team 架构是网状拓扑（任意成员可直接通信）。**

但换一个角度看，这些"局限"也是"约束"。串行调用意味着执行顺序完全可控，主 Agent 中转意味着所有决策路径可追溯，不支持中途干预意味着每个 Subagent 的输入输出是确定性的。在工程实践中，可控往往比灵活更重要。

---

## 怎么判断该用 Skill 还是 Subagent

不需要每次都选 Subagent。判断标准很直接：

| 需求特征 | 推荐方案 |
|---------|---------|
| 只需要给主 Agent 注入流程规范或领域知识 | Skill |
| 需要独立上下文，避免污染主会话 | Subagent |
| 需要工具隔离，不同角色用不同工具 | Subagent |
| 需要不同角色用不同模型 | Subagent |
| 任务简单，一个 Agent 按流程做就行 | Skill |
| 任务复杂，需要多个独立角色分工 | Subagent |
| 需要 Agent 之间直接通信和自主协商 | 当前都做不到，但 Subagent + 主 Agent 编排可以近似实现 |

大多数场景下，Skill 就够了。只有当上下文污染、工具隔离或模型差异化成为真实痛点时，才需要升级到 Subagent。

---

## 结论

Skill 和 Subagent 不是同一层东西。Skill 是给主 Agent 加流程指令，Subagent 是给主 Agent 加独立帮手。用 Skill 模拟多 Agent 协作，看起来像那么回事，跑起来全是角色扮演——共享上下文、共享工具、虚构通信。

改造路径也不复杂：把角色定义从 `.codebuddy/skills/` 搬到 `.codebuddy/agents/`，去掉虚构的通信 API，改成结构化输出 + 主 Agent 编排。改完之后不是完美的多 Agent 协作，但至少上下文隔离是真的、工具隔离是真的、每个角色的执行边界是清楚的。

至于真正的网状通信和 Agent 自主协商——那需要平台层面的支持，比如 `team_create` 和 `send_message` 这套 API。在那之前，Subagent + 主 Agent 编排是当前架构下最接近"真多 Agent"的方案。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
