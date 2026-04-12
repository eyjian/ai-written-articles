# 基于源码深度分析 Superpowers：从行为塑造到工作流编排

> 大多数人知道 Superpowers 是"给 AI 加纪律"，但纪律是怎么加上去的？一条 Markdown 写的 Iron Law，凭什么能让 Agent 真的不跳步骤？本文基于 Superpowers v5.0.7 源码，拆解它内部的行为塑造机制、技能编排逻辑和多平台适配实现，并附带与 OpenSpec、CodeBuddy Plan 的结构性对比。

Superpowers 的介绍文章已经有不少——概览类的讲它有什么，关系类的讲它和 OpenSpec、Harness 怎么配合。但一个问题始终没人展开：**源码里到底藏了哪些"让 Agent 听话"的具体手段？**

不是"加了 TDD 技能"这种一句话概括，而是具体到：铁律写成什么样、为什么用全大写、防御表堵了哪些借口、红旗表卡的是什么层级的念头。这些细节决定了 Superpowers 到底是"又一个 prompt 模板"，还是一套经过实战打磨的行为塑造系统。

本文不重复已有文章的概览内容，聚焦源码层面的机制分析。

## 一、先给判断

1. **Superpowers 的 skill 不是文档，是行为塑造代码。** 它用 Iron Laws、Rationalization Prevention Table、Red Flags Table 三张"心理学武器"逼 Agent 遵守纪律。
2. **14 个 skill 不是平铺的，是有依赖关系和工作流编排的。** using-superpowers 是入口，brainstorming → writing-plans → TDD → review → verification 是主链路。
3. **多平台适配靠的是 `hooks/session-start` 这个 Bash 脚本，在 session 启动时把 using-superpowers 的内容注入 Agent 上下文——零代码依赖，纯文本注入。**

## 二、行为塑造的三板斧：Iron Laws、Rationalization Prevention、Red Flags

本节是全文核心。Superpowers 如何用纯文本"控制" Agent 行为？答案是三层递进的行为塑造机制。

### 2.1 Iron Laws（铁律）

从源码里拆出三条核心铁律的原文：

