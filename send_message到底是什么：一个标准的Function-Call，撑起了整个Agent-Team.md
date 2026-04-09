# send_message 到底是什么：一个标准的 Function Call，撑起了整个 Agent Team

> Agent Team 里的 Agent 到底怎么互相说话的？很多人用了半天多 Agent 协作，一直没想过这个问题。答案没什么神秘的：靠的就是一个叫 `send_message` 的标准 Function Call 工具。它不是内置魔法，和 `read_file`、`bash` 属于同一类东西——只不过别的工具是让 Agent 操作文件和系统，`send_message` 是让 Agent 开口说话。没有它，Agent Team 里的成员就是一群各自干活、互相不说话的独立进程。

有人问过我一个问题："Agent Team 里的 reviewer 把审稿结果发给 writer，这中间到底走的是什么机制？是某种内部消息队列？还是共享内存？还是什么私有协议？"

都不是。就是 reviewer 调了一次 `send_message`，把 `recipient` 设成 `"writer"`，`content` 里塞了审稿意见。和你让 Agent 调 `read_file` 读个文件，在机制上没有任何区别——都是 Function Call。

这个事实听起来平淡，但它的工程含义一点都不平淡：**Agent Team 的一切协作——谁跟谁说话、谁能指挥谁、信息怎么流转——全部建立在这一个工具函数之上。**

---

## 先给判断

赶时间就记这三句：

- **`send_message` 是一个标准的 Function Call 工具**，和 `read_file`、`write_file`、`bash` 属于同一类。
- **它的作用是让 Agent 主动发消息**——发给用户、发给协调者、发给其他 Agent。简单说：**`send_message` 就是 Agent 的嘴。** `read_file` 是眼睛，`bash` 是手，`send_message` 让它能开口说话。
- **Agent Team 的通信拓扑（星型还是网状）取决于 `send_message` 被允许发给谁**——这是 Sub-Agent 和 Agent Team 的关键分水岭之一。

---

## 一、send_message 就是一个 Function Call

### 标准格式长什么样

在 CodeBuddy 的 Agent 体系里，`send_message` 的定义和其他工具一模一样：

```json
{
  "name": "send_message",
  "parameters": {
    "type": "object",
    "properties": {
      "type": {
        "type": "string",
        "description": "消息类型：message / broadcast / shutdown_request 等"
      },
      "recipient": {
        "type": "string",
        "description": "接收者的 Agent 名称，如 main、writer、reviewer"
      },
      "content": {
        "type": "string",
        "description": "消息内容"
      },
      "summary": {
        "type": "string",
        "description": "消息摘要（5-10 个词）"
      }
    },
    "required": ["type"]
  }
}
```

Agent 调用时输出的也是标准的 Function Call 格式：

```json
{
  "name": "send_message",
  "parameters": {
    "type": "message",
    "recipient": "main",
    "content": "阶段二完成，产出：架构文档，状态：成功",
    "summary": "架构阶段完成通知"
  }
}
```

没有任何特殊协议。LLM 生成这段 JSON，运行时解析它，把消息投递到对应 Agent 的收件箱。就这样。

### 和其他工具放在一起看

| 工具 | 做什么 | 本质 |
|------|--------|------|
| `read_file` | 读文件 | Function Call |
| `write_to_file` | 写文件 | Function Call |
| `execute_command` | 执行 shell 命令 | Function Call |
| `search_content` | 搜索代码内容 | Function Call |
| **`send_message`** | **给用户或其他 Agent 发消息** | **Function Call** |

**全部都是 Function Call，没有例外。** `send_message` 唯一的特殊之处在于它的"操作对象"不是文件系统或终端，而是**另一个 Agent 或用户**。

---

## 二、它到底在做什么事

`send_message` 承担了 Agent 所有的"开口说话"场景。拆开看，主要是三类。

### 1. Agent → 用户：提问、确认、汇报

这是最基础的用法。Agent 需要主动和用户交互时——问一个问题、确认一个操作、汇报进度——都通过 `send_message` 实现。

```text
场景：Agent 在写代码前需要确认需求边界
调用：send_message(type="message", recipient="main", content="请确认：密码强度是否需要考虑特殊字符？")
```

