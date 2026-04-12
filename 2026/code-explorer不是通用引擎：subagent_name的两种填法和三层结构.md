# code-explorer 不是通用引擎：subagent_name 的两种填法和三层结构

> 运行日志里打出 "Task code-explorer subagent: writer"——code-explorer 和 writer 同时出现，到底谁是谁？前一篇《四个工具》把 scout、writer 放在 subagent_name 列，让人以为它们和 code-explorer 是同一层次。实际上，task 的 subagent_name 有两种用法：填内置 subagent（如 code-explorer），或者填自定义 Agent 名称。搞混了，Skill 会踩坑；搞清楚了，才知道自定义 Agent 的能力边界在哪。

## 一、问题从哪来

跑 article-team 工作流时，IDE 的输出面板里会刷出这样一行：

```text
Task code-explorer subagent: writer
```

第一反应大概是困惑。code-explorer 是什么？不是说好了 writer 才是写手角色吗？两个名字同时出现在一行日志里，谁管谁？

回头翻《CodeBuddy 的多 Agent 协作是怎么搭起来的：从 team_create 到 team_delete 的四个工具》（以下简称《四个工具》），第六章"可用的 Agent 类型"那张表把 scout、writer、reviewer 和 code-explorer 放在了同一个 `subagent_name` 列里，给人的印象是它们属于同一层次。

但日志里 `code-explorer` 和 `writer` 明显不在同一层。要理解这行日志，先得搞清楚一件事：`subagent_name` 这个参数有两种完全不同的填法。

---

## 二、subagent_name 的两种填法

task 的 `subagent_name` 参数看起来只是一个字符串，但填法不同，Agent 的性质截然不同。

### 填内置 subagent 名称

CodeBuddy 平台提供了少量内置 subagent，目前已知的有 `code-explorer` 和 `code-reviewer`。填内置名称时，Agent 以**同步子任务**的方式运行——执行完就销毁，不留在团队里。

```text
task(
  subagent_name: "code-explorer",
  description: "搜索项目里所有 .md 文件",
  prompt: "列出项目根目录下所有 Markdown 文件..."
)
```

这种用法下，code-explorer 拿到的是平台固定分配的工具集（主要是只读搜索类工具），任务结束后实例直接回收。

### 填自定义 Agent 名称

Skill 开发者可以定义自己的 Agent（通过 `.md` 文件描述角色），然后在 task 的 `subagent_name` 里直接填这个自定义名字，同时配合 `name` 和 `team_name` 进入团队模式。

```text
task(
  subagent_name: "article-writer",
  name: "writer",
  team_name: "article-team-20260412",
  mode: "bypassPermissions",
  prompt: "<agents/article-writer.md 的完整内容>"
)
```

这种用法下，Agent 以**团队成员**的方式常驻运行，通过 `send_message` 持续通信，直到收到 `shutdown_request` 才退出。工具集不是固定的，而是由角色定义的 frontmatter 中的 `tools` 列表声明。

### 两种填法的核心差异

| | 内置 subagent | 自定义 Agent |
|---|---|---|
| **subagent_name 填什么** | 平台内置名称（如 `code-explorer`） | 自定义角色名（如 `article-writer`） |
| **定义来源** | 平台内置 | `.md` 文件 |
| **工具集** | 平台固定分配 | frontmatter 中 `tools` 列表声明 |
| **通信方式** | 返回结果后销毁 | `send_message` 持续通信 |
| **生命周期** | 一次性任务 | 常驻，直到 `shutdown_request` |
| **是否需要 team_name** | 不需要 | 需要 |

关键区分点在这里：内置 subagent 是平台提供的"现成工具人"，能力固定、用完即走；自定义 Agent 是 Skill 自己定义的"团队成员"，能力可配、常驻协作。

---

## 三、那日志里的 code-explorer 是怎么来的

回到开头那行日志：

```text
Task code-explorer subagent: writer
```

既然自定义 Agent 的 subagent_name 可以直接填 `writer`，为什么日志里还是出现了 `code-explorer`？

这涉及到 Skill 的实现选择。article-team 这个 Skill 在实际运行时，选择了用 `code-explorer` 作为 subagent_name，然后用 `name` 参数赋予角色身份。也就是说，它走的是内置 subagent + 团队模式的混合用法：

```text
task(
  subagent_name: "code-explorer",   ← 用了内置 subagent
  name: "writer",                    ← 角色身份
  team_name: "article-team-20260412",
  mode: "bypassPermissions",
  prompt: "<agents/writer.md 的完整内容>"
)
```

