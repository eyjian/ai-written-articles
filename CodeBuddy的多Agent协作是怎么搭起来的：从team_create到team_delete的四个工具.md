# CodeBuddy 的多 Agent 协作是怎么搭起来的：从 team_create 到 team_delete 的四个工具

> 不是直接给模型发 prompt，而是用 team_create 建团队、task 派 Agent、send_message 传话、team_delete 收场——四个工具串起整条多 Agent 工作流。

很多人用 CodeBuddy 跑多 Agent 协作时，看到一堆 Agent 自动分工、互相传话、最后汇总交付，会觉得底层一定有一套很复杂的编排引擎。

其实没有。

CodeBuddy 的多 Agent 协作，从创建到销毁，核心就靠四个工具函数：`team_create` 建团队，`task` 派 Agent，`send_message` 传消息，`team_delete` 收场。这四个工具和 `read_file`、`execute_command` 属于同一类东西——都是标准的 Function Call，没有任何特殊协议。区别只是操作对象不同：`read_file` 操作的是文件系统，这四个工具操作的是 Agent 团队的生命周期和通信。

这篇不讲 Agent Team 的概念和理论，只讲这四个工具——每个怎么调、参数怎么填、什么时候用同步模式什么时候用异步模式。全文的示例都用一个"文章编写 Agent Team"来串：scout 选题、architect 设计大纲、writer 写初稿、reviewer 审稿、polisher 润色终稿——五个角色靠这四个工具协作起来。

---

## 先给判断

- **`team_create`**：创建团队容器。多个 Agent 要协作，先得有一个团队把它们装进去。
- **`task`**：派发 Agent。核心工具，既能同步派一个 Agent 干活等结果，也能异步把 Agent 扔进团队后台跑。
- **`send_message`**：Agent 之间的通信通道。发消息、广播、请求关闭，全靠它。
- **`team_delete`**：销毁团队。工作完成后清理资源，不是可选项。

四个工具的关系是线性的：**先建团队 → 再派 Agent → Agent 之间用消息协作 → 最后清理团队。**

---

## 一、team_create——先有团队才有协作

`team_create` 做的事很简单：创建一个团队容器，给后续的 Agent 提供一个协作空间。

比如要搭一个文章编写团队：

```text
team_create(
  team_name: "article-team-20260410",
  description: "文章编写协作工作流"
)
```

### 参数速查

| 参数 | 必填 | 说明 |
|------|------|------|
| `team_name` | 是 | 团队唯一名称，建议用 `lowercase-with-hyphens` 格式 |
| `description` | 否 | 团队目标描述，方便追溯 |

### 什么时候需要建团队

不是所有任务都需要建团队。判断标准很直接：

- **需要多个 Agent 异步协作、互相传消息** → 建团队
- **只需要一个 Agent 同步完成一件事** → 不用建团队，直接用 `task` 的同步模式

如果只是让一个 Agent 写一段代码或做一次分析，没必要先建一个团队再往里塞人。`team_create` 是为多 Agent 异步协作准备的，单 Agent 同步任务跳过这一步就行。

### 命名建议

`team_name` 的命名建议带上业务标识和时间戳，比如 `"article-team-20260410"` 或 `"review-task-20260410"`。原因是团队名在运行时是唯一标识，如果多个任务复用同一个名字，会出现冲突。

---

## 二、task——派发 Agent 的核心工具

`task` 是这四个工具里最重要的一个。它负责创建一个 Agent 实例并派发任务。关键在于：**它有两种模式——同步模式和异步团队模式——触发条件取决于传不传 `name` 和 `team_name` 参数。**

### 同步模式：派一个 Agent，等它做完

```text
task(
  subagent_name: "writer",
  description: "写一段引言",
  prompt: "根据以下大纲，写出文章的引言部分..."
)
```

同步模式下，`task` 会创建一个 Agent，等它执行完毕后返回结果。整个过程是阻塞的——调用方会一直等到这个 Agent 做完才继续往下走。

适合什么场景？简单任务、不需要和其他 Agent 协作、结果可以直接用。

### 异步团队模式：把 Agent 扔进团队后台跑

