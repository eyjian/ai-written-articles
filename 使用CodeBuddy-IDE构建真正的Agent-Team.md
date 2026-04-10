# 使用 CodeBuddy IDE 构建真正的 Agent Team：从 Skill 隐式编排到 team_create 工具链

> 上一篇用 Skill 编排搭了一个 Agent Team——5 个角色、网状通信、自主决策。CodeBuddy 的 Team 模式确实提供了独立上下文、点对点通信、并行执行这些基础设施，但那篇文章的编排方式是**隐式的**：团队的组建、成员的派发、协作的启动，全部藏在 Skill 的 prompt 设计和文件组织里，看不到 team_create，看不到 task，也看不到显式的 send_message 调用。这篇讲怎么用 team_create/task/send_message/team_delete 四个显式工具函数，把同样 5 个角色从"Skill 隐式编排"改造成"工具链显式编排"——让团队的创建、通信、生命周期全部可见可控。

---

## 第一章：旧文章到底搭了个什么？——Skill 编排的隐式 Agent Team

先回顾上一篇《使用 CodeBuddy IDE 构建 Agent Team》做了什么：一个 Skill 入口（`SKILL.md`）、一个编排命令（`commands/article-team.md`）、5 个成员 prompt 文件（`agents/` 目录下的 scout、architect、writer、reviewer、polisher）。目录结构清晰，角色分工明确，prompt 里写好了通信规则——reviewer 可以 `send_message` 给 writer 退回修改，architect 可以直接找 scout 讨论选题。

旧文章确认了 CodeBuddy 的 Team 模式已经具备三项关键基础设施：

| 能力 | 说明 |
|------|------|
| **独立上下文** | Team 模式下每个 member 有自己的对话历史 |
| **点对点通信** | `send_message` 的 `recipient` 可以是任意 member |
| **并行执行** | Team 模式支持并行启动 |

这三项都是真实存在的平台能力，不是旧文章编造的。旧文章的 Agent Team 也确实建立在这些能力之上。

### 那问题出在哪？

问题不在"能力有没有"，而在**"能力怎么被调用的"**。

旧文章的编排方式是：写一组 Skill 文件（SKILL.md + 编排命令 + 成员 prompt），用户触发 `/article-team` 后，由 Skill 的加载机制把这些 prompt 送进运行时。团队的组建、成员的派发、协作的启动——这些事情**都发生在 Skill 加载和 prompt 注入的过程中**，在运行时看不到一行 `team_create` 调用，看不到一行 `task` 派发，`send_message` 也是写在 prompt 的通信规则描述里。

打个比方：就像一个工厂，旧方案是**把组织架构图贴在墙上**（prompt 文件），工人看了架构图自行组织协作；新方案是**用 ERP 系统下工单、派任务、追踪进度**（team_create/task/send_message/team_delete），每一步操作都有系统记录。

工人（Agent）可能是同一批人，工厂（CodeBuddy 的 Team 模式基础设施）也是同一个——但**管理方式完全不同**。

### 三个实际局限

旧方案的隐式编排带来三个具体问题：

**1. 团队组建不透明**

团队是怎么建起来的？成员是什么时候加入的？哪些成员在运行？——这些问题在 Skill 编排方式下，只能通过阅读 prompt 文件来推断，运行时没有显式的创建动作可追溯。

**2. 通信过程不可控**

prompt 里写了"reviewer 可以 send_message 给 writer"，但这条通信规则是**描述性的**——它描述了 Agent 应该做什么，而不是给了 Agent 一个它能感知到的工具调用指令。Agent 是否真正调用了 `send_message`、消息是否投递成功、对方是否收到——在隐式编排下不容易确认。

**3. 生命周期无管理**

Skill 执行完了就结束了。成员是什么时候退出的？有没有正在进行的任务被中断？资源有没有释放？——没有显式的 `shutdown_request` 和 `team_delete`，这些问题缺少可控的管理手段。