没有 `send_message`，Agent 就没有办法主动发起对话。它只能被动等你给它指令，不能反过来问你问题。

### 2. Agent → Agent：Team 模式下的成员间通信

这是 Agent Team 的核心场景。多个 Agent 之间需要传递信息——审稿人把意见发给写手，协调者把任务分配给开发者——全靠 `send_message`。

```text
场景：reviewer 审完稿子，直接把意见发给 writer
调用：send_message(type="message", recipient="writer", content="第三节的案例数据有误，建议修改...", summary="审稿意见")
```

这条消息不经过协调者中转，reviewer 直接发给 writer。这种**直接通信**正是 Agent Team 区别于 Sub-Agent 流水线的关键能力。

### 3. 控制类消息：关闭、广播

除了普通消息，`send_message` 还承载了一些控制类操作：

| type | 作用 |
|------|------|
| `message` | 普通消息，发给指定 Agent |
| `broadcast` | 广播消息，发给所有 Agent |
| `shutdown_request` | 请求某个 Agent 关闭 |
| `shutdown_response` | 回应关闭请求 |

这些控制类消息也是 Function Call，只是 `type` 字段不同。

### 类比：send_message 就是 Agent 的嘴

如果把 Agent 比作一个人：

- `read_file` 是它的**眼睛**——看文件
- `execute_command` 是它的**手**——操作系统
- `search_content` 是它的**搜索能力**——找信息
- **`send_message` 是它的嘴**——说话、提问、汇报、和同事沟通

一个没有 `send_message` 的 Agent，就是一个**能看能做但不会说话**的工人。它可以闷头干活，但它没法告诉你它在干什么、遇到了什么问题、需要你确认什么。更没法和其他 Agent 协作。

---

## 三、为什么 Agent Team 离不开它

### 没有 send_message 的 Agent Team 是什么样

假设你拆了 5 个 Agent：scout、architect、writer、reviewer、polisher。但它们都没有 `send_message` 工具。

会发生什么？

- scout 选完题，没法告诉 architect"选题定了，你可以开始设计大纲了"
- architect 写完大纲，没法把大纲发给 writer
- reviewer 发现问题，没法把审稿意见发给任何人
- polisher 润色完，没法通知用户"终稿好了"

**5 个 Agent 各自埋头干活，但谁都不知道别人做到哪了、产出了什么。** 这不是 Agent Team，这是 5 个互不认识的独立进程。

`send_message` 就是把这 5 个独立进程串成一个团队的那根线。

### recipient 决定了拓扑

这是 `send_message` 最被低估的一个参数。

在 prompt 里写了"所有消息只发给 main"，那 `recipient` 永远是 `"main"`——这就是**星型拓扑**，也就是 Sub-Agent 流水线的典型形态：

```text
Sub-Agent 拓扑：
  scout → main → architect → main → writer → main → reviewer → main
  所有人只和 main 说话
```

在 prompt 里写了"可以直接发给其他成员"，那 `recipient` 可以是 `"writer"`、`"reviewer"` 等任意 Agent——这就是**网状拓扑**，也就是 Agent Team 的典型形态：

```text
Agent Team 拓扑：
  reviewer → writer（直接发审稿意见）
  architect → writer（直接发大纲）
  polisher → reviewer（发现技术问题退回）
```

**同一个 `send_message` 工具，prompt 里的通信规则不同，出来的拓扑就完全不同。** 这就是为什么有些项目用了 Team 模式的 API，做出来的却是 Sub-Agent 流水线——不是工具不支持，是 prompt 把 `recipient` 限死成只能发给 `"main"` 了。

### 案例：同一个工具，两种通信模式

| 维度 | Sub-Agent 流水线 | Agent Team |
|------|-----------------|------------|
| `recipient` 的值 | 只有 `"main"` | `"main"`、`"writer"`、`"reviewer"` 等 |
| 通信方向 | 单向上报 | 多向协作 |
| 谁决定下一步 | 协调者 | 发消息者和接收者都可能影响流程 |
| 消息内容 | "阶段 X 完成，状态：成功" | "第三节案例数据有误，建议改成..." |