```text
task(
  subagent_name: "scout",
  name: "scout",
  team_name: "article-team-20260410",
  mode: "bypassPermissions",
  description: "选题调研",
  prompt: "搜索 AI 工程化领域近期热点，阅读作者已有文章避免重复选题，
           给出 3-5 个选题方案。完成后使用 send_message 通知 main。"
)
```

异步团队模式下，Agent 被加入指定团队，在后台持续运行。它不会阻塞调用方，而是通过 `send_message` 和其他团队成员通信。

**触发异步模式的关键**：只要同时提供了 `name` 和 `team_name`，就进入异步团队模式。缺了任何一个，就是同步模式。

### 两种模式对比

| 维度 | 同步模式 | 异步团队模式 |
|------|---------|-------------|
| 触发条件 | 不传 `name` 和 `team_name` | 同时传 `name` 和 `team_name` |
| 执行方式 | 阻塞等待完成 | 后台运行 |
| 通信方式 | 直接返回结果 | 通过 `send_message` |
| 适合场景 | 单任务、结果直接用 | 多 Agent 协作、需要互相传消息 |
| 是否需要 `team_create` | 不需要 | 需要先建团队 |

### 参数速查

| 参数 | 必填 | 说明 |
|------|------|------|
| `subagent_name` | 是 | Agent 类型名，须匹配已注册的 Agent（如 `scout`、`architect`、`writer`、`reviewer`、`polisher` 等） |
| `description` | 是 | 3-5 字任务简述 |
| `prompt` | 是 | 详细任务指令，告诉 Agent 具体做什么 |
| `name` | 否 | 提供后触发异步团队模式，作为该 Agent 在团队内的成员名 |
| `team_name` | 否 | 要加入的团队名，需与 `team_create` 时的名称一致 |
| `mode` | 否 | 权限模式，控制 Agent 的执行权限 |
| `max_turns` | 否 | 最大执行轮次，防止 Agent 长时间运行 |

### mode 的四种取值

| mode 值 | 含义 | 适合场景 |
|---------|------|---------|
| `"default"` | 默认权限，操作需要确认 | 不确定 Agent 会做什么时 |
| `"acceptEdits"` | 自动接受文件编辑 | Agent 需要改文件但不涉及危险操作 |
| `"bypassPermissions"` | 跳过所有权限确认 | 信任 Agent 的操作、需要全自动执行 |
| `"plan"` | 只做规划不执行 | 需要 Agent 先出方案再决定是否执行 |

`mode` 的选择本质上是一个信任度判断。对 Agent 的信任越高，给的权限越大；对任务风险的判断越谨慎，给的权限越小。在文章编写这类场景里，Agent 的操作主要是读写 Markdown 文件和搜索资料，风险可控，大多数角色可以用 `"bypassPermissions"`。

---

## 三、send_message——Agent 之间的唯一通信通道

`send_message` 在之前的文章《send\_message 到底是什么：一个标准的 Function Call，撑起了整个 Agent Team》里已经详细拆解过它的机制和设计原理。这里不重复原理，只讲在实际工作流里怎么用。

比如 scout 选题完成后，通知协调者：

```text
send_message(
  type: "message",
  recipient: "main",
  content: "选题调研完成，推荐 4 个方案，请用户确认。",
  summary: "选题待确认"
)
```

又比如 reviewer 审稿后，直接把意见发给 writer——不需要经过协调者中转：

```text
send_message(
  type: "message",
  recipient: "writer",
  content: "第三节的代码示例有误，task 的 name 参数不是必填项，请修正。",
  summary: "审稿意见"
)
```

### 参数速查

| 参数 | 必填 | 说明 |
|------|------|------|
| `type` | 是 | 消息类型 |
| `recipient` | 否 | 接收者名称。`"main"` 指协调者，也可以是其他成员名 |
| `content` | 否 | 消息正文 |
| `summary` | 否 | 5-10 个词的消息摘要 |

### 四种消息类型

