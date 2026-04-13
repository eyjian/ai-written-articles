# 文章大纲：基于源码深度分析 Superpowers：从行为塑造到工作流编排

> 定位：源码级深度分析，面向有一定基础的开发者/AI 工程实践者。区别于已有的概览文章（《Superpowers 是什么》）和关系梳理文章（《三层拼图》《实战配合》），本文从源码出发，拆解 Superpowers 内部的行为塑造机制、技能编排逻辑和多平台适配实现，并附带与 OpenSpec、CodeBuddy Plan 的结构性对比。

---

## 引言（场景切入）

- 切入角度：大多数人知道 Superpowers 是"给 AI 加纪律"，但纪律是怎么加上去的？一条 Markdown 写的 Iron Law，凭什么能让 Agent 真的不跳步骤？
- 引出本文核心问题：**Superpowers 的源码里，到底藏了哪些"让 Agent 听话"的具体手段？**
- 明确本文不重复已有文章的概览内容，聚焦源码层面的机制分析

---

## 一、先给判断

  1. Superpowers 的 skill 不是文档，是**行为塑造代码**——它用 Iron Laws、Rationalization Prevention Table、Red Flags Table 三张"心理学武器"逼 Agent 遵守纪律
  2. 14 个 skill 不是平铺的，是有**依赖关系和工作流编排**的——using-superpowers 是入口，brainstorming → writing-plans → TDD → review → verification 是主链路
  3. 多平台适配靠的是 `hooks/session-start` 这个 Bash 脚本，在 session 启动时把 using-superpowers 的内容注入 Agent 上下文——**零代码依赖，纯文本注入**

---

## 二、行为塑造的三板斧：Iron Laws、Rationalization Prevention、Red Flags

> 本节是全文核心，从源码拆解 Superpowers 如何用纯文本"控制" Agent 行为

### 2.1 Iron Laws（铁律）

- 从 TDD skill 源码拆出具体铁律内容："NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST"
- 从 systematic-debugging 拆出："NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST"
- 从 verification-before-completion 拆出："NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE"
- 分析铁律的写法特点：全大写、祈使句、绝对化表述——这不是建议，是命令
- 关键洞察：**铁律是写给 LLM 的 prompt engineering，全大写和绝对化表述在 prompt 层面确实有更强的约束效果**

### 2.2 Rationalization Prevention Table（理性化防御表）

- 从源码中提取具体示例（如 TDD skill 中的 rationalization table）
- 表的结构：左列"Agent 会找的借口"，右列"为什么这个借口不成立"
- 示例摘录（2-3 条典型条目）
- 关键洞察：**这是在对抗 LLM 的"合理化逃避"倾向——模型不是不知道规则，而是会给自己找理由绕开规则。防御表提前堵死了常见的逃避路径**

### 2.3 Red Flags Table（红旗表）

- 从 using-superpowers 源码提取 Red Flags 表
- 表的结构：左列"如果你在想这些念头"，右列"停下来，你在合理化"
- 分析典型条目："This is just a simple question" → "Questions are tasks. Check for skills."
- 关键洞察：**Red Flags 是元认知层面的约束——不是告诉 Agent 不能做什么，而是告诉它"如果你在想跳过某个步骤，这个想法本身就是问题"**

### 2.4 三板斧的协同：为什么单用一个不够

- Iron Laws 定底线 → Rationalization Prevention 堵借口 → Red Flags 卡念头
- 类比：法律条文 + 反诡辩手册 + 自我审查清单
- 表格对比三者的作用层次

---

## 三、14 个 Skill 的依赖关系与工作流编排

### 3.1 Skill 分类总览

- 用表格呈现 5 大类 14 个 skill 的分类（元技能、协作流程、开发纪律、代码审查、分支管理）
- 标注每个 skill 的类型：Rigid（严格遵循）vs Flexible（灵活适配）

### 3.2 主工作流链路：从 brainstorming 到 finishing

- 用 ASCII 流程图展示主链路：
  ```
  using-superpowers（入口）
    → brainstorming（设计先行）
    → writing-plans（写计划）
    → [executing-plans | subagent-driven-development]（执行）
    → test-driven-development（TDD 实现）
    → systematic-debugging（调试）
    → requesting-code-review（请求审查）
    → receiving-code-review（接收审查反馈）
    → verification-before-completion（完成前验证）
    → finishing-a-development-branch（收尾）
  ```
- 从源码分析 skill 间的触发关系：using-superpowers 中如何引导到其他 skill

### 3.3 两条执行路径：executing-plans vs subagent-driven-development

- 从源码对比两者的差异
- executing-plans：当前 session 内联执行，适合简单任务
- subagent-driven-development：派发子代理，适合可并行的独立任务
- dispatching-parallel-agents：并行调度的具体机制
- 表格对比：适用场景、粒度、复杂度

### 3.4 元技能：using-superpowers 和 writing-skills

- using-superpowers 是整个系统的引导入口，分析其源码中的 skill 路由逻辑
- writing-skills 是"写 skill 的 skill"——引用了 agentskills.io 的 OpenSpec 规范（衔接下一节）

---

## 四、多平台适配机制：hooks/session-start 引导注入

### 4.1 引导机制的源码分析