### 对照图：隐式编排 vs 显式编排

```
Skill 隐式编排：                           工具链显式编排：

  .codebuddy/skills/article-team/          team_create("article-team-20260410")
  ├── SKILL.md                                 │
  ├── commands/article-team.md                 ├─ task(scout)     ← 显式派发
  └── agents/                                  ├─ task(architect) ← 显式派发
      ├── scout.md                             ├─ task(writer)    ← 显式派发
      ├── architect.md                         ├─ task(reviewer)  ← 显式派发
      ├── writer.md                            ├─ task(polisher)  ← 显式派发
      ├── reviewer.md                          │
      └── polisher.md                          ├─ send_message    ← 显式通信
                                               ├─ shutdown_request← 显式关闭
  触发：/article-team                          └─ team_delete()   ← 显式销毁
  团队组建：隐式（Skill 加载时发生）
  通信方式：prompt 描述通信规则                运行时每一步操作都有对应的工具调用
  生命周期：隐式（Skill 结束时结束）
```

**一句话总结：旧文章用 Skill 的文件组织和 prompt 设计来隐式编排 Agent Team，新方案用 team_create/task/send_message/team_delete 四个工具来显式编排——底层的 Team 模式能力是同一套，区别在编排层。**

这不是说隐式编排没有价值——对很多场景它够用了（第六章会讲什么时候够用）。但当需要精确控制团队的创建时机、成员的派发顺序、通信的可追溯性和资源的显式释放时，就需要换成工具链显式编排。

---

## 第二章：显式编排需要什么？——从隐式到显式的四个跨越

从 Skill 的隐式编排升级到工具链的显式编排，核心是让四个原本"藏在 Skill 加载过程中"的能力变得**可见、可控、可追溯**：

### 四个显式化要求

简单说，就是把四件原来"自动发生"的事变成"亲手调用"：

1. **建团队**——用 `team_create` 替代 Skill 加载时的隐式组建
2. **派成员**——用 `task` 替代读取 `agents/` 目录的隐式加载
3. **发消息**——用 `send_message` 工具调用替代 prompt 里的通信规则描述
4. **关团队**——用 `shutdown_request` + `team_delete` 替代 Skill 结束时的隐式清理

每一个的具体做法和差异，第三章逐个展开。

### 对照表：Skill 隐式编排 vs 工具链显式编排

| 维度 | Skill 隐式编排 | 工具链显式编排 |
|------|--------------|--------------|
| **团队创建** | Skill 加载时隐式发生 | `team_create` 显式调用 |
| **成员派发** | 读取 agents/ 目录的 prompt 文件 | `task` 逐个显式派发独立实例 |
| **通信方式** | prompt 里的通信规则描述 | `send_message` 工具的显式调用 |
| **执行控制** | 由 Skill 编排命令统一驱动 | 协调者按需派发，可控制顺序和时机 |
| **生命周期** | Skill 结束时隐式结束 | `shutdown_request` + `team_delete` 显式管理 |
| **可追溯性** | 低——需要读 prompt 文件推断 | 高——每一步操作都有对应的工具调用 |

显式编排的代价是**搭建成本更高**——需要手写 team_create、task 调用逻辑、处理 send_message 的路由和时序。隐式编排的优势正是**简单**——写好 prompt 文件、触发 Skill 就能跑。选哪种取决于对可控性的需求（第六章详谈）。

---

## 第三章：四个工具串起整条链路——team_create / task / send_message / team_delete

上一篇文章《CodeBuddy 的多 Agent 协作是怎么搭起来的》已经详细拆解了四个工具的参数和用法。这里不重复参数细节，重点讲**每个工具在"从隐式编排到显式编排"这个改造过程中解决了什么问题**。

### 3.1 team_create：建团队容器——Skill 编排版没有这一步

Skill 编排版的"团队"是什么？是 `.codebuddy/skills/article-team/` 目录下的一组文件。触发 Skill 后，团队的组建隐式发生在 Skill 加载过程中——没有显式的"建团队"动作。