| type | 用途 | 典型场景 |
|------|------|---------|
| `"message"` | 发送普通消息给指定成员 | scout 完成选题后通知 main；reviewer 直接把审稿意见发给 writer |
| `"broadcast"` | 广播给所有团队成员 | 全局状态变更通知，比如用户改了选题方向，所有人需要知道 |
| `"shutdown_request"` | 请求某个成员关闭 | 文章终稿完成后，逐个关闭 scout、architect、writer、reviewer、polisher |
| `"shutdown_response"` | 响应关闭请求 | Agent 收到关闭请求后确认可以退出 |

### recipient 决定通信方向

这一点在实际使用中经常被忽略。`recipient` 填什么，决定了消息发给谁：

- 填 `"main"` → 消息发给协调者（最常见，用于汇报和请求决策）
- 填某个成员名（如 `"writer"`、`"reviewer"`） → 消息直接发给该成员，不经过协调者

后者就是 Agent Team 区别于 Sub-Agent 流水线的关键能力——成员之间可以直接对话。比如 reviewer 发现文章第三节有事实错误，不需要先汇报给 main 再由 main 转达给 writer，直接 `recipient: "writer"` 就发过去了。architect 觉得选题方向需要微调，也可以直接联系 scout 讨论，不走协调者中转。

### shutdown 的标准流程

工作流结束时，关闭 Agent 不是直接杀进程，而是走一套协商流程：

1. 协调者发 `shutdown_request` 给目标 Agent
2. 目标 Agent 收到后，完成手头的收尾工作
3. 目标 Agent 发 `shutdown_response` 确认可以关闭
4. 协调者确认后，该 Agent 退出

这个流程的意义在于：Agent 可能正在写文件或做最后的数据持久化，直接杀掉会丢数据。`shutdown_request` 给了它一个体面收尾的机会。比如 polisher 正在做最后的格式校对，收到 `shutdown_request` 后它会先把校对结果写入文件，再响应关闭。

---

## 四、team_delete——收场不是可选项

```text
team_delete()
```

`team_delete` 是四个工具里最简单的一个，没有参数。它做的事就是销毁当前团队，释放所有资源。

### 为什么必须清理

不调 `team_delete` 会怎样？团队容器和相关资源会一直存在，占用系统资源。如果后续再创建同名团队，可能会出现冲突或状态混乱。

### 清理前的检查清单

在调 `team_delete` 之前，确认三件事：

1. **所有 Agent 都已关闭**：先用 `send_message` 发 `shutdown_request` 给每个成员，等它们都响应了再删团队
2. **产出已经保存**：Agent 的工作成果（文章终稿、大纲文件、审稿报告）已经写入了持久化存储，不会因为团队销毁而丢失
3. **没有正在进行的通信**：确认消息队列里没有未处理的消息

---

## 五、四个工具串起来——一个完整的工作流

把前面四个工具串起来，用文章编写 Agent Team 的完整流程做示例：

```text
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 第 1 步：创建团队
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
team_create(team_name: "article-team-20260410")

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 第 2 步：派发选题侦察员
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
task(
  subagent_name: "scout",
  name: "scout",
  team_name: "article-team-20260410",
  mode: "bypassPermissions",
  description: "选题调研",
  prompt: "搜索 AI 工程化近期热点，阅读作者已有文章避免重复，
           给出 3-5 个选题方案。完成后 send_message 通知 main。"
)

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 第 3 步：用户确认选题后，派发大纲架构师
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
task(
  subagent_name: "architect",
  name: "architect",
  team_name: "article-team-20260410",
  mode: "bypassPermissions",
  description: "大纲设计",
  prompt: "基于已确认的选题，设计文章结构和大纲。
           完成后 send_message 通知 main。"
)

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 第 4 步：大纲确认后，派发初稿写手
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
task(
  subagent_name: "writer",
  name: "writer",
  team_name: "article-team-20260410",
  mode: "bypassPermissions",
  description: "撰写初稿",
  prompt: "按大纲撰写 Markdown 初稿。
           完成后 send_message 通知 main。"
)

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 第 5 步：派发审稿人
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
task(
  subagent_name: "reviewer",
  name: "reviewer",
  team_name: "article-team-20260410",
  mode: "bypassPermissions",
  description: "技术审稿",
  prompt: "审查初稿的准确性和完整性。
           如有问题，直接 send_message 给 writer 要求修改；
           审稿通过后，send_message 通知 main。"
)

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 第 6 步：派发润色师
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
task(
  subagent_name: "polisher",
  name: "polisher",
  team_name: "article-team-20260410",
  mode: "bypassPermissions",
  description: "终稿润色",
  prompt: "对审稿通过的文章做最终打磨：统一术语、优化节奏、降低 AI 味。
           完成后 send_message 通知 main。"
)

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 第 7 步：终稿确认后，关闭所有成员
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
send_message(type: "shutdown_request", recipient: "scout")
send_message(type: "shutdown_request", recipient: "architect")
send_message(type: "shutdown_request", recipient: "writer")
send_message(type: "shutdown_request", recipient: "reviewer")
send_message(type: "shutdown_request", recipient: "polisher")

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 第 8 步：删除团队
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
team_delete()
```

