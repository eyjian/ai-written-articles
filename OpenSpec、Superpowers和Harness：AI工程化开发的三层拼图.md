# OpenSpec、Superpowers 和 Harness：AI 工程化开发的三层拼图

> AI 编程从"让模型写代码"走向"让模型像团队一样开发"，中间差的不是更强的模型，而是三层工程基础设施：OpenSpec 管"做什么"，Superpowers 管"怎么做"，Harness 管"谁来做、怎么协同"。三者各管一层，合在一起才是完整的工程化开发体系。

用 AI 写代码，很多人经历过类似的演进：

一开始是让模型直接写。能跑，但质量不可控——没有规范、没有测试、没有审查，改了这里崩了那里。后来开始给模型加规则，写 `AGENTS.md`、加 Prompt 约束、上 Harness，稳了不少。再后来上多 Agent 协作——角色分工、并行开发、自动审查——但新问题又来了：每个 Agent 按自己的理解做事，规范不统一，流程不一致。

要解决第三阶段的问题，需要三层东西同时到位：**统一的规范、统一的流程纪律、统一的团队协作框架。** 对应的就是 OpenSpec、Superpowers 和 Harness。

## 三层各管什么

| 层次 | 工具 | 管什么 | 类比 |
|------|------|--------|------|
| **规范层** | OpenSpec | 做什么——需求、接口、验收标准 | 施工图纸 |
| **纪律层** | Superpowers | 怎么做——TDD、代码审查、验证流程 | 施工规范手册 |
| **协作层** | Harness / Agent Team | 谁来做——角色分工、任务调度、权限管控 | 项目经理 + 工地管理 |

一句话关系：**OpenSpec 定标准，Superpowers 保纪律，Harness 管团队。**

## OpenSpec：规范层——锁定"做什么"

OpenSpec 是一个 Spec-Driven Development（规范驱动开发）框架，由 Fission-AI 开源。

它解决的核心问题是：**多个 Agent 协作时，怎么保证大家对"做什么"的理解一致？**

没有 OpenSpec 的场景：你在聊天窗口里告诉 Agent "加一个用户认证模块"，Agent 按自己的理解去做。做出来发现接口不对、字段命名不一致、验收标准不明确。改了三轮还在返工。

有 OpenSpec 的场景：先写一份 spec 文件，明确定义需求、接口、数据模型、验收条件。所有 Agent 读同一份 spec 开工，做完了按 spec 验证——符合就通过，不符合就打回。

### OpenSpec 的工作流

```
propose（提出变更）→ spec（编写规格）→ verify（验证产出）→ archive（归档）
```

输出的 `openspec/` 目录包含：提案、规格、任务拆解、验证结果——结构化、可追溯、所有 Agent 共享。

### 关键价值

- 需求不再只活在聊天记录里，而是**结构化文档**
- 多 Agent 共享同一份规范，不会各做各的
- 验收有明确标准，不是"看起来差不多就行"

## Superpowers：纪律层——锁定"怎么做"

Superpowers 是一个面向 AI 编程的技能系统，由 obra 开源。

它解决的核心问题是：**怎么让 Agent 按工程最佳实践写代码，而不是走野路子？**

没有 Superpowers 的场景：Agent 直接上手写代码，不写测试、不做计划、不走审查。代码能跑但不可维护，测试覆盖率为零。

有 Superpowers 的场景：Agent 在开发前被加载了 TDD（测试驱动开发）技能——先写测试、再写实现、实现完了跑测试、不通过就改。代码审查技能确保每段代码都经过评审。验证技能确保不会"宣称完成但实际没做"。

### Superpowers 的核心能力

| 技能 | 作用 |
|------|------|
| `test-driven-development` | 先写测试再写实现，不通过就改 |
| `writing-plans` | 开发前先出计划，不允许直接上手 |
| `code-review` | 代码必须经过评审才能合并 |
| `verification-before-completion` | 完成前必须验证，防止"嘴上完成" |
| `brainstorming` | 复杂问题先头脑风暴再动手 |

技能是纯 Markdown 定义，零依赖，可以嵌入任何 AI 会话。按需加载——不是一次性塞满，而是根据任务类型激活对应技能。

### 关键价值

- Agent 的开发流程有纪律，不是想怎么写就怎么写
- 强制遵循 TDD、审查、验证等工程最佳实践
- 产出质量有保障，不只是"能跑"，而是"能维护"

## Harness / Agent Team：协作层——锁定"谁来做"