```text
team_create(team_name: "article-team-20260410", description: "文章编写协作工作流")
```

`team_create` 做的事是在运行时创建一个真正的团队容器。这个容器提供：
- **成员注册表**：后续通过 `task` 加入的 Agent 在这里注册，`send_message` 的路由依赖它
- **消息路由**：Agent A 发给 Agent B 的消息，靠团队容器找到 B 的收件箱并投递
- **生命周期管理**：团队里有谁、谁还活着、谁已关闭，容器统一管理

没有这一步，后面的 `task` 没有地方注册 Agent，`send_message` 发出去找不到人。

### 3.2 task（异步团队模式）：派发独立 Agent 实例——关键差异是 name + team_name 触发异步

Skill 编排版的"派发 Agent"是怎么发生的？Skill 加载时自动把成员 prompt 注入运行时，成员的创建和启动由 Skill 机制内部完成——在运行时看不到 `task` 调用。

显式编排版的派发：

```text
task(
  subagent_name: "scout",
  name: "scout",
  team_name: "article-team-20260410",
  mode: "bypassPermissions",
  description: "选题调研",
  prompt: "搜索 AI 工程化近期热点，阅读作者已有文章避免重复选题，
           给出 3-5 个选题方案。完成后使用 send_message 通知 main。"
)
```

关键差异：**同时传入 `name` 和 `team_name` 触发异步团队模式**。`task` 会创建一个**独立的 Agent 实例**——有自己的上下文窗口、自己的执行线程。创建后它在后台运行，不阻塞调用方。

> **subagent_name 从哪里来？** `task` 的 `subagent_name` 引用的是已注册的 Agent 类型名称。文章编写团队的 `scout`/`architect`/`writer`/`reviewer`/`polisher` 不是内置 Agent——如果用的是自定义多 Agent 插件，需要先在 `plugin.json` 中注册为 subagent，`task` 才能找到它们。详见第五章踩坑记录中的"Agent 注册失败"。如果不想自定义注册，可以统一用内置的 `code-explorer` 作为 `subagent_name`，把角色职责全部放在 `prompt` 参数里。

对比一下：
- **Skill 隐式编排**：成员的创建和派发由 Skill 加载机制自动完成，在运行时看不到派发过程
- **工具链显式编排**：`task` 显式创建独立实例 → 实例在后台运行 → 通过 `send_message` 通信 → 派发时机和顺序完全由控制

### 3.3 send_message：显式的消息投递

这是隐式编排和显式编排差异最明显的地方。

Skill 隐式编排版里，prompt 写了"reviewer 可以 send_message 给 writer"。这是一条**通信规则描述**——它告诉 Agent 应该做什么，但 Agent 是否真正调用了 `send_message` 工具、消息投递过程是否可追踪，在隐式编排下不容易确认。

显式编排版里：

```text
# reviewer 调用
send_message(
  type: "message",
  recipient: "writer",
  content: "第三节的代码示例有误，task 的 name 参数不是必填项，请修正。",
  summary: "审稿意见"
)
```

这条消息会：
1. 被 reviewer 的 Agent 实例生成为一个 Function Call
2. 运行时解析这个 Function Call，找到团队容器里注册的 writer
3. 把消息投递到 writer 的收件箱
4. writer 在下一个执行轮次读到这条消息并处理

**消息经过了真正的投递过程，reviewer 和 writer 是两个不同的 Agent 实例。** 这就是显式编排的优势——每条消息的发送者、接收者、内容都是可追踪的工具调用，reviewer 退回修改后，writer 在自己的上下文里处理审稿意见，不会被其他角色的历史内容干扰。

### 3.4 team_delete：显式销毁

Skill 隐式编排版没有显式的"销毁"步骤——Skill 执行流程结束时，团队的清理隐式发生。

显式编排版需要明确清理：