### 流程拆解

```text
team_create     task(scout)     task(architect)    task(writer)     task(reviewer)    task(polisher)    shutdown      team_delete
    │               │                │                 │                 │                 │               │               │
    ▼               ▼                ▼                 ▼                 ▼                 ▼               ▼               ▼
 建团队 ──→  选题侦察员入队 ──→  大纲架构师入队 ──→  初稿写手入队 ──→  审稿人入队 ──→  润色师入队 ──→  逐个关闭 ──→  销毁团队
                 │                   │                  │                 │                │
                 ▼                   ▼                  ▼                 ▼                ▼
           scout 搜索选题      architect 设计大纲   writer 写初稿    reviewer 审稿    polisher 润色
           完成→通知 main      完成→通知 main      完成→通知 main   可直接→writer   完成→通知 main
```

注意第 5 步的 reviewer：它的 prompt 里写了"如有问题，直接 send\_message 给 writer 要求修改"。这就是 Agent Team 的网状通信——reviewer 不需要先把问题报告给 main，再由 main 转发给 writer，而是直接发给 writer。这条直接通信路径，是靠 `send_message` 的 `recipient` 参数实现的。

整条流程的控制权在协调者（main）手里。协调者根据 `send_message` 收到的完成通知来决定下一步派谁。但成员之间也可以绕过协调者直接对话——这就是 Agent Team 和 Sub-Agent 流水线的核心区别。

---

## 六、可用的 Agent 类型

`task` 的 `subagent_name` 参数需要匹配一个已注册的 Agent 类型。以文章编写团队为例，用到的 Agent 类型和职责如下：

| subagent_name | 角色 | 职责 |
|---------------|------|------|
| `scout` | 选题侦察员 | 搜索热点、调研资料、给出选题方案 |
| `architect` | 大纲架构师 | 设计文章结构、标注表达手法、安排收藏型结构件 |
| `writer` | 初稿写手 | 按大纲撰写 Markdown 初稿 |
| `reviewer` | 技术审稿人 | 审查准确性和完整性，可自主退回要求修改 |
| `polisher` | 终稿润色师 | 统一术语、优化节奏、降低 AI 味、格式规范化 |

除了文章编写场景，CodeBuddy 还支持研发工作流场景的 Agent 类型：

| subagent_name | 角色 | 职责 |
|---------------|------|------|
| `analyst` | 需求分析师 | 分析需求单、输出需求文档、定义验收标准 |
| `be-architect` | 后端架构师 | 设计接口、定义数据模型、技术选型 |
| `be-dev` | 后端开发工程师 | 实现 API、业务逻辑、数据库操作 |
| `fe-architect` | 前端架构师 | 设计组件结构、状态管理方案 |
| `fe-dev` | 前端开发工程师 | 实现页面、组件、交互逻辑 |
| `coder` | 通用编码工程师 | 不区分前后端的编码任务 |
| `reviewer` | 代码评审专家 | 代码审查、规范检查 |
| `tester` | 开发自测工程师 | 编写测试用例、执行自动化测试 |
| `env-init` | 环境初始化助手 | 搭建开发环境、初始化项目结构 |
| `release` | 发布准备助手 | 构建打包、发布检查 |
| `code-explorer` | 代码探索（内置） | 代码库导航和理解 |
| `code-reviewer` | 代码审查（内置） | 自动化代码审查 |