**TDD skill 的铁律：**

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
```

附带的执行规则毫不留情——"Write code before the test? Delete it. Start over."，逐条封死所有退路：不准保留作为"参考"、不准"改编"、不准看一眼，删就是删。

**systematic-debugging skill 的铁律：**

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

四阶段调查流程（根因调查 → 模式分析 → 假设验证 → 实现修复）没走完，就不允许提出修复方案。

**verification-before-completion skill 的铁律：**

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

当前消息中没有执行验证命令并看到输出，就不能声称任务完成。源码里直白地写道："Claiming work is complete without verification is dishonesty, not efficiency."——未验证就宣称完成，定性为"不诚实"。

三条铁律的写法有几个共同特点：

- **全大写**——这不是排版偏好，是 prompt engineering 层面的强调手段。全大写在 LLM 的注意力机制中确实能获得更高的权重。
- **祈使句、绝对化表述**——"NO...WITHOUT...FIRST" 的句式不留解释空间。
- **配套封堵条款**——每条铁律下面都跟着一串"No exceptions"清单，把常见的绕行路径逐一堵死。

这不是写给人看的文档，是写给 LLM 的 prompt engineering。

### 2.2 Rationalization Prevention Table（理性化防御表）

铁律定了底线，但 LLM 有一个特性：它不是不知道规则，而是会给自己找理由绕开规则。Superpowers 的应对方式是在源码中构建"理性化防御表"，提前堵死常见的逃避路径。

以 TDD skill 的 Common Rationalizations 表为例，直接从源码摘录：

| Excuse | Reality |
|--------|---------|
| "Too simple to test" | Simple code breaks. Test takes 30 seconds. |
| "I'll test after" | Tests passing immediately prove nothing. |
| "Tests after achieve same goals" | Tests-after = "what does this do?" Tests-first = "what should this do?" |
| "Deleting X hours is wasteful" | Sunk cost fallacy. Keeping unverified code is technical debt. |
| "Keep as reference, write tests first" | You'll adapt it. That's testing after. Delete means delete. |
| "TDD will slow me down" | TDD faster than debugging. Pragmatic = test-first. |

verification-before-completion skill 也有类似的表：

| Excuse | Reality |
|--------|---------|
| "Should work now" | RUN the verification |
| "I'm confident" | Confidence ≠ evidence |
| "Just this once" | No exceptions |
| "Agent said success" | Verify independently |
| "Partial check is enough" | Partial proves nothing |

表的结构很统一：左列是"Agent 会找的借口"，右列是"为什么这个借口不成立"。

这套机制的核心洞察是：**模型不是不知道规则，而是会合理化自己的违规行为。** 防御表通过预先枚举违规理由并逐条反驳，压缩了 LLM 的"创造性逃避"空间。writing-skills 源码里把这个思路说得更明确——"Agents are smart and will find loopholes when under pressure."

### 2.3 Red Flags Table（红旗表）

防御表堵的是借口，红旗表卡的是更底层的东西——**念头**。

using-superpowers 源码中的 Red Flags 表：

| Thought | Reality |
|---------|---------|
| "This is just a simple question" | Questions are tasks. Check for skills. |
| "I need more context first" | Skill check comes BEFORE clarifying questions. |
| "Let me explore the codebase first" | Skills tell you HOW to explore. Check first. |
| "This doesn't need a formal skill" | If a skill exists, use it. |
| "I remember this skill" | Skills evolve. Read current version. |
| "The skill is overkill" | Simple things become complex. Use it. |
| "I'll just do this one thing first" | Check BEFORE doing anything. |
| "This feels productive" | Undisciplined action wastes time. Skills prevent this. |

注意左列不是"借口"，而是"念头"。红旗表的设计逻辑是元认知层面的约束——不是告诉 Agent "不能做什么"，而是告诉它"如果在想跳过某个步骤，这个想法本身就是危险信号"。

TDD skill 和 systematic-debugging skill 也有各自的 Red Flags 清单。TDD 的版本列出了 13 种红旗信号，统一指向一个结论："All of these mean: Delete code. Start over with TDD." debugging 的版本有 11 种，统一指向 "STOP. Return to Phase 1."

### 2.4 三板斧的协同：为什么单用一个不够

三层机制各管一层，缺一不可：

| 机制 | 作用层次 | 类比 |
|------|---------|------|
| Iron Laws（铁律） | 定底线——什么绝对不能做 | 法律条文 |
| Rationalization Prevention（防御表） | 堵借口——常见的违规理由逐条反驳 | 反诡辩手册 |
| Red Flags（红旗表） | 卡念头——想跳过就是危险信号 | 自我审查清单 |

只有铁律没有防御表，Agent 会说"这次情况特殊"绕过去。只有防御表没有红旗表，Agent 会在念头层面就把"需要检查 skill"这个步骤跳过。三板斧组合起来，从行为底线到思维模式全覆盖。

每个 skill 源码开头都有一句元规则："Violating the letter of the rules is violating the spirit of the rules."——直接切断了"遵守精神但灵活变通"这条最常见的逃逸路径。

## 三、14 个 Skill 的依赖关系与工作流编排

### 3.1 Skill 分类总览

从源码的 `skills/` 目录和 `README.md` 中提取的完整分类：

| 类别 | Skill | 类型 | 触发场景 |
|------|-------|------|---------|
| **元技能** | using-superpowers | Rigid | 每次会话启动 |
| **元技能** | writing-skills | Rigid | 创建/编辑 skill |
| **协作流程** | brainstorming | Flexible | 任何创造性工作之前 |
| **协作流程** | writing-plans | Flexible | 有 spec 或需求时 |
| **协作流程** | executing-plans | Flexible | 在独立 session 执行计划 |
| **协作流程** | subagent-driven-development | Flexible | 当前 session 派发子代理执行 |
| **协作流程** | dispatching-parallel-agents | Flexible | 2+ 个独立任务可并行 |
| **开发纪律** | test-driven-development | Rigid | 实现任何功能或修复 |
| **开发纪律** | systematic-debugging | Rigid | 遇到 bug 或异常行为 |
| **开发纪律** | verification-before-completion | Rigid | 声称完成之前 |
| **代码审查** | requesting-code-review | Flexible | 完成任务后、合并前 |
| **代码审查** | receiving-code-review | Flexible | 收到审查反馈时 |
| **分支管理** | using-git-worktrees | Flexible | 需要隔离工作区 |
| **分支管理** | finishing-a-development-branch | Flexible | 实现完成后 |

Rigid 和 Flexible 的区分来自 using-superpowers 源码："**Rigid** (TDD, debugging): Follow exactly. Don't adapt away discipline. **Flexible** (patterns): Adapt principles to context."

### 3.2 主工作流链路：从 brainstorming 到 finishing

从源码中各 skill 的触发关系和交叉引用，可以还原出完整的主链路：

```
using-superpowers（入口，每次会话自动加载）
  → brainstorming（设计先行，HARD-GATE 禁止跳过）
    → writing-plans（写实现计划，每步 2-5 分钟）
      → [executing-plans | subagent-driven-development]（执行）
        → test-driven-development（TDD 实现每个任务）
          → systematic-debugging（遇到问题时）
        → requesting-code-review（每个任务完成后请求审查）
          → receiving-code-review（处理审查反馈）
        → verification-before-completion（声称完成前验证）
      → finishing-a-development-branch（收尾：合并/PR/保留/丢弃）