```text
# 先逐个关闭成员
send_message(type: "shutdown_request", recipient: "scout")
send_message(type: "shutdown_request", recipient: "architect")
send_message(type: "shutdown_request", recipient: "writer")
send_message(type: "shutdown_request", recipient: "reviewer")
send_message(type: "shutdown_request", recipient: "polisher")

# 等所有成员确认关闭后
team_delete()
```

为什么需要？因为每个 Agent 实例占用独立的资源（上下文窗口、执行线程）。不清理就一直挂着。而且 `shutdown_request` 给了 Agent 一个收尾机会——polisher 可能正在写最后一段，直接杀掉会丢数据。

### 3.5 完整流程对照

```text
Skill 隐式编排版：                         工具链显式编排版：

触发 /article-team                         team_create("article-team-20260410")
  │                                            │
  ├─ Skill 加载，隐式组建团队                    ├─ task(scout) → 选题完成后
  ├─ 成员 prompt 自动注入运行时                  ├─ task(architect) → 大纲确认后
  ├─ 通信规则写在 prompt 描述中                  ├─ task(writer) → 初稿完成后
  ├─ 协作过程由 Skill 机制内部驱动               ├─ task(reviewer) → 审稿通过后
  │                                            ├─ task(polisher) → 按需派发
  │                                            │
  └─ Skill 结束，隐式清理                       ├─ Agent 之间通过 send_message 显式通信
                                               ├─ reviewer → writer 消息真正可追踪
                                               ├─ shutdown_request → 逐个关闭
                                               └─ team_delete() → 显式销毁

团队组建/通信/清理：隐式              团队组建/通信/清理：显式、每一步有对应工具调用
```

---

## 第四章：每个角色的 prompt 怎么改——从"描述通信规则"到"使用真正的工具"

这是全文最关键的落地章节。Skill 隐式编排版和工具链显式编排版的 prompt 差异，不在于角色定义或职责描述（那些基本一样），而在于**通信和协作的部分怎么写**。

### 4.1 总体原则

**Skill 隐式编排版**的 prompt 写的是"通信规则描述"——描述 Agent 应该和谁沟通、沟通什么，但通信的实际发生由 Skill 机制内部驱动。

**工具链显式编排版**的 prompt 写的是"工具使用指令"——告诉一个独立运行的 Agent 实例"有 send_message 这个工具，用它给 writer 发消息"。这些指令会被 Agent 显式执行为 Function Call。

核心区别就一句话：**隐式编排版写的是"应该怎么做"的描述，显式编排版写的是"用什么工具做"的指令。**

### 4.2 逐角色对比

#### 协调者（编排命令）

**Skill 隐式编排版（`commands/article-team.md`）**：

```markdown
你是文章编写团队的协调者。
按顺序读取 agents/ 目录下的 prompt，组织各角色完成任务。
通信规则在 prompt 中描述，由 Skill 机制驱动执行。
```

**工具链显式编排版**：

```markdown
你是文章编写团队的协调者。

## 你的工具
- team_create：创建团队容器
- task：派发独立 Agent 实例（注意 name + team_name 触发异步模式）
- send_message：和团队成员通信
- team_delete：工作完成后销毁团队

## 工作流程
1. 用 team_create 建团队
2. 按需用 task 派发成员（不一次性全派，根据流程节点决定）
3. 接收成员的 send_message 通知，在关键节点请用户确认
4. 全部完成后，先 shutdown_request 所有成员，再 team_delete

## 你不做的事
- 不中转成员之间的消息（让他们直接 send_message）
- 不决定文章退不退回修改（reviewer 自己决定）
```

关键差异：隐式编排版协调者通过 Skill 的 prompt 组织来驱动流程，显式编排版协调者用 `task` 派发独立实例、用 `send_message` 通信——每一步操作都有对应的工具调用。

#### 技术审稿人（reviewer）

**Skill 隐式编排版**：