### 怎么选 Agent 类型

选择逻辑不复杂：先看任务属于哪个场景（文章编写还是研发），再按职责匹配角色。如果任务跨领域或不好归类，`coder` 是通用兜底选项。

---

## 常见误区

### 误区 1：不建团队也能跑多 Agent 异步协作

不行。异步团队模式要求 `task` 里传入 `team_name`，而 `team_name` 指向的团队必须先通过 `team_create` 创建。没有团队容器，Agent 没有地方注册，也没有消息路由——`send_message` 发出去找不到人。

同步模式确实不需要建团队，但同步模式下 Agent 之间也不能互相通信。

### 误区 2：task 只有异步模式

`task` 默认是同步模式。只有同时传了 `name` 和 `team_name` 才会进入异步团队模式。很多简单任务用同步模式就够了——派一个 writer 写一段文字，等结果回来，继续下一步。不是所有场景都需要建团队搞异步。

### 误区 3：send_message 只能发给 main

`send_message` 的 `recipient` 可以是任意团队成员的名字。填 `"main"` 是发给协调者，填 `"writer"` 就是发给 writer。文章编写团队里 reviewer 直接把审稿意见发给 writer、architect 直接找 scout 讨论选题——这些直接通信都是靠把 `recipient` 设成对应成员名实现的。

### 误区 4：team_delete 可以不调

不建议。团队资源不会自动释放。不调 `team_delete` 虽然不会立刻出错，但团队容器和相关状态会一直挂着。养成"建了就删"的习惯，和开发中的"打开了就关闭"是一个道理。

### 误区 5：写了 Agent 定义文件就能被 task 调度

这是自定义多 Agent 插件时最容易踩的坑。

很多人在插件目录下写好了 Agent 定义文件（比如 `agents/be-architect.md`），然后在工作流里直接用 `task(subagent_name: "be-architect", ...)` 去调度——结果报错：**"自定义 Agent 未注册为 subagent"**。

原因在于 `task` 的 `subagent_name` 参数不是文件路径，而是**引用一个已注册的 agent 定义名称**。CodeBuddy 的 `Task` 工具在运行时需要通过 `subagent_name` 在注册表里查找对应的 agent 定义，然后才能创建子 agent 实例。光有定义文件但没注册，框架根本不知道这个 agent 的存在。

打个比方：写了一份简历（agent 定义文件），但没有在公司的人事系统里录入（没注册到 `plugin.json`）。项目经理想把任务派给你，在系统里按名字一搜——查无此人。

具体表现：

| 操作 | 未注册 | 已注册 |
|------|--------|--------|
| `task(subagent_name: "be-architect")` | 报错：未注册为 subagent | 正常启动 |
| `read_file("...agents/be-architect.md")` | 能读到文件内容 | 同左 |

注意看第二行——文件确实存在，`read_file` 能读到。但 `task` 不是通过文件路径找 agent 的，它走的是注册表。这就是为什么"文件在、定义在、但就是调不起来"。

报错后 AI 的典型行为是**降级**：它发现通过 `Task` 工具调度 `be-architect` 失败了，就会放弃调度，转而自己来做——往往回退到内置的 `code-explorer`。如果在日志里看到"让我直接使用 code-explorer 作为辅助"这类表述，大概率就是 agent 没注册导致的降级。

解决方法：在插件的 `plugin.json` 中把自定义 agent 注册为 subagent。注册后，`task` 的 `subagent_name` 才能正确解析到对应的 agent 定义，Agent 才能被创建和调度。

---

## 结论

CodeBuddy 的多 Agent 协作，底层就是四个工具的组合：`team_create` 建容器，`task` 派角色，`send_message` 传消息，`team_delete` 收场。没有隐藏的编排引擎，没有私有协议，全部都是标准的 Function Call。

就像这篇文章本身的生产过程——scout 搜索选题、architect 设计大纲、writer 撰写初稿、reviewer 审查质量、polisher 打磨终稿——五个角色的整条协作链路，从建团队到清理现场，全部建立在这四个工具之上。理解了它们的参数和组合方式，就理解了 CodeBuddy 多 Agent 协作的工具层。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