```

这条链路不是人为画出来的流程图，而是从源码的交叉引用中提取的。几个关键的跨 skill 引用：

- brainstorming 源码中有 `<HARD-GATE>` 标签："Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it."
- brainstorming 结束后强制调用 writing-plans："**The terminal state is invoking writing-plans.** Do NOT invoke frontend-design, mcp-builder, or any other implementation skill."
- writing-plans 结束后提供两条执行路径的选择，并标注"**REQUIRED SUB-SKILL**"。
- subagent-driven-development 中对 TDD 的引用："**Subagents should use:** superpowers:test-driven-development"
- executing-plans 和 subagent-driven-development 结束后都要求调用 finishing-a-development-branch。

### 3.3 两条执行路径：executing-plans vs subagent-driven-development

writing-plans 完成后，源码提供两条执行路径供选择。从源码对比差异：

**executing-plans** 的定位是"在独立 session 内联执行"。源码开头就建议使用另一条路径："Tell your human partner that Superpowers works much better with access to subagents."流程相对简单：加载计划 → 批量执行任务 → 调用 finishing-a-development-branch。

**subagent-driven-development** 的定位是"每个任务派发新的子代理"。核心机制是"Fresh subagent per task + two-stage review (spec then quality)"。每个任务经过：实现子代理 → spec 合规审查 → 代码质量审查，三个角色各有独立的 prompt 模板。

还有一个 **dispatching-parallel-agents**，定位更窄——专门处理"2+ 个独立任务可并行"的场景。源码强调"Dispatch one agent per independent problem domain. Let them work concurrently."

三者的结构性对比：

| 维度 | executing-plans | subagent-driven-development | dispatching-parallel-agents |
|------|----------------|---------------------------|---------------------------|
| 执行方式 | 当前 session 内联 | 每任务派发子代理 | 多代理并行 |
| 审查机制 | 无独立审查 | 两阶段审查（spec + quality） | 返回后统一审查 |
| 上下文隔离 | 共享 session 上下文 | 每个子代理独立上下文 | 每个代理独立上下文 |
| 适用场景 | 简单计划、无子代理支持时 | 标准开发流程 | 独立问题并行修复 |
| 成本 | 低 | 高（实现+2 审查/任务） | 中等 |

### 3.4 元技能：using-superpowers 和 writing-skills

**using-superpowers** 是整个系统的引导入口。从源码看，它的核心职责是两件事：

第一，**建立 skill 路由规则**——"Invoke relevant or requested skills BEFORE any response or action. Even a 1% chance a skill might apply means that you should invoke the skill to check."这条规则通过 Red Flags 表强制执行。

第二，**定义优先级体系**——源码中明确了三层优先级："1. User's explicit instructions (CLAUDE.md, GEMINI.md, AGENTS.md, direct requests) — highest priority. 2. Superpowers skills — override default system behavior. 3. Default system prompt — lowest priority." 用户指令永远优先于 Superpowers。

**writing-skills** 是"写 skill 的 skill"——一个元技能。它把 TDD 的思路迁移到了文档编写："**Writing skills IS Test-Driven Development applied to process documentation.**" 先写压力测试场景让 Agent 在没有 skill 的情况下跑一遍（RED），观察它怎么违规；然后写 skill 针对性地堵住违规路径（GREEN）；最后通过反复测试发现新的逃逸路径并封堵（REFACTOR）。

writing-skills 中还有一个细节：YAML frontmatter 的 `description` 字段要求遵循 agentskills.io 的 OpenSpec 规范，源码中明确警告——description 里不能包含 skill 的工作流摘要，否则 Agent 会走捷径直接按 description 执行，而跳过完整的 skill 内容。这是实测发现的问题，不是理论推导。

## 四、多平台适配机制：hooks/session-start 引导注入

### 4.1 引导机制的源码分析

`hooks/session-start` 是一个 57 行的 Bash 脚本，做的事就一件：**在 session 启动时读取 using-superpowers 的 Markdown 内容，包裹进 `<EXTREMELY_IMPORTANT>` 标签后注入 Agent 的初始上下文。**

核心逻辑：

```bash
# 读取 using-superpowers 内容
using_superpowers_content=$(cat "${PLUGIN_ROOT}/skills/using-superpowers/SKILL.md")