Harness 在这个体系里负责的是**多 Agent 的编排与管控**。

它解决的核心问题是：**多个 Agent 怎么分工、怎么并行、怎么不互相冲突？**

没有 Harness 的场景：五个 Agent 各自领了任务开始干，但没人统一分工。前端 Agent 和后端 Agent 对接口的理解不一致，测试 Agent 拿到的是过时的代码，安全 Agent 根本不知道其他人改了什么。

有 Harness 的场景：

- **角色定义**：`AGENTS.md` 明确每个 Agent 的职责——架构师、后端、测试、安全、Team Lead
- **任务调度**：Team Lead Agent 读取 OpenSpec 的任务列表，分派给对应角色
- **权限管控**：每个 Agent 只能操作自己负责的文件和目录
- **硬约束**：提交前必须跑测试、Lint、安全扫描——不通过就不让合并
- **反馈回路**：测试失败自动回灌给开发 Agent，修复后重新验证

### 关键价值

- 多 Agent 协作有分工、有秩序，不是各自为战
- 权限隔离防止越权，硬约束防止偷工减料
- 任务可并行、可追踪、可回滚

## 三者怎么串起来

一个完整的开发周期走下来大致是这样：

```
Phase 1：规范
  OpenSpec 定义需求、接口、验收标准
  → 产出 openspec/ 目录，所有 Agent 共享

Phase 2：分工
  Harness 组建 Agent Team，分配角色
  → 架构师读 spec 输出架构，Team Lead 拆任务分派

Phase 3：开发
  各 Agent 按 Superpowers 的技能纪律开发
  → TDD 写代码、代码审查、完成前验证

Phase 4：验收
  OpenSpec 验证产出是否符合 spec
  → 通过则归档，不通过则打回
```

**调用链条**：Harness 启动 → 加载 `AGENTS.md` → Architect Agent 读 `openspec/specs/` → Superpowers 激活 TDD → Backend Agent 按 spec + TDD 写代码 → Reviewer Agent 按 Superpowers 规则评审 → Harness 校验权限、运行钩子 → OpenSpec 验证 → 归档。

三层的配合原则是：**规范不变、流程不变、团队可弹性扩展。** OpenSpec 和 Superpowers 定义的是稳定的标准和纪律，Harness 定义的是灵活的团队结构和调度策略——项目大了可以加 Agent，项目小了可以减，但规范和纪律不变。

## 没有这三层会怎样

| 缺什么 | 会出什么问题 |
|--------|------------|
| 缺 OpenSpec | Agent 各做各的，需求理解不一致，返工率高 |
| 缺 Superpowers | Agent 走野路子，不写测试、不做审查、代码质量不可控 |
| 缺 Harness | Agent 协作混乱，权限冲突、任务重复、无法并行 |
| 三个都缺 | 裸模型写代码——能跑但不敢用，每次结果不一样 |

反过来，三个都有的时候：

- 需求有文档、有验收标准（OpenSpec）
- 开发有纪律、有最佳实践（Superpowers）
- 协作有分工、有管控、有兜底（Harness）

这才是**从 Demo 到生产级**的完整跨越。

## 落地建议

如果准备在项目里引入这套体系，不用一步到位。按优先级来：

1. **先上 OpenSpec**——哪怕只是手写一份需求 spec，也比在聊天窗口里口头描述强十倍
2. **再上 Superpowers 的核心技能**——至少启用 `test-driven-development` 和 `verification-before-completion`，这两个投入产出比最高
3. **最后上 Harness / Agent Team**——等需求规范和开发纪律稳定了，再上多 Agent 协作，否则分工越多乱得越厉害

**避坑提醒**：

- Superpowers 技能不要全部加载，按需启用。全加载会让上下文过长，模型反而表现变差
- `AGENTS.md` 写精简——只写"先看什么、按什么流程"，架构和业务细节放 `docs/`
- 如果 Harness 自带代码审查能力，关掉 Superpowers 里的对应技能，避免重复和冲突

## 结论

一句话收尾：**OpenSpec 锁需求，Superpowers 锁纪律，Harness 锁协作——三层拼齐，AI 才能从"写代码的助手"变成"可靠的开发团队"。**

这不是未来的愿景，是现在就能落地的工程实践。OpenSpec、Superpowers 和 Harness 都是开源项目，文档齐全，可以从最小配置开始用起。关键不在于一步到位，而在于知道三层各管什么、缺哪层会出什么问题。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
