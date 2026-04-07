# DeerFlow 入门实战：从零跑通一个 Super Agent Harness

> 这篇不聊概念，直接上手。从 clone 仓库到跑通第一个多 Agent 任务，再到自定义一个 Skill，把 DeerFlow 的核心能力过一遍。前提是对 DeerFlow 有基本了解——如果还没看过，建议先读《认识 DeerFlow：一个跑在 LangGraph 上的 Super Agent Harness》。

DeerFlow 的文档和 README 写得很全，但对第一次上手的人来说，信息量有点大。这篇把路径收窄：只走一条最短的线——**clone → 配置 → 启动 → 跑任务 → 体验 Harness 能力 → 自定义 Skill**，中间穿插实际会踩到的坑。

## 跑通需要什么

先把环境要求摆出来。

### 环境速查表

| 依赖 | 最低版本 | 说明 |
|------|---------|------|
| Python | 3.12+ | DeerFlow 后端运行时 |
| Node.js | 22+ | 前端构建 |
| Docker | 最新稳定版 | 沙箱隔离必须，推荐部署方式也依赖它 |
| uv | 最新版 | Python 包管理，替代 pip |
| pnpm | 最新版 | 前端包管理 |
| nginx | 最新版 | 本地开发路径需要，Docker 路径不需要 |
| Git | — | 克隆仓库 |

### 需要的 API Key

