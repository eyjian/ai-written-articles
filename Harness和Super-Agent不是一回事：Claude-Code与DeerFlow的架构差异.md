# Harness 和 Super Agent 不是一回事：Claude Code 与 DeerFlow 的架构差异

> Harness 管的是"Agent 跑得住不住"，Super Agent 管的是"Agent 能做多少事"。两者不矛盾，但不在一个维度。Claude Code 是 Harness 做得很完整的单 Agent，DeerFlow 是 Harness 做得很完整的多 Agent 系统。搞清这个区分，选型和架构设计才不会跑偏。

讨论 AI Agent 架构时，一个常见的混淆是把 Harness 和 Super Agent 当成同一件事，或者觉得 Claude Code 和 DeerFlow 做的是一样的活。

不一样。它们在不同的维度上发力，解决的是不同层面的问题。

## 两个维度，不是两个阵营

先把坐标系画清楚。Harness 和 Super Agent 不是对立关系，而是两个独立的维度：

- **Harness（运行控制系统）**关心的是：Agent 在执行任务的过程中，怎么约束行为、隔离环境、管理上下文、闭环纠错、全程可追踪。它解决的是**稳定性和可控性**。
- **Super Agent（超级智能体）**关心的是：Agent 能覆盖多少种任务、能调度多少个子 Agent、能跨多少个领域工作。它解决的是**能力范围和任务复杂度**。

一个 Agent 系统可以只有 Harness 没有 Super Agent 能力（比如 Claude Code），也可以同时具备两者（比如 DeerFlow），也可以两者都缺（比如一个裸调模型的脚本）。

## Claude Code：单 Agent + 完备 Harness

Claude Code 是 Anthropic 的命令行 AI 编程工具。从 2026 年 3 月泄露的源码看，它的架构可以概括为：

> **一个 Agent + 一套非常完整的 Harness**

**Agent 层面**：只有一个 Agent——Claude 模型。没有多 Sub-Agent 并行，没有 Lead Agent 拆任务分配的机制。所有推理、决策、执行都由这一个 Agent 完成。

**Harness 层面**：做得极其完整——

| Harness 要素 | Claude Code 的实现 |
|---|---|
| 工具系统 | 文件操作、命令执行、代码检索、Git、MCP |
| 权限与安全 | 沙箱、白名单/黑名单、人工审批、审计日志 |
| 上下文与记忆 | 动态注入、历史压缩、`CLAUDE.md`、跨会话状态 |
| 反馈回路 | Agent Loop、失败重试、错误反馈、降级处理 |
| 验证与可观测 | diff 检查、自动测试、错误捕获、状态回滚 |

**任务范围**：聚焦编码领域。写代码、改代码、跑测试、修 Bug——这是它的主战场，不做通用任务。

Claude Code 强在 Harness 的深度，不在任务的广度。

## DeerFlow：多 Agent + 完备 Harness

DeerFlow 给自己的官方定位是 **Super Agent Harness**——这个标签同时包含了两层含义。

**Agent 层面**：Lead Agent + 动态拉起多个 Sub-Agents。面对复杂任务，Lead Agent 会拆解子任务，分配给不同的 Sub-Agent 并行执行，最后汇总结果。这是典型的 Agent Team 架构。

**Harness 层面**：同样完整——

| Harness 要素 | DeerFlow 的实现 |
|---|---|
| 工具系统 | 网页搜索、抓取、文件操作、Bash、MCP |
| 沙箱隔离 | 每个 Task 独立 Docker 容器，完整文件系统 |
| 上下文管理 | Sub-Agent 间上下文隔离、长会话压缩、转存 |
| 反馈回路 | Sub-Agent 失败重试、Lead Agent 重新分配 |
| 可观测性 | LangSmith 集成，全链路追踪 |

**任务范围**：跨领域通用。研究、编码、内容生成、幻灯片制作、网页构建——通过 Skills 体系可以不断扩展能力边界。

DeerFlow 既有 Harness 的深度，也有 Super Agent 的广度。

## 一张表看清两者的架构差异

| 维度 | Claude Code | DeerFlow |
|------|------------|----------|
| **Agent 架构** | 单 Agent | Lead Agent + 多 Sub-Agents |
| **Harness 完整度** | 完整（五要素全覆盖） | 完整（五要素全覆盖） |
| **任务范围** | 聚焦编码 | 跨领域通用 |
| **多 Agent 协作** | 无 | 有（并行 Sub-Agents） |
| **扩展机制** | MCP 协议接入外部工具 | Skills 体系 + MCP |
| **技术底座** | 自研 TypeScript 框架 | LangChain + LangGraph |
| **定位标签** | Harness（编码 Agent） | Super Agent Harness（通用 Agent） |

两者的 Harness 层能力大致对标，差异主要在 **Agent 架构**和**任务范围**上。

## 这个区分为什么重要

搞清 Harness 和 Super Agent 的区别，直接影响两个决策：

**选型决策**：如果需求是"让 Agent 在编码场景里稳定干活"，Claude Code 的单 Agent + 深度 Harness 架构就够了，不需要多 Agent 的复杂度。如果需求是"让 Agent 跨领域处理复杂任务"，需要 DeerFlow 这样的多 Agent + Harness 架构。

**架构设计决策**：自己搭 Agent 系统时，Harness 和 Super Agent 能力应该分开设计。先把 Harness 搭好（约束、沙箱、上下文、反馈、可观测），再决定是用单 Agent 还是多 Agent 架构。别反过来——先堆了一堆 Sub-Agent，结果 Harness 层是空的，跑起来全是乱。

这也是为什么 DeerFlow 把自己叫 "Super Agent Harness" 而不是只叫 "Super Agent"——它强调的是 Harness 先行，Super Agent 能力建立在 Harness 之上。

## 结论

一句话收尾：**Harness 管稳定性，Super Agent 管能力范围。两者不在一个维度，但最好同时具备。** Claude Code 证明了单 Agent + 完备 Harness 在垂直领域能走多远，DeerFlow 证明了多 Agent + 完备 Harness 在通用场景能覆盖多广。不管走哪条路，Harness 都是底座。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