- `hooks/session-start` 是一个 Bash 脚本，在 session 启动时执行
- 它做的事：读取 using-superpowers skill 的 Markdown 内容，注入到 Agent 的初始上下文中
- 零代码依赖——不需要 npm、pip、任何 runtime

### 4.2 六个平台的适配方式

- 用表格展示 Claude Code、Cursor、Codex、OpenCode、Copilot CLI、Gemini CLI 各自的适配文件和注入点
- 关键差异：hook 机制、配置文件位置、上下文注入方式
- 源码中的平台检测和分支逻辑

### 4.3 为什么选择"文本注入"而不是"API 集成"

- 设计哲学：纯文本 = 零依赖 = 跨平台
- 代价：没有运行时强制力，所有约束都是"prompt 级"的
- 与 Harness Engineering 的硬约束对比：Superpowers 是"高水平的 prompt engineering"，不是"代码级门禁"

---

## 五、与 OpenSpec 的关系：遵循者 vs 定义者

### 5.1 OpenSpec 定义了什么

- OpenSpec 是 skill 的格式标准（YAML frontmatter 字段、目录结构、分发规范）
- 来源：agentskills.io/specification

### 5.2 Superpowers 如何遵循 OpenSpec

- 从 Superpowers 的 skill 文件结构看 OpenSpec 规范的落地
- writing-skills 中对 agentskills.io 的直接引用
- YAML frontmatter 的具体字段对照

### 5.3 一句话关系

- OpenSpec 管"skill 长什么样"（格式标准）
- Superpowers 管"skill 装什么内容"（具体实现）
- 类比：OpenSpec 是 HTML 规范，Superpowers 是按规范写的一组网页

---

## 六、与 CodeBuddy Plan 的对比：平台能力 vs 插件能力

### 6.1 CodeBuddy Plan 是什么

- IDE 内置的规划功能，用户发起后 Agent 先输出计划再执行
- 平台级能力，开箱即用

### 6.2 Superpowers 的计划系统

- writing-plans + executing-plans / subagent-driven-development
- 从源码看计划的粒度要求（每步 2-5 分钟）
- 强制 TDD、两阶段 review（spec compliance + code quality）

### 6.3 结构性对比

| 维度 | CodeBuddy Plan | Superpowers Plan |
|------|----------------|------------------|
| 性质 | 平台内置能力 | 插件/skill 能力 |
| 启动方式 | 用户手动发起 | skill 自动触发 |
| 计划粒度 | 由模型自由规划 | 强制每步 2-5 分钟 |
| TDD 集成 | 不强制 | 强制 |
| 代码审查 | 无内置 | 两阶段 review |
| 适用场景 | 快速规划、轻量任务 | 严格工程化、复杂项目 |
| 学习成本 | 低 | 中等 |

### 6.4 怎么选

- 不是二选一，而是互补
- 轻量任务用 CodeBuddy Plan，复杂项目叠加 Superpowers 的纪律
- 可以在 CodeBuddy Plan 的基础上，手动引入 Superpowers 的核心 skill（TDD + verification）

---

## 七、源码中的设计哲学："human partner" 而非 "user"

- Superpowers 源码中刻意使用 "human partner" 而非 "user"
- 这不是文字游戏——"partner" 暗示协作关系，"user" 暗示服务关系
- 对 Agent 行为的影响：partner 关系下，Agent 更倾向于"讨论后再做"而非"接到指令就执行"
- 延伸思考：skill 中的用语选择本身就是 prompt engineering 的一部分

---

## 八、落地建议与局限

### 8.1 从哪开始

- 最小起步：只启用 `test-driven-development` + `verification-before-completion`
- 进阶：加上 `brainstorming` + `writing-plans`
- 完整工作流：全部启用，配合 OpenSpec 和 Harness

### 8.2 局限与注意事项

- 所有约束都是 prompt 级的，不是代码级门禁——模型可以违反（对比 Harness 的硬约束）
- 全部 skill 同时加载会导致上下文过长，模型表现反而下降
- Skill 之间的依赖关系没有运行时强制——理论上 Agent 可以跳过 brainstorming 直接写代码
- v5.0.7 版本分析，后续版本可能有变化

---

## 结论

- 一句话收尾：**Superpowers 的核心不是 14 个 skill，而是那套"铁律 + 反合理化 + 红旗"的行为塑造方法论——它证明了纯文本 prompt 也能构建出有纪律的开发流程，只要你懂得怎么写。**
- 回扣主题：源码分析的价值不在于"知道它有什么"，而在于"理解它为什么这么设计"——这些设计手法可以迁移到任何 Agent 系统中

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*

---

## 大纲元信息

- **目标字数**：6000-8000 字
- **目标读者**：有一定基础的开发者、技术人员或 AI 工程实践者
- **与已有文章的差异化**：
  - 《Superpowers 是什么》：概览介绍 → 本文从源码拆机制
  - 《三层拼图》：三者关系 → 本文深入单个（Superpowers）的内部
  - 《实战配合》：OpenSpec+Superpowers 联合实战 → 本文聚焦 Superpowers 自身的源码分析
  - 《与 Harness 的区别》：两者对比 → 本文不以对比为主线，对比只是辅助定位
- **风格基线**：沿用场景驱动 + 表格对比 + 直白口语化的写法，保持"先给判断"的阅读友好结构