| Key | 用途 | 获取方式 |
|-----|------|---------|
| 模型 API Key | 驱动 LLM（OpenAI / DeepSeek / 豆包等） | 对应平台申请 |
| TAVILY_API_KEY | 网页搜索工具 | [tavily.com](https://tavily.com) 注册 |

模型选择建议：DeerFlow 官方推荐具备长上下文（100k+ tokens）、强推理能力和稳定 Tool Use 的模型。国内可选豆包 Seed 2.0 Code、DeepSeek v3.2、Kimi 2.5 等。

### 两条部署路径

| 路径 | 适合谁 | 复杂度 |
|------|--------|--------|
| **Docker 部署**（推荐） | 想快速体验、不想折腾依赖的人 | 低 |
| **本地开发** | 需要改源码或调试的开发者 | 中 |

下面两条路径都会讲。如果只想快速体验，直接走 Docker 路径。

## 第一步：克隆和配置

不管走哪条路径，这一步都一样。

```bash
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow
make config
```

`make config` 会在项目根目录生成配置文件。接下来编辑两个东西：`config.yaml` 和 `.env`。

### 配置模型（config.yaml）

打开 `config.yaml`，至少配一个模型。以 DeepSeek 为例：

```yaml
models:
  - name: deepseek-v3
    display_name: DeepSeek V3.2
    use: langchain_openai:ChatOpenAI
    model: deepseek-chat
    base_url: https://api.deepseek.com/v1
    api_key: $DEEPSEEK_API_KEY
    max_tokens: 8192
    temperature: 0.7
```

注意 `use` 字段——DeerFlow 直接用 LangChain 的类路径加载模型，所以任何 LangChain 支持的模型提供商都可以接入。这就是它和 LangChain 的关系在配置层的体现。

如果用 OpenAI 兼容接口（比如 OpenRouter），换一下 `base_url` 和 `model` 就行。

### 配置 API Key（.env）

在项目根目录的 `.env` 文件里：

```bash
# 模型 Key（按实际使用的平台配置）
DEEPSEEK_API_KEY=your-deepseek-key-here
# 或
OPENAI_API_KEY=your-openai-key-here

# 搜索工具
TAVILY_API_KEY=your-tavily-key-here
```

### 常见坑：Windows 用户注意

DeerFlow 不支持原生 cmd.exe 或 PowerShell。Windows 上**必须用 Git Bash** 运行所有 make 命令。这个坑不少人踩，README 里有写但容易忽略。

## 第二步：启动服务

### Docker 路径（推荐）

两条命令：

```bash
# 1. 拉取沙箱镜像（首次运行必须，后续不用重复）
make docker-init

# 2. 启动服务（支持热更新）
make docker-start
```

启动完成后，访问 **http://localhost:2026**，看到 DeerFlow 的 Web UI 就说明成功了。

如果要跑生产模式（构建优化镜像）：

```bash
make up        # 启动
make down      # 停止
```

### 本地开发路径

```bash
# 1. 检查环境是否满足要求
make check

# 2. 安装所有依赖
make install

# 3. 启动开发服务
make dev
```

同样访问 **http://localhost:2026** 验证。如果未配置 nginx 反向代理，前端可能跑在 **http://localhost:3000**，后端 API 在另一个端口——具体以终端输出为准。

### 验证启动成功的标志

- 浏览器打开 http://localhost:2026 能看到聊天界面
- 终端没有报错退出
- 如果走 Docker 路径，`docker ps` 能看到 DeerFlow 相关容器在运行

## 第三步：跑通第一个任务

打开 Web UI，在对话框里输入一个任务。推荐先用一个研究类任务试水，比如：

```
帮我调研 Harness Engineering 的核心实践，列出主要框架和方法论
```

然后观察 DeerFlow 的执行过程。会看到几件事发生：

1. **Lead Agent 接收任务**，分析后决定拆成几个子任务
2. **Sub-Agents 被动态拉起**，各自领一块去执行——有的去搜索网页，有的去抓取页面内容，有的去整理结果
3. **并行执行**——多个 Sub-Agent 同时工作，而不是排队串行
4. **Lead Agent 汇总**——各 Sub-Agent 完成后，Lead Agent 把结果整合成最终输出

这就是 DeerFlow 的 Agent Team 在运行：Lead Agent 是 Manager，Sub-Agents 是按需拉起的团队成员。整个过程跑在 Harness 里——每个 Sub-Agent 有独立上下文、独立工具集、独立终止条件。

如果任务执行过程中需要运行代码，会看到 Agent 在 Docker 沙箱里操作——这就是 Harness 的沙箱隔离能力在工作。

## 第四步：体验 Harness 能力

第一个任务跑通之后，可以针对性地体验 DeerFlow 的 Harness 能力。

### 沙箱隔离

输入一个需要执行代码的任务：

```
用 Python 写一个脚本，抓取 Hacker News 首页的前 10 条标题，保存到文件
```

观察 Agent 的执行——它会在 Docker 容器里运行 Python 代码，有独立的文件系统路径（`/mnt/user-data/workspace/`）。代码执行结果不会污染宿主机环境，这就是沙箱隔离。

### Skills 约束

DeerFlow 内置了多个 Skill，比如研究、报告生成、幻灯片制作、网页生成等。可以在 Web UI 的设置里看到当前启用的 Skills。

每个 Skill 本质上是一个 Markdown 文件，定义了 Agent 在执行特定类型任务时应该遵循的工作流和最佳实践。这是 DeerFlow 实现"约束机制"的主要方式——不是靠硬编码的规则，而是靠结构化的 Skill 定义。

### 可观测性（接入 LangSmith）

如果想看完整的调用链路，在 `.env` 里配置：

```bash
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your-langsmith-api-key
LANGCHAIN_PROJECT=deerflow-demo
```

重启服务后，每次任务执行的完整轨迹都会出现在 LangSmith 面板里——包括每次 LLM 调用的输入输出、工具执行结果、Agent 的决策过程。这是排查"Agent 为什么跑偏了"的利器。

### 长期记忆

连续发几轮对话，然后关掉浏览器，重新打开同一个会话。看看 DeerFlow 是否还记得之前的上下文和偏好。长期记忆保存在本地，数据由用户完全控制。

### 反馈回路

反馈回路是 Harness Engineering 的核心要素之一——Agent 出了错不用人介入，系统自己能纠偏。

在 DeerFlow 里体验这个能力，可以故意给一个容易出错的任务：

```
用 Python 调用一个不存在的第三方库 fakelib，抓取某个网页的标题
```

观察执行过程：Sub-Agent 在沙箱里运行代码时会报 `ModuleNotFoundError`。关键是接下来发生什么——DeerFlow 的 Sub-Agent 会捕获这个错误，把失败信息反馈给自己（或回传给 Lead Agent），然后自动调整方案：换一个可用的库、修正代码、重新执行。

这条"执行 → 失败 → 反馈 → 修正 → 重试"的回路不需要人工干预。如果接入了 LangSmith，可以在调用链路里清楚看到重试过程的每一步决策。

还可以试一个更复杂的场景：给一个需要多步骤完成的研究任务，中间某个搜索源返回空结果。观察 Lead Agent 是否会重新分配任务给其他 Sub-Agent，或调整搜索策略。这就是多 Agent 层面的反馈回路。

### 防御机制

DeerFlow 的防御机制体现在几个层面，可以逐一体验：

**沙箱级防御**——Agent 执行的所有代码和文件操作都在 Docker 容器里，天然隔离了对宿主机的危险操作。试试在对话里要求 Agent 访问宿主机的系统路径，会发现它只能操作沙箱内的 `/mnt/user-data/` 目录。

**上下文隔离**——给 DeerFlow 一个需要拆成多个 Sub-Agent 的任务，然后观察：Sub-Agent A 的中间结果不会泄漏到 Sub-Agent B 的上下文里。每个 Sub-Agent 只能看到自己被分配的任务和工具。这防止了 Agent 之间的上下文污染——一个 Sub-Agent 的错误不会扩散到整个团队。

**Token 溢出防御**——给一个需要处理大量信息的长任务（比如"调研 10 个 AI Agent 框架的优缺点"）。观察 DeerFlow 如何在长会话中积极总结子任务结果、压缩中间上下文、转存到文件系统，而不是让 Token 持续膨胀直到溢出。

这些防御不像 CI/CD 的硬门禁那样明确拦截，更像是运行时的安全网——跑偏了能拉回来，出错了不扩散，信息量大了能自动压缩。

## 第五步：自定义一个 Skill

Skills 是 DeerFlow 区别于普通 Agent 框架的关键能力之一。理解它的最好方式是自己写一个。

### Skills 的本质

每个 Skill 是一个 Markdown 文件，放在项目的 Skills 目录下（具体路径可在仓库中搜索已有的内置 Skill 来确认）。它定义了：

- **这个 Skill 做什么**：任务类型和适用场景
- **怎么做**：工作流步骤、最佳实践
- **边界在哪**：不该做什么、什么时候该停下来

Agent 在执行任务时会按需加载匹配的 Skill，按其中定义的工作流和约束来执行。这就是 Harness Engineering 里"约束机制"的落地方式。

### 创建一个简单的 Skill

比如创建一个"技术概念对比"Skill，让 Agent 在对比两个技术概念时遵循固定结构：

在 Skills 目录下创建 `tech-comparison.md`（参考已有内置 Skill 的存放位置）：

```markdown
# 技术概念对比

## 适用场景
当用户要求对比两个或多个技术概念、工具或框架时触发。

## 工作流
1. 明确对比对象和对比维度
2. 分别调研每个对象的核心特征
3. 按统一维度整理对比表
4. 给出判断：什么场景该用哪个

## 输出要求
- 必须包含一张对比表
- 必须给出"什么时候该用 / 什么时候别用"的判断
- 不做无依据的优劣评价

## 边界
- 不替用户做最终决策
- 如果某个对象的信息不够可靠，明确标注
```

保存后，重启服务（或根据文档说明热加载）。然后在 Web UI 里输入：

```
对比 DeerFlow 和 CrewAI，帮我判断该用哪个
```

观察 Agent 是否按照 Skill 定义的结构来执行——有没有出对比表、有没有给判断、有没有标注边界。

这就是"用 Markdown 写约束"的实际效果。约束的强度取决于 Skill 写得多严格——这也是 DeerFlow 当前的边界：它的约束是软约束，不是 CI/CD 式的硬门禁。

## 从理论代码到 DeerFlow：Harness Engineering 怎么落的地

如果看过《Harness Engineering 从零理解到动手实践》，会发现里面有大量示意代码——状态机、权限边界、反馈回路、循环检测、上下文管理。一个自然的问题是：**这些在 DeerFlow 里到底对应什么？是不是需要自己写这些代码？**

答案是：**不需要。DeerFlow 已经把这些机制内置了。** 下面逐一对照。

### 状态机 → DeerFlow 的 LangGraph 编排

Harness Engineering 文章里写了一个 `PhaseStateMachine`，用来锁定 Agent 的执行阶段（RESEARCH → PLAN → EXECUTE → VERIFY），控制每个阶段能用哪些工具。

在 DeerFlow 里，这个能力由 **LangGraph** 提供。LangGraph 本身就是状态机驱动的编排引擎——每个节点是一个状态，边定义了合法的转移。DeerFlow 的 Lead Agent 在拆解任务、分配 Sub-Agent、汇总结果的过程中，走的就是 LangGraph 定义的状态图。

**不需要自己写 `PhaseStateMachine`**，但如果想自定义编排逻辑（比如增加一个"人工审批"节点），可以修改 DeerFlow 后端的 LangGraph 图定义。

### 权限边界 → DeerFlow 的 Docker 沙箱

文章里写了一个 `PermissionBoundary`，用代码拦截对 `.env`、`secrets/`、`.git/` 等路径的访问。

DeerFlow 用了一种更彻底的方案：**Docker 沙箱隔离**。Agent 的所有文件操作和代码执行都在 Docker 容器内部进行，容器只挂载了 `/mnt/user-data/` 下的三个目录（uploads、workspace、outputs）。Agent 根本接触不到宿主机的文件系统——不是"不让碰"，而是"看都看不到"。

这比代码层的路径拦截更安全：即使 Agent 试图执行 `rm -rf /`，也只影响容器内部，不会波及宿主机。

### 循环失败检测 → DeerFlow 的 Sub-Agent 重试机制

文章里写了一个 `LoopDetector`，检测 Agent 是否在重复同一个失败动作，超过阈值就强制打断。

DeerFlow 的 Sub-Agent 机制内置了类似逻辑：Sub-Agent 执行失败后，Lead Agent 会收到失败信息，判断是否需要换一个策略、重新分配任务、或直接放弃该子任务。这不是一个显式的 `LoopDetector` 类，而是 Lead Agent 的推理能力 + LangGraph 状态转移共同实现的。

如果想更严格地控制重试次数，可以在 LangGraph 图的边上加条件判断——比如某个节点失败超过 3 次就跳转到"降级处理"节点。

### 自动化验证 → DeerFlow 的沙箱 + 可观测性

文章里写了一个 `FeedbackLoop`，在 Agent 改完代码后自动跑 typecheck、lint、test，不通过就打回。

DeerFlow 的沙箱天然支持这个场景——Agent 在容器内执行代码时，可以在同一个沙箱里运行测试。如果测试失败，错误信息会直接反馈给 Agent，Agent 会据此修正代码并重新执行。

配合 LangSmith 的可观测性，每一次"执行 → 失败 → 反馈 → 修正"的完整链路都可以在调用追踪里看到。

### 双 Agent 评审 → DeerFlow 的 Sub-Agent 协作

文章里写了一个 `dual_agent_review`，一个 Agent 写代码、另一个 Agent 审查。

DeerFlow 的 Sub-Agent 机制天然支持这种分工：Lead Agent 可以拉起一个 Sub-Agent 负责执行，再拉起另一个 Sub-Agent 负责审查。两个 Sub-Agent 上下文完全隔离，审查 Agent 不会受到执行 Agent 的"自我暗示"影响。

### 上下文管理 → DeerFlow 的上下文工程

文章里写了一个 `ContextManager`，当上下文利用率超过 60% 时，让 Agent 生成交接文档、重置会话。

DeerFlow 内置了更完整的上下文工程：

- **Sub-Agent 间上下文完全隔离**——每个 Sub-Agent 只能看到自己被分配的任务和工具，不会被其他 Sub-Agent 的中间结果污染
- **长会话自动压缩**——积极总结子任务结果，转存中间产出到文件系统，压缩上下文占用
- **长期记忆**——跨会话积累的知识保存在本地，新会话自动加载

### 经验持久化 → DeerFlow 的长期记忆 + Skills

文章里建议维护一份 `.harness/lessons-learned.md`，把 Agent 的典型错误记录下来。

在 DeerFlow 里，这由两个机制覆盖：

- **长期记忆**：用户偏好、历史交互中的关键信息会自动保存，跨会话可用
- **Skills**：如果某类错误反复出现，可以把防护措施写进自定义 Skill——这等于把经验固化成了约束规则

### 对照总结

| Harness Engineering 理论 | 文章里的示意代码 | DeerFlow 的实际实现 |
|---|---|---|
| 状态机锁定阶段 | `PhaseStateMachine` | LangGraph 状态图编排 |
| 权限边界 | `PermissionBoundary` | Docker 沙箱隔离（文件系统级） |
| 循环失败检测 | `LoopDetector` | Lead Agent 推理 + LangGraph 条件转移 |
| 自动化验证 | `FeedbackLoop` | 沙箱内执行测试 + 错误自动反馈 |
| 防"嘴上完成" | `ToolCallValidator` | Sub-Agent 独立终止条件 + Lead Agent 汇总验证 |
| 双 Agent 评审 | `dual_agent_review` | Sub-Agent 协作（执行/审查分离） |
| 上下文重置 | `ContextManager` | 上下文隔离 + 自动压缩 + 文件转存 |
| 经验持久化 | `lessons-learned.md` | 长期记忆 + 自定义 Skills |

**一句话总结：Harness Engineering 文章里那些需要自己写的编排代码，DeerFlow 已经封装好了。** 理解了理论，再看 DeerFlow 的设计就会很清楚——它不是随便堆砌功能，每一个能力都对应着 Harness Engineering 的某个具体要素。

## 常见问题和排坑

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| `make docker-init` 拉镜像失败 | Docker Hub 网络不通 | 配置 Docker 镜像加速源，或手动拉取镜像 |
| 模型调用超时 | base_url 配错或 API Key 无效 | 检查 `config.yaml` 的 base_url 和 `.env` 的 Key |
| Windows 上 make 命令报错 | 用了 cmd 或 PowerShell | 切换到 Git Bash |
| 启动后 Web UI 打不开 | 端口被占用或服务未完全启动 | 检查终端输出，确认无报错；用 `lsof -i :2026` 查端口 |
| Agent 执行代码失败 | 沙箱容器未启动 | 确认 `make docker-init` 已执行过，且 Docker 正在运行 |
| 国内访问 OpenAI 模型慢或不通 | 网络限制 | 换用国产模型（豆包、DeepSeek、Kimi）或走 OpenRouter |

## 结论

一句话收尾：**跑通 DeerFlow 不难，难的是理解它为什么这么设计——每一个看起来"多余"的层（沙箱、Skills、上下文隔离、可观测），都是 Harness Engineering 在工程上的落地。**

这篇走完了最短路径：clone → 配置 → 启动 → 跑任务 → 体验 Harness 能力 → 自定义 Skill。如果想理解 DeerFlow 在 AI Agent 工程体系里的位置——它和 Harness Engineering、Agent Team、Workflow、LangChain/LangGraph 分别是什么关系——可以回头看《认识 DeerFlow：一个跑在 LangGraph 上的 Super Agent Harness》。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