**区别不在于 `send_message` 本身，而在于它被允许发给谁、发什么。**

---

## 四、跨框架对比：各家叫法不同，本质一样

`send_message` 不是 CodeBuddy 独有的。几乎所有 Agent 框架都有同功能的工具，只是命名不同。以下信息基于各框架公开文档整理，具体命名和参数以你使用的实际版本为准：

| 框架 / 平台 | 工具名 | 主要用途 |
|-------------|--------|---------|
| **CodeBuddy** | `send_message` | Agent ↔ Agent、Agent → 用户 |
| **OpenAI Assistants API** | `send_message`（内置） | Agent → 用户 |
| **Claude Code** | `send_message` / `interact` | Agent → 用户、Agent 间通信 |
| **LangChain / LangGraph** | 自定义 `send_message` 或内置通信节点 | Agent ↔ Agent |
| **AutoGen** | `send` / `initiate_chat` | Agent ↔ Agent |
| **CrewAI** | 继承 LangChain 通信 + `ChatMessageTool` | Agent 间通信 |
| **Semantic Kernel** | `SendMessageSkill` / `ChatMessage` | 跨语言通信 |

命名变体还有很多：`reply_user`、`ask_user`、`notify_user`、`chat_message`、`respond`、`feedback`。名字不同，做的事完全一样：**让 Agent 开口说话**。

这也意味着：你在 CodeBuddy 里学会的 `send_message` 用法和设计思路，搬到其他框架里也成立。核心逻辑是通用的。

---

## 五、常见误解

### 误解 1：send_message 是某种内置魔法

不是。它就是一个注册在工具列表里的 Function Call。LLM 根据上下文决定"我现在需要发一条消息"，然后生成一段调用 JSON，运行时解析并执行。和调用 `read_file` 读文件的机制完全一样。

### 误解 2：有了 send_message 就是 Agent Team

不是。`send_message` 只是一个通信工具。就像有了电话不等于有了团队——关键不是你有没有电话，而是**你被允许打给谁、打了之后谁做决定**。

如果所有 Agent 的 prompt 都写着"只能发消息给 main"，那即使 `send_message` 支持发给任何人，实际通信也是星型的。工具有能力，但 prompt 没放权。

### 误解 3：send_message 只能发纯文本

不一定。在 CodeBuddy 里，`content` 字段是字符串，但你可以在里面放 JSON、Markdown、结构化数据。很多 Agent Team 的实践里，Agent 之间传递的不是一句话，而是一份结构化的任务报告、一份审稿清单、一份测试结果。

```json
{
  "type": "message",
  "recipient": "writer",
  "content": "## 审稿结果\n\n- AC-1 ✅\n- AC-2 ❌ 缺少边界校验\n- AC-3 ✅\n\n请修复 AC-2 后重新提交。",
  "summary": "审稿结果：1项未通过"
}
```

`send_message` 的灵活性取决于你往 `content` 里放什么，而不是工具本身的限制。

---

## 结论

`send_message` 是 Agent 世界里最容易被忽视的一个工具。它没有 `read_file` 那么直观，没有 `bash` 那么酷，但**它是 Agent 之间唯一的通信通道**。

没有它，Agent 能看、能做、能搜索，但不会说话。有了它，Agent 才能提问、汇报、协商、退回、确认——才能形成真正的协作。

而 Agent Team 和 Sub-Agent 的分水岭，往往也不在于用了什么 API 或什么框架，而在于 `send_message` 的 `recipient` 被允许填谁。填 `"main"` 是星型流水线，填任意成员是网状协作。**一个参数的权限，决定了整个团队的拓扑。**

下次看到一个项目自称 Agent Team，不用看别的，先看它的 `send_message` 发给了谁。答案就在那个 `recipient` 里。

## 延伸阅读

- Sub-Agent 和 Agent Team 的完整判定标准 → 《Sub-Agent 与 Agent Team 的本质区别》
- Agent Team 的角色怎么拆 → 《Agent Team 的 Agent 该怎么划分：按职责、按阶段还是按能力？》
- 为什么只有 Workflow 不够 → 《没有 Harness 的 Agent Team，只是看起来像流水线》

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
