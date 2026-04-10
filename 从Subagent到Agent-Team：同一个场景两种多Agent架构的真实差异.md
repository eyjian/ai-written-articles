# 从 Subagent 到 Agent Team：同一个场景，两种多 Agent 架构的真实差异

> Subagent 是星形调用——主 Agent 在中心，逐个派任务、等结果、再派下一个。Agent Team 是网状协作——成员之间可以直接对话、自主退回、不经过协调者中转。同样五个角色写一篇文章，架构不同，能力边界完全不同。

上一篇《Skill 和 Subagent 到底有什么区别》讲了一件事：用 Skill 模拟多 Agent 协作是假的，改成 Subagent 之后上下文隔离和工具隔离是真的，但 Subagent 之间不能直接通信——所有协调逻辑都压在主 Agent 身上，本质上是星形拓扑。

那篇的结尾留了一个口子：

> 至于真正的网状通信和 Agent 自主协商——那需要平台层面的支持，比如 `team_create` 和 `send_message` 这套 API。

这篇就接着往下讲。同样是五个角色（scout、architect、writer、reviewer、polisher）协作写一篇文章，从 Subagent 星形架构切换到 Agent Team 网状架构之后，到底多了什么能力、改了什么机制、付出了什么代价。

---

## 先给判断

两种架构的核心区别就一句话：**Subagent 模式下所有通信必须经过主 Agent，Agent Team 模式下成员之间可以直接对话。**

这一个差异，连带改变了四件事：

1. **通信拓扑**：从星形变成网状
2. **决策权分布**：从主 Agent 集中决策变成成员可自主决策
3. **协调者角色**：从全能决策者变成传话人 + 启动器
4. **执行模式**：从同步串行变成异步并行

---

## 一、两种架构长什么样

### Subagent 模式：星形拓扑

```text
              ┌──────────────┐
              │  主 Agent     │
              │ （协调者）    │
              └──┬──┬──┬──┬──┘
                 │  │  │  │
         ┌───┐ ┌┴┐┌┴┐┌┴┐┌┴──┐
         │ S │ │A││W││R││ P │
         └───┘ └─┘└─┘└─┘└───┘

S=scout  A=architect  W=writer  R=reviewer  P=polisher
```

主 Agent 逐个调用 Subagent，每次拿到结果后自己决定下一步。所有通信路径都经过中心节点。reviewer 想让 writer 改稿？不行，reviewer 只能输出审稿结论，由主 Agent 判断后重新调用 writer。

### Agent Team 模式：网状拓扑

```text
         scout ◄──────► architect
           │  ╲          ╱  │
           │    ╲      ╱    │
           │      main      │
           │    ╱      ╲    │
           │  ╱          ╲  │
       writer ◄──────► reviewer
                  │
               polisher
```

任意成员可以直接对话。reviewer 发现初稿有问题，直接 `send_message` 给 writer 要求修改——不需要先汇报给 main，不需要 main 转发。architect 觉得选题方向需要微调，直接联系 scout 讨论。

---

## 二、创建方式的差异

### Subagent 模式：`task` 同步调用

```text
task(
  subagent_name: "scout",
  description: "选题调研",
  prompt: "搜索热点，输出选题方案到 scout-output.json。"
)
```

不建团队，不传 `name` 和 `team_name`。主 Agent 调一个、等结果、再调下一个。每个 Subagent 执行完就退出，不在后台驻留。

### Agent Team 模式：`team_create` + `task` 异步派发

```text
# 先建团队
team_create(team_name: "article-team-20260410")

# 再派发成员（异步，后台驻留）
task(
  subagent_name: "scout",
  name: "scout",
  team_name: "article-team-20260410",
  mode: "bypassPermissions",
  description: "选题调研",
  prompt: "搜索热点，给出选题方案。完成后 send_message 通知 main。"
)
```

关键区别：同时传了 `name` 和 `team_name`，触发异步团队模式。Agent 被加入团队后在后台持续运行，不会阻塞调用方，通过 `send_message` 和其他成员通信。

---

## 三、通信机制的差异

这是两种架构最本质的区别。

### Subagent 模式：中间文件 + 主 Agent 中转