# 构造注入内容
session_context="<EXTREMELY_IMPORTANT>\nYou have superpowers.\n\n
**Below is the full content of your 'superpowers:using-superpowers' skill...**\n\n
${using_superpowers_escaped}\n\n
</EXTREMELY_IMPORTANT>"
```

然后根据平台环境变量选择不同的 JSON 输出格式。整个过程零代码依赖——不需要 npm、pip、任何 runtime，只需要 bash。

### 4.2 六个平台的适配方式

从源码中可以还原出六个平台的完整适配方案：

| 平台 | 适配文件 | 注入方式 | 环境检测 |
|------|---------|---------|---------|
| **Claude Code** | `.claude-plugin/plugin.json` + `hooks/hooks.json` | SessionStart hook → `hookSpecificOutput.additionalContext` | `CLAUDE_PLUGIN_ROOT` 环境变量 |
| **Cursor** | `.cursor-plugin/plugin.json` + `hooks/hooks-cursor.json` | sessionStart hook → `additional_context`（snake_case） | `CURSOR_PLUGIN_ROOT` 环境变量 |
| **Copilot CLI** | 通过 marketplace 安装 | SessionStart hook → `additionalContext`（SDK 标准格式） | `COPILOT_CLI` 环境变量 |
| **Gemini CLI** | `gemini-extension.json` + `GEMINI.md` | `contextFileName` 指向 `GEMINI.md`，内含 `@` 引用 | 无需检测，直接加载 |
| **OpenCode** | `.opencode/plugins/superpowers.js` | JS 插件，注入首条用户消息 | 无需检测，插件系统处理 |
| **Codex** | `.codex/INSTALL.md` | 手动安装，自动发现 skills 目录 | 无需检测 |

关键差异在于 Claude Code 和 Cursor 虽然底层接近，但 JSON 格式不同。`session-start` 脚本里有一段精心设计的平台检测逻辑：

```bash
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  # Cursor 用 snake_case
  printf '{"additional_context": "%s"}' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ]; then
  # Claude Code 用嵌套结构
  printf '{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": "%s"}}' "$session_context"
else
  # Copilot CLI 和其他平台用 SDK 标准格式
  printf '{"additionalContext": "%s"}' "$session_context"