```markdown
## 审查结果处理
- 有严重问题：send_message 给 writer，附上具体问题和修改建议
- 无严重问题：审稿通过，send_message 给 polisher 启动润色

## 通信规则
- send_message 给 writer：退回修改
- send_message 给 polisher：审稿通过
- send_message 给 main：需要用户介入时
```

**工具链显式编排版**：

```markdown
## 审查结果处理——用 send_message 工具执行
- 有严重问题：
  调用 send_message(type: "message", recipient: "writer", content: "具体问题和修改建议", summary: "审稿退回")
  不用经过 main，不用等任何人批准
- 无严重问题：
  调用 send_message(type: "message", recipient: "polisher", content: "审稿通过，请开始润色", summary: "审稿通过")

## 通信工具使用
- send_message(recipient: "writer")：退回修改
- send_message(recipient: "polisher")：审稿通过
- send_message(recipient: "main")：超 3 轮需用户介入
- send_message(recipient: "scout"/"architect")：讨论选题或结构问题

## 重要
- 你是一个独立运行的 Agent 实例，writer 是另一个独立实例
- 你发给 writer 的消息会真正进入 writer 的收件箱
- 退回决策完全自主，审查-修改循环最多 3 轮
```

关键差异：显式编排版明确告诉 Agent"你有 send_message 工具""writer 是另一个独立实例""消息会真正投递"。这些不是废话——Agent 需要知道自己的操作会被执行为真正的 Function Call，否则它可能只是"描述"一下发消息的意图，而不是真正调用工具。

#### 初稿写手（writer）

**Skill 隐式编排版**：

```markdown
## 自主判断
- 大纲某部分不合理，直接 send_message 给 architect 讨论
- reviewer 要求修改时，自行判断修改方案，不用等 main 指示
- 修改完成后直接通知 reviewer 重新审查
```

**工具链显式编排版**：

```markdown
## 协作工具使用
- 发现大纲问题：调用 send_message(recipient: "architect", content: "具体问题")
- 收到 reviewer 的修改意见：直接修改文章，完成后调用 send_message(recipient: "reviewer", content: "已修改，请重新审查")
- 初稿完成：调用 send_message(recipient: "main", content: "初稿已完成", summary: "初稿完成")

## 你会收到的消息
- 来自 main 的任务指令（包含大纲内容）
- 来自 reviewer 的审稿意见（需要处理并回复）
- 来自 architect 的大纲调整通知（需要据此修改）
```

关键差异：显式编排版增加了"你会收到的消息"这一段。因为 writer 是独立实例，它需要知道自己的收件箱里可能出现哪些来源的消息，以及如何响应。隐式编排版不需要这个——通信的接收和处理由 Skill 机制内部管理。

#### 选题侦察员（scout）和大纲架构师（architect）

改法和上面类似，核心变化是：
1. 把"send_message 给 XXX"从描述性文字改成工具调用指令
2. 增加"你会收到的消息"部分
3. 明确告知"你是独立实例，对方也是独立实例"

#### 终稿润色师（polisher）

```markdown
## 协作工具使用
- 发现技术错误：调用 send_message(recipient: "reviewer", content: "发现技术问题，详情如下...")
- 标题问题：调用 send_message(recipient: "scout", content: "标题建议调整...")
- 润色完成：调用 send_message(recipient: "main", content: "终稿已完成", summary: "终稿待确认")

## 你会收到的消息
- 来自 reviewer 的审稿通过通知（表示可以开始润色）
- 来自 main 的任务指令
```

### 4.3 可复用的通信规则 prompt 模板

不管哪个角色，通信相关的 prompt 都可以套用这个模板：