```text
scout 写入 scout-output.json
        ↓
主 Agent 读取 → 判断 → 调用 architect 时注入上下文
        ↓
architect 写入 architect-output.json
        ↓
主 Agent 读取 → 判断 → 调用 writer ...
```

所有信息流都经过主 Agent。Subagent 之间看不到彼此的存在——它们只和文件系统交互，由主 Agent 负责串联。

### Agent Team 模式：`send_message` 直接通信

```text
# reviewer 直接发给 writer（不经过 main）
send_message(
  type: "message",
  recipient: "writer",
  content: "第三节代码示例有误，task 的 name 参数不是必填项。",
  summary: "审稿意见"
)

# scout 完成后通知 main
send_message(
  type: "message",
  recipient: "main",
  content: "选题调研完成，推荐 4 个方案。",
  summary: "选题待确认"
)
```

`recipient` 可以是任意成员名。填 `"main"` 就发给协调者，填 `"writer"` 就直接发给 writer。这条直接通信路径是 Agent Team 区别于 Subagent 流水线的核心能力。

### 通信对比

| 维度 | Subagent 模式 | Agent Team 模式 |
|------|--------------|----------------|
| 通信介质 | 中间文件（JSON/MD） | `send_message`（内置 API） |
| 通信方向 | 只能写文件，由主 Agent 读 | 任意成员间直接发送 |
| reviewer → writer | 不可能直接通信 | `recipient: "writer"` 直接发送 |
| 广播 | 不支持 | `type: "broadcast"` 全员广播 |
| 通信延迟 | 高（写文件 → 主 Agent 轮询 → 读文件） | 低（消息直接投递到收件箱） |

---

## 四、决策权的差异

### Subagent 模式：主 Agent 集中决策

reviewer 输出审稿报告（比如 `{"pass": false, "issues": [...]}`），它自己不能决定退回 writer。主 Agent 读到这个 JSON 后，自己判断：不通过？好，重新调用 writer 并把审稿意见注入 prompt。

**所有分支决策都在主 Agent 手里。** Subagent 只产出结论，不做流程控制。

### Agent Team 模式：成员可自主决策

reviewer 审完初稿发现问题，直接 `send_message` 给 writer 要求修改——不需要请示 main。writer 收到后自行修改，改完再通知 reviewer 复审。这个"退回 → 修改 → 复审"的循环可以在 reviewer 和 writer 之间自主进行，main 甚至可以不知道。

**决策权下放到了成员层面。** 协调者不再是全局决策者，而更像传话人和启动器——负责派发成员、转述用户需求、在关键节点暂停等用户确认。

### 决策对比

| 场景 | Subagent 模式 | Agent Team 模式 |
|------|--------------|----------------|
| reviewer 发现初稿有问题 | 输出 `{"pass": false}`，由主 Agent 决定是否重调 writer | 直接 `send_message` 给 writer 要求修改 |
| architect 觉得选题需要微调 | 输出建议，由主 Agent 决定是否重调 scout | 直接联系 scout 讨论 |
| polisher 发现事实错误 | 输出问题，由主 Agent 回退给 reviewer | 直接 `send_message` 给 reviewer 确认 |
| 多轮修改 | 每轮都经过主 Agent 中转 | reviewer 和 writer 之间自主循环 |

---

## 五、生命周期的差异

### Subagent 模式：用完即走

每个 Subagent 被 `task` 同步调用，执行完毕后返回结果就退出。不在后台驻留，没有持久状态。下次需要时重新创建一个新实例。

### Agent Team 模式：驻留 + 显式关闭

成员被 `task` 异步派发后，在后台持续运行，直到收到 `shutdown_request` 才退出。整个团队的生命周期是显式管理的：

```text
team_create → 成员入队并驻留 → 成员间协作 → shutdown_request 逐个关闭 → team_delete 销毁团队
```

驻留的好处是：reviewer 可以在 writer 修改的过程中随时介入——它不需要每次都被重新创建和初始化。坏处是：资源一直占着，必须显式清理。

---

## 六、一张表看清全部差异