fi
```

源码注释解释了为什么要区分："Claude Code reads BOTH additional_context and hookSpecificOutput without deduplication, so we must emit only the field the current platform consumes."——如果两个都输出，Claude Code 会注入两份相同内容。

Gemini CLI 的适配走了一条不同的路：不用 hook 脚本，而是通过 `gemini-extension.json` 中的 `contextFileName` 指向 `GEMINI.md`。`GEMINI.md` 的内容极简——只有两行 `@` 引用，让 Gemini 在启动时直接加载 using-superpowers 的内容和工具映射表。

OpenCode 的适配最重型——一个 112 行的 JS 插件。它通过 `experimental.chat.messages.transform` 钩子，在首条用户消息前插入 bootstrap 内容。源码注释说明了选择用户消息而非系统消息的原因："Token bloat from system messages repeated every turn" 和 "Multiple system messages breaking Qwen and other models"。

### 4.3 为什么选择"文本注入"而不是"API 集成"

所有平台的适配方式都有一个共同点：**最终注入 Agent 的都是纯文本。** 不是 API 调用，不是工具注册，不是运行时钩子——就是一段 Markdown 被塞进了上下文窗口。

这是一个有意识的设计选择。纯文本 = 零依赖 = 跨平台。换任何一个支持上下文注入的 AI 编程工具，只要能在 session 启动时往上下文里塞一段文本，Superpowers 就能工作。

代价也很明确：**没有运行时强制力。** 所有的铁律、防御表、红旗表都是 prompt 级约束，不是代码级门禁。Agent 在技术上完全可以无视这些规则。和 Harness Engineering 的硬约束对比——Harness 可以做到"调研阶段拿不到写文件的工具"，Superpowers 只能做到"告诉 Agent 在调研阶段不应该写文件"。

Superpowers 本质上是**高水平的 prompt engineering**，不是代码级的执行管控。

## 五、与 OpenSpec 的关系：遵循者 vs 定义者

### 5.1 OpenSpec 定义了什么

OpenSpec 是一个 skill 的格式标准，来源于 agentskills.io/specification。它定义的是 skill 文件的结构规范：YAML frontmatter 应该有哪些字段、目录怎么组织、怎么分发。

### 5.2 Superpowers 如何遵循 OpenSpec

从 Superpowers 的 skill 文件结构可以看到 OpenSpec 规范的落地痕迹。每个 skill 的 `SKILL.md` 都以标准的 YAML frontmatter 开头：

```yaml
---
name: test-driven-development
description: Use when implementing any feature or bugfix, before writing implementation code
---
```

writing-skills 源码中对 OpenSpec 的引用是直接的："Two required fields: `name` and `description` (see agentskills.io/specification for all supported fields)"。并且对 description 字段给出了严格要求：以 "Use when..." 开头、只描述触发条件不描述工作流、第三人称、500 字符以内。

目录结构也遵循 OpenSpec 的扁平命名空间要求：

```
skills/
  skill-name/
    SKILL.md              # 主文件（必需）
    supporting-file.*     # 辅助文件（按需）