```markdown
## 你的通信工具
你有 send_message 工具，可以直接和团队成员通信。

## 发送消息
- 发给 [角色名]：send_message(type: "message", recipient: "[角色名]", content: "消息内容", summary: "5-10字摘要")
- 发给协调者：send_message(type: "message", recipient: "main", content: "消息内容", summary: "摘要")

## 你会收到的消息
- 来自 [角色A] 的 [什么类型的消息]：[如何响应]
- 来自 [角色B] 的 [什么类型的消息]：[如何响应]
- 来自 main 的任务指令：[如何响应]

## 通信原则
- 直接发给目标角色，不经过 main 中转
- 需要用户确认时才发给 main
- 每条消息附上 summary，方便协调者追踪
```

把角色名和消息类型填进去，就能用。这个模板的关键是三段式：**能发什么 + 会收到什么 + 发送原则**。

> **注意**：`send_message` 的四个参数中，只有 `type` 是必填的（参见 schema 定义 `required: ["type"]`）。`recipient`、`content`、`summary` 都是可选参数，视场景使用。但在实际团队通信中，`recipient` 和 `content` 几乎必带——不指定收件人和内容的消息没有意义。

---

## 第五章：踩坑记录

从 Skill 隐式编排改造到工具链显式编排的过程中，以下问题大概率会遇到。

| 踩坑 | 现象 | 原因 | 解法 |
|------|------|------|------|
| **Agent 注册失败** | `task` 调用报错"未注册为 subagent" | 只写了 Agent 定义文件（如 `agents/reviewer.md`），没有在 `plugin.json` 中注册 | 在插件的 `plugin.json` 中把自定义 Agent 注册为 subagent，`task` 的 `subagent_name` 引用的是注册表里的名称，不是文件路径 |
| **消息发出去没人接** | reviewer 发了 send_message 给 writer，但 writer 没有任何反应 | writer 还没被 `task` 派发到团队里，团队注册表里找不到 writer | 确保 `task` 派发的顺序合理——收消息的 Agent 必须先于发消息的 Agent 被加入团队，或者在发送前确认目标 Agent 已注册 |
| **无限退回循环** | reviewer 和 writer 之间不断来回退回修改，停不下来 | prompt 里没有写退出条件，两个 Agent 各自按规则执行 | 在 reviewer 的 prompt 里设置轮次上限（如"审查-修改循环最多 3 轮，超过后汇总发给 main 让用户介入"），同时设置 `task` 的 `max_turns` 参数作为硬兜底 |
| **上下文膨胀** | Agent 运行一段时间后开始"忘记"早期指令，输出质量下降 | 虽然每个 Agent 有独立上下文，但长时间运行后单个 Agent 的上下文也会很长——特别是 writer 和 reviewer 之间多次来回后 | 在 send_message 里传结构化摘要而非全文（比如 reviewer 只发具体问题列表，不把整篇文章的审查详情都塞进 content）；关键文件产出写入磁盘而非留在上下文里 |
| **调试困难** | 多个 Agent 同时运行，出了问题不知道是谁的锅 | Agent 之间的消息和执行过程分散在各自独立的上下文中，没有统一的日志视图 | 在每个 Agent 的 prompt 里要求关键动作前发一条摘要给 main（如"即将退回 writer 修改"），让协调者的上下文成为事实上的日志聚合点 |
| **team_delete 后资源残留** | 删了团队但有些 Agent 似乎还在运行 | 没有先 shutdown_request 就直接 team_delete，Agent 实例还在后台 | 严格执行"先逐个 shutdown_request，等 shutdown_response 确认，再 team_delete"的流程 |

这些坑不是理论推演——每一个都是实际改造过程中真实遇到的。其中"无限退回循环"和"Agent 注册失败"出现频率最高。

---

## 第六章：适用边界——什么时候 Skill 隐式编排够用，什么时候必须上工具链显式编排

### Skill 隐式编排够用的场景

1. **角色之间没有复杂交互**：流程是线性的，A 做完给 B，B 做完给 C，不存在退回、并行、讨论。比如简单的"选题→大纲→写稿→润色"流水线，每一步输出直接作为下一步输入。

2. **总上下文长度可控**：所有角色的输入输出加起来不超过模型的有效注意力范围。一篇 3000 字的短文，5 个角色的总产出可能只占 10K token，模型处理起来毫无压力。