| 维度 | Subagent 模式 | Agent Team 模式 |
|------|--------------|----------------|
| 拓扑 | 星形（主 Agent 在中心） | 网状（任意成员可直接通信） |
| 创建方式 | `task` 同步调用 | `team_create` + `task` 异步派发 |
| 通信机制 | 中间文件 + 主 Agent 中转 | `send_message` 直接通信 |
| 决策权 | 主 Agent 集中决策 | 成员可自主决策 |
| 生命周期 | 用完即走 | 驻留 + 显式关闭 |
| 执行模式 | 同步串行 | 异步，可并行 |
| 协调者角色 | 全能决策者 | 传话人 + 启动器 |
| 成员间直接通信 | 不支持 | 支持 |
| 成员自主退回 | 不支持 | 支持 |
| 调试难度 | 低（所有路径经过主 Agent，完全可追溯） | 高（成员间直接通信可能绕过日志） |
| 资源占用 | 低（用完即走） | 高（驻留直到显式关闭） |
| 平台要求 | 只需 `task` + `.codebuddy/agents/` | 需要 `team_create`/`send_message`/`team_delete` API |

---

## 七、同一个场景的两种实现

用文章编写的完整流程做对照：

### Subagent 模式下的流程

```text
主 Agent 调用 scout → 拿到选题方案 → 请用户确认
主 Agent 调用 architect → 拿到大纲
主 Agent 调用 writer → 拿到初稿
主 Agent 调用 reviewer → 拿到审稿报告
    ↓ 如果不通过
主 Agent 重新调用 writer（注入审稿意见）→ 拿到修改稿
主 Agent 重新调用 reviewer → 通过
主 Agent 调用 polisher → 拿到终稿
```

每一步都是"调用 → 等结果 → 判断 → 调用下一个"。reviewer 的退回要经过主 Agent，多轮修改意味着主 Agent 要多次中转。

### Agent Team 模式下的流程

```text
team_create("article-team-20260410")

task(scout, 异步入队) → scout 完成后 send_message 通知 main
    ↓ main 请用户确认
task(architect, 异步入队) → architect 完成后 send_message 通知 main
task(writer, 异步入队) → writer 完成后 send_message 通知 main
task(reviewer, 异步入队)
    ↓ reviewer 发现问题
    reviewer 直接 send_message 给 writer → writer 修改 → writer 通知 reviewer
    ↓ reviewer 复审通过 → send_message 通知 main
task(polisher, 异步入队) → polisher 完成后 send_message 通知 main

shutdown 逐个关闭 → team_delete()
```

reviewer 和 writer 之间的"退回 → 修改 → 复审"循环完全在两人之间完成，main 不需要介入。这是 Subagent 模式做不到的。

---

## 八、怎么选

不是所有场景都需要 Agent Team。判断标准：

| 需求特征 | 推荐方案 |
|---------|---------|
| 任务简单，角色之间严格串行，不需要互相交流 | Subagent |
| 需要成员间直接通信（比如 reviewer 直接退回 writer） | Agent Team |
| 需要成员自主决策，不想所有分支都经过主 Agent | Agent Team |
| 追求可控性和可追溯性，接受所有路径经过中心节点 | Subagent |
| 平台没有 `team_create`/`send_message` API | Subagent（唯一选择） |
| 平台有完整的 Agent Team 支持，且任务复杂度需要网状协作 | Agent Team |

大多数简单场景，Subagent 就够了——可控、可追溯、资源占用低。只有当"成员间直接通信"和"自主决策"成为真实需求时，才值得引入 Agent Team 的复杂度。

复杂度不是免费的。Agent Team 的网状通信带来灵活性，也带来调试难度——成员之间的直接通信可能绕过协调者的日志，出了问题不一定能快速定位是哪条消息、哪个决策节点偏了。Subagent 模式下所有路径都经过主 Agent，天然就是一条完整的决策链。

---

## 结论

Subagent 和 Agent Team 不是新旧替代关系，是两种不同拓扑的架构选择。Subagent 是星形——可控、简单、所有路径经过中心。Agent Team 是网状——灵活、成员自主、直接通信。

同样是五个角色写一篇文章，Subagent 模式下 reviewer 退回 writer 要经过主 Agent 中转两次；Agent Team 模式下 reviewer 一条 `send_message` 就发过去了，writer 改完直接回复，循环在两人之间闭合。这条直接通信路径，就是从星形到网状的那一步。

选哪种，看场景。不是越复杂越好，是刚好够用最好。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