这里有三个参数各管各的事：

| 参数 | 来源 | 管什么 |
|------|------|--------|
| `subagent_name` | 填了内置 subagent 名称 | 决定底层能力集 |
| `name` | Skill 定义的角色名 | 决定团队内身份标识 |
| `prompt` | 从 agents/*.md 读取 | 决定具体行为 |

### 为什么不直接填 writer

一种可能的原因：如果 Skill 没有在 `.codebuddy/agents/` 目录下把 `writer` 注册为自定义 subagent，平台无法识别这个名字，调用会失败或降级。用 `code-explorer` 作为 subagent_name 是一种稳妥的做法——借用内置 subagent 的能力集，再通过 prompt 注入角色行为。

另一种情况：如果 Skill 已经在 `.codebuddy/agents/` 目录下注册了自定义 Agent，就可以直接写 `task(subagent_name: "article-writer", name: "writer", ...)`，让 Agent 使用 frontmatter 里声明的工具集，不再依赖 code-explorer。

### 两种方式的流程对比

```text
方式 A：借用内置 subagent（article-team 当前做法）

  agents/writer.md ──→ Skill 读取文件内容
                            │
                            ▼
                       组装 task 参数
                            │
                            ├── subagent_name: "code-explorer"  ← 内置 subagent
                            ├── name: "writer"                   ← 角色身份
                            └── prompt: "<文件全文>"              ← 行为指令
                            │
                            ▼
                       Agent 实例启动
                       底座 = code-explorer 的能力集
                       身份 = writer
                       行为 = prompt 里写的一切


方式 B：注册自定义 Agent

  .codebuddy/agents/ 注册 article-writer ──→ 平台识别自定义 Agent
  agents/writer.md ──→ frontmatter 声明 tools 列表
                            │
                            ▼
                       组装 task 参数
                            │
                            ├── subagent_name: "article-writer"  ← 自定义 Agent
                            ├── name: "writer"                   ← 角色身份
                            └── prompt: "<文件全文>"              ← 行为指令
                            │
                            ▼
                       Agent 实例启动
                       工具集 = frontmatter 中声明的 tools
                       身份 = writer
                       行为 = prompt 里写的一切
```

日志里看到 `code-explorer` 出现在 `writer` 前面，说明当前 Skill 走的是方式 A。如果哪天日志变成了 `Task article-writer subagent: writer`，大概率是 Skill 改成了方式 B。

---

## 四、code-explorer 的能力边界

用方式 A（借用内置 subagent）还是方式 B（注册自定义 Agent），取决于场景需要什么能力。

code-explorer 作为内置 subagent，能力集是平台固定分配的，主要覆盖：

| 能力 | 说明 |
|------|------|
| 读文件（read_file） | 读取项目中的文件内容 |
| 搜索内容（search_content） | 在文件中搜索匹配模式 |
| 搜索文件（search_file） | 按文件名模式查找文件 |
| 列目录（list_dir） | 查看目录结构 |
| 代码搜索 | 代码库导航和理解 |

注意这个列表——以只读搜索类工具为主。code-explorer 的设计初衷是"代码探索"，不是"什么都能干"。

article-team 之所以用 code-explorer 还能跑通文章编写流程，从实际运行结果来看，团队模式下 Agent 实例获得的能力超出了 code-explorer 原始工具集的范围——观察到的现象是，Agent 可以使用 `send_message`、`write_to_file`、`replace_in_file` 等工具，而这些并不在 code-explorer 的原始能力集中。推测平台在团队模式下会为 Agent 注入额外的通信和文件操作能力。这也是为什么方式 A 能工作——Agent 的能力不完全等于内置 subagent 的原始工具集。

但如果需要更精确地控制 Agent 的工具集，方式 B 更合适。自定义 Agent 在 frontmatter 里显式声明需要哪些工具，比如：

```yaml
---
tools:
  - read_file
  - write_to_file
  - search_content
  - send_message
  - web_search
---
```

声明了什么就有什么，没声明的就没有。比 code-explorer 的"平台固定分配"更可控。

---

## 五、对《四个工具》第六章的勘误

回过头看，前文那张表需要澄清一下。

《四个工具》第六章"可用的 Agent 类型"给出了两张表。第一张列了 article-team 的角色：

| subagent_name | 角色 | 职责 |
|---------------|------|------|
| `scout` | 选题侦察员 | 搜索热点、调研资料、给出选题方案 |
| `writer` | 初稿写手 | 按大纲撰写 Markdown 初稿 |
| `reviewer` | 技术审稿人 | 审查准确性和完整性 |

第二张列了研发工作流的角色，最后两行是：

| subagent_name | 角色 | 职责 |
|---------------|------|------|
| `code-explorer` | 代码探索（内置） | 代码库导航和理解 |
| `code-reviewer` | 代码审查（内置） | 自动化代码审查 |

问题在于：scout、writer 和 code-explorer 被放在同一列（`subagent_name`），但它们的性质不同。code-explorer 是平台内置 subagent，scout 和 writer 是 Skill 自定义的角色。两者在工具集来源、生命周期、通信方式上有本质区别。

更准确的表述应该区分两个层次：

| 类型 | subagent_name | 性质 | 工具集来源 |
|------|--------------|------|-----------|
| 内置 | `code-explorer` | 平台提供 | 平台固定分配 |
| 内置 | `code-reviewer` | 平台提供 | 平台固定分配 |
| 自定义 | `article-scout` | Skill 定义 | frontmatter 声明 |
| 自定义 | `article-writer` | Skill 定义 | frontmatter 声明 |
| 自定义 | `article-reviewer` | Skill 定义 | frontmatter 声明 |

不过，article-team 当前的实现走的是方式 A——subagent_name 填 `code-explorer`，再通过 `name` 和 `prompt` 赋予角色身份和行为。所以在日志层面，看到的确实是 `code-explorer` 而不是 `writer`。《四个工具》如果是在描述这种实现方式，把 scout、writer 列在 subagent_name 列确实有误导性。

原文的表述更像是一种简写——省略了内置 subagent 和自定义 Agent 的区别，把所有名字混在了一列里。对于只想了解"团队里有哪些角色"的读者，这种简写不影响理解。但对于要自己写 Skill 的开发者，会产生困惑：到底是填 `code-explorer` 还是填 `writer`？

---

## 六、什么时候需要关心这些区别

不是所有人都需要搞清楚 subagent_name 的两种填法。

### 写 Skill 的人必须清楚

开发 Skill 时，第一个要做的决定就是：用内置 subagent 还是注册自定义 Agent。

选内置（方式 A），实现简单，不需要注册流程，但工具集受限于平台分配。选自定义（方式 B），需要在 `.codebuddy/agents/` 目录下注册、在 frontmatter 里声明工具，但能力完全可控。

一个典型的踩坑路径：Skill 开发者写好了 `agents/writer.md`，直接在 task 里填 `subagent_name: "article-writer"`，但忘了把 `.md` 文件放到 `.codebuddy/agents/` 目录下注册——平台不认识这个名字，调用失败或降级到默认行为。

### 普通使用者大多数时候不用管

日常使用 Agent Team 时，只需要知道团队里有哪些角色、各管什么事。subagent_name 填的是 code-explorer 还是 writer，对使用体验没有直接影响。Skill 已经把这些决策封装好了。

### 排查问题时看日志里的名字

日志里出现 `Task code-explorer subagent: writer`，说明 Skill 用的是方式 A。如果看到 `Task article-writer subagent: writer`，说明用的是方式 B。

碰到 Agent 行为异常时，先看日志里的 subagent_name 判断用的哪种方式，再看 prompt 是否正确注入。方式 A 的问题通常出在 prompt 注入；方式 B 的问题可能出在 frontmatter 的 tools 声明。

### 判断表

| 角色 | 场景 | 需要关心什么 |
|------|------|------------|
| Skill 开发者 | 选择 Agent 方案 | 内置 vs 自定义——能力需求决定 |
| Skill 开发者 | 注册自定义 Agent | .codebuddy/agents/ 注册 + frontmatter tools |
| Skill 开发者 | 编写 agents/*.md | prompt 内容——具体行为定义 |
| 普通使用者 | 日常使用 | 不需要关心 |
| 问题排查者 | 看日志 | subagent_name 判断方式 A/B |
| 问题排查者 | 分析行为异常 | 方式 A 查 prompt，方式 B 查 tools 声明 |

---

判断 subagent_name 该填什么，归结到一个问题：平台的内置 subagent 能力集够不够用。够用就借用内置的，省事；不够用或者需要精确控制工具集，就注册自定义 Agent。article-team 当前走的是前者，日志里出现 code-explorer 就是这个原因。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