3. **不需要并行**：任务本身是串行的，没有"architect 设计大纲的同时 scout 继续搜索"这种需求。

4. **快速原型**：先用 Skill 隐式编排跑通流程逻辑，验证角色分工和通信规则是否合理，再决定要不要升级到工具链显式编排。

### 必须上工具链显式编排的场景

1. **需要精确控制成员派发时机**：比如 reviewer 必须在 writer 完成后才启动，而不是一开始就全部加载进来。

2. **需要可追踪的通信**：reviewer 退回 writer 修改后，你需要确认消息是否投递成功、writer 是否收到并处理了。

3. **需要并行加速**：长文写作时，不同章节可以由不同 writer 实例并行撰写；或者 reviewer 审前几章的同时 writer 继续写后几章。

4. **角色之间需要多轮异步交互**：architect 和 scout 就选题方向来回讨论三轮，这种多轮交互需要每条消息都是可追踪的工具调用，而不是隐藏在 prompt 驱动的内部过程中。

5. **需要显式的资源管理**：团队什么时候建、成员什么时候关、资源什么时候释放——这些需要明确的 `team_create` 和 `team_delete` 来保障。

### 渐进式迁移路径

不需要一步到位。推荐的升级路线：

```
第 1 阶段：Skill 隐式编排
  └─ 一个 Skill 入口 + 多个角色 prompt
  └─ 适合：验证流程、短文写作、线性任务

        ↓ 当需要上下文隔离但流程仍是线性时

第 2 阶段：Subagent 星形
  └─ 用 task 同步模式逐个调用 Agent
  └─ 每个 Agent 有独立上下文，但不能互相通信
  └─ 适合：需要上下文隔离、但流程仍是线性的场景

        ↓ 当需要 Agent 之间直接对话和并行执行时

第 3 阶段：Agent Team 网状
  └─ team_create + task 异步模式 + send_message 网状通信
  └─ 适合：复杂协作、多轮退回、并行执行
```

### 选型决策表

| 判断维度 | Skill 隐式编排 | Subagent 星形 | Agent Team 网状 |
|---------|-----------|--------------|----------------|
| 角色数量 | 3-5 个 | 3-7 个 | 5+ 个 |
| 角色间是否需要直接通信 | 否 | 否 | 是 |
| 是否需要退回修改 | 简单退回 | 简单退回 | 多轮退回 |
| 是否需要并行 | 否 | 否 | 是 |
| 上下文总长度 | < 30K token | 不限（独立上下文） | 不限（独立上下文） |
| 调试难度 | 低 | 中 | 高 |
| 搭建成本 | 低（只写 prompt） | 中（需要编排逻辑） | 高（需要团队管理和通信设计） |
| 推荐场景 | 快速原型、短文、线性流程 | 需要上下文隔离的线性流程 | 复杂协作、长文写作、多角色讨论 |

---

## 结论

**Skill 隐式编排和工具链显式编排的分界线，在于团队的创建、通信和生命周期是否由你显式控制。** 两者都建立在 CodeBuddy Team 模式的同一套基础设施之上——独立上下文、点对点通信、并行执行——区别只在编排层：前者由 Skill 机制自动驱动，后者由 `team_create`/`task`/`send_message`/`team_delete` 四个工具显式调用。

这四个工具各自解决了一个跨越：`team_create` 让团队组建可见，`task` 的异步模式让成员派发可控，`send_message` 让通信可追踪，`team_delete` 让资源清理可确认。但它们的代价也很明确——搭建成本高、调试链路长。

所以选型的判断标准不是"哪个更先进"，而是**你对可控性的需求到了哪一级**。线性流程、短文写作、快速验证——Skill 隐式编排完全够用。等遇到多轮退回、并行写作、通信追溯这些硬需求时，再沿着"Skill → Subagent 星形 → Agent Team 网状"的路径渐进迁移。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