```

### 5.3 一句话关系

OpenSpec 管"skill 长什么样"——格式标准。Superpowers 管"skill 装什么内容"——具体实现。类比：OpenSpec 是 HTML 规范，Superpowers 是按规范写的一组网页。

## 六、与 CodeBuddy Plan 的对比：平台能力 vs 插件能力

### 6.1 CodeBuddy Plan 是什么

CodeBuddy Plan 是 IDE 内置的规划功能。发起后 Agent 先输出实现计划再执行，平台级能力，开箱即用，不需要安装任何插件。

### 6.2 Superpowers 的计划系统

Superpowers 的计划系统由 writing-plans + executing-plans / subagent-driven-development 三个 skill 组成。

从 writing-plans 源码看计划的粒度要求——每步是一个 2-5 分钟的原子操作："Write the failing test" 是一步，"Run it to make sure it fails" 是一步，"Implement the minimal code" 是一步。计划文档要求"no placeholders"，每步都必须包含完整代码。

执行阶段，subagent-driven-development 强制两阶段审查：先检查 spec 合规性，再检查代码质量。spec 审查不通过，代码质量审查不会启动——"**Start code quality review before spec compliance is ✅** (wrong order)" 被列为 Red Flag。

### 6.3 结构性对比

| 维度 | CodeBuddy Plan | Superpowers Plan |
|------|----------------|------------------|
| 性质 | 平台内置能力 | 插件/skill 能力 |
| 启动方式 | 用户手动发起 | skill 自动触发 |
| 计划粒度 | 由模型自由规划 | 强制每步 2-5 分钟 |
| TDD 集成 | 不强制 | 强制 |
| 代码审查 | 无内置 | 两阶段 review（spec + quality） |
| 适用场景 | 快速规划、轻量任务 | 严格工程化、复杂项目 |
| 学习成本 | 低 | 中等 |

### 6.4 怎么选

不是二选一，而是互补。轻量任务用 CodeBuddy Plan 足够——快速出计划、快速执行。复杂项目需要纪律保障时，叠加 Superpowers 的核心 skill。最小组合是 `test-driven-development` + `verification-before-completion`，这两个投入产出比最高。

进阶组合加上 `brainstorming` + `writing-plans`，把设计先行和计划细化也管起来。完整工作流则全部启用，跑满 brainstorming → writing-plans → subagent-driven-development → finishing 的完整链路。

## 七、源码中的设计哲学："human partner" 而非 "user"

Superpowers 源码中有一个反复出现的用语选择：**"your human partner"，而不是 "the user"。**

TDD skill："Exceptions (ask your human partner)"。systematic-debugging："Discuss with your human partner before attempting more fixes"。receiving-code-review："your human partner's rule: External feedback - be skeptical, but check carefully"。CLAUDE.md 里更是直接警告——"your human partner" is deliberate, not interchangeable with "the user"。

这不是文字游戏。"user" 暗示服务关系——用户发指令，Agent 执行。"partner" 暗示协作关系——双方有共同目标，Agent 有义务在 partner 的决策可能出问题时提出异议。

对 Agent 行为的实际影响：partner 关系下，TDD skill 可以写"ask your human partner"让 Agent 主动发起讨论，而非默默执行一个可能有问题的指令。receiving-code-review 可以写"If conflicts with your human partner's prior decisions: Stop and discuss"——换成 user 模式这句话不太自然。

CLAUDE.md 中进一步印证了这种设计哲学——94% 的 PR 拒绝率背景下，对 Agent 的要求是"protect your human partner from that outcome"。Agent 不是执行工具，是有义务保护合作伙伴免于低质量输出的搭档。

用语选择本身就是 prompt engineering 的一部分。

## 八、落地建议与局限

### 8.1 从哪开始

- **最小起步：** 只启用 `test-driven-development` + `verification-before-completion`。这两个 skill 覆盖了最高频的纪律问题——不写测试和虚报完成。
- **进阶：** 加上 `brainstorming` + `writing-plans`。设计先行 + 计划细化，减少返工。
- **完整工作流：** 全部启用，配合 OpenSpec 定需求，配合 Harness 管协作。

### 8.2 局限与注意事项

**所有约束都是 prompt 级的，不是代码级门禁。** 模型可以违反铁律——它只是被强烈建议不要这么做。和 Harness 的硬约束相比，Superpowers 靠的是"说服力"而非"执行力"。

**全部 skill 同时加载会导致上下文过长。** README 和已有文章都提到过这个问题——上下文塞满了，模型的注意力被稀释，表现反而下降。按需加载是基本原则。

**Skill 之间的依赖关系没有运行时强制。** 理论上 Agent 可以跳过 brainstorming 直接写代码，也可以不跑 TDD 直接提交。链路上的每一步都是"skill 告诉 Agent 下一步该做什么"，而非"系统不允许 Agent 跳到其他步骤"。

**本文基于 v5.0.7 版本分析。** Superpowers 迭代活跃，后续版本的 skill 内容和平台适配方式可能有变化。

## 结论

**Superpowers 的核心不是 14 个 skill，而是那套"铁律 + 反合理化 + 红旗"的行为塑造方法论。** 它证明了纯文本 prompt 也能构建出有纪律的开发流程，只要设计者足够了解 LLM 的行为特征——知道它会在哪里偷懒、会找什么借口、会在什么念头层面开始走偏。

源码分析的价值不在于"知道它有什么"，而在于"理解它为什么这么设计"。全大写铁律、逐条反驳的防御表、元认知层面的红旗表、"human partner" 的刻意用语——这些设计手法可以迁移到任何 Agent 系统中。不一定要用 Superpowers，但它的行为塑造思路值得借鉴。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
