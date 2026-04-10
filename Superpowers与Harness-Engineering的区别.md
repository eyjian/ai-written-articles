# Superpowers 和 Harness Engineering，到底差在哪：一个偏工作流，一个偏控制面

> 如果你眼下最想解决的是"AI 写代码总爱抢跑、漏测、急着交差"，先看 `Superpowers`；如果你更在意 Agent 在长期、多步骤任务里的可控性，就得把视角扩展到 `Harness Engineering`。

很多人第一次听到 `Harness Engineering`，都会顺手把它和 `Superpowers` 放到一类：都在给 AI 加流程、加约束、加反馈，看上去像在说同一件事。

这种联想不算错，但还不够准。[`Superpowers`](https://github.com/obra/superpowers) 是 Jesse Vincent（obra）创建的开源项目，是一套基于可组合 Skills 的 AI 编码工作流框架；`Harness Engineering` 更像一种围绕 Agent 设计运行环境、约束机制和纠偏系统的工程视角。

两者谈的是同一个问题域，只是不是同一个抽象层级。后面就把这件事拆开说清。

## 先给判断

赶时间的话，先记这三句：

- **`Superpowers`**：更接近 AI 编码场景里的工作流实践，重点是把需求澄清、计划拆解、TDD、调试、审查、验证这些环节补齐。
- **`Harness Engineering`**：更接近围绕 Agent 构建控制面、反馈机制和运行环境的一种工程方法。
- **两者关系**：可以借 `Harness Engineering` 的视角来理解 `Superpowers` 的价值，但没必要把两者画成严格的一一对应关系。

说得再直白一点：**`Superpowers` 更关心"这次编码任务怎么少翻车"，`Harness Engineering` 更关心"这类 Agent 怎么长期不失控"。**

## `Harness Engineering` 这个词，原始语境是什么

目前公开讨论里，一个重要源头是 Mitchell Hashimoto 在 2026 年 2 月 5 日发表的《My AI Adoption Journey》。他当时写得很克制：行业里未必已经有一个被广泛接受的正式术语，但他越来越习惯把这类工作叫作 `harness engineering`。

他想说的重点其实是：**只要你发现 Agent 会反复犯某类错，就别只补一次 prompt，而要把解决方案工程化，让它以后尽量别再犯同样的错。** 这可能表现为维护 `AGENTS.md`、补验证脚本、加测试、加工具，或者把高风险步骤改造成可重复执行的流程。

到了 2026 年 4 月 2 日，Birgitta Böckeler 在 Martin Fowler 网站发表《Harness engineering for coding agent users》，把这套思路整理得更系统，也提出了一个很有启发的观察角度：**要建立对 coding agent 的信任，不能只靠 prompt，还要靠前馈式引导和反馈式传感。**

Birgitta 在文中也明确提出了一个简洁的表述：

> **Agent = Model + Harness**

她的原话是："The term harness has emerged as a shorthand to mean everything in an AI agent except the model itself." Mitchell 原文并没有直接写出这个公式，但 Birgitta 在 Martin Fowler 网站给出了这一正式概括，LangChain 等社区也广泛采用。

## 两者最核心的差别，不在"像不像"，而在"抽象层级"

先看一张总表：

| 维度 | `Superpowers` | `Harness Engineering` |
|------|---------------|------------------------|
| **抽象层级** | 更接近可直接采用的工作流实践 | 更接近上层工程视角与控制思路 |
| **主要对象** | AI 编码过程 | Agent 的整体运行方式 |
| **核心问题** | 怎样让 AI 写代码别跳步骤 | 怎样让 Agent 更可靠、可控、可校正 |
| **典型手段** | 需求澄清、计划拆解、TDD、调试、审查、验证 | 上下文设计、约束机制、引导规则、反馈传感、工具化补强 |
| **更适合谁** | 直接在用 AI 写代码的开发者 | 需要设计 Agent 控制体系的人 |
| **来源与成熟度** | 有明确的开源项目和创建者（obra/superpowers） | 社区共识概念，无单一项目承载，由多篇文章和实践共同定义 |

真正容易让人混淆的地方在于：**它们讨论的是同一个问题域，但切口不一样。** 要看清差别，需要换一把尺子。

举两个例子，让差别更具体：

**`Superpowers` 的 Skill 具体长什么样？** 比如它的 `tdd` Skill 会强制 AI 先写测试再写实现，不跑通测试就不让进入下一步；`debug` Skill 会要求 AI 在报错时先复现问题、定位根因，而不是猜着改。这些 Skill 像一套嵌入编码过程的检查清单，每一步都逼 AI 按规矩来。

**`Harness Engineering` 落地时第一步该做什么？** 最常见的起手式是写一份 `AGENTS.md`——把项目的技术栈、目录结构、编码规范、禁止操作等信息放进去，让 Agent 每次开工前都先读这份文件。再进一步就是补验证脚本：Agent 每改完一个文件，自动跑 lint、跑测试、跑类型检查，不通过就打回重做。

## 用一把更顺手的尺子看：前馈、反馈、长期治理

如果借 Birgitta Böckeler 那篇文章里的思路来看，可以把这类控制机制先粗略分成两部分：

- **Feedforward Guides**：Agent 动手之前，先给它方向、边界和约束
- **Feedback Sensors**：Agent 动手之后，用各种机制检查它有没有偏航

如果再往长期一点看，还会多出第三层问题：**系统跑久以后，文档漂不漂、技术债积不积、规则会不会失效。** 这已经更接近治理层，而不只是单次任务里的引导或纠偏。（注：Birgitta 原文主要聚焦前馈和反馈两层，"长期治理"是笔者基于工程实践的延伸。）

用这把尺子回头看 `Superpowers`，会比较顺：

| 观察维度 | `Harness Engineering` 更关心什么 | `Superpowers` 的典型表现 |
|---------|----------------------------------|---------------------------|
| **前馈** | 让 Agent 开工前就拿到足够背景、边界和规则 | 有覆盖，比如需求澄清、计划拆解、把约束提前说透 |
| **反馈** | 在执行过程中尽快发现错误并拉回正轨 | 覆盖很强，这恰好是 `Superpowers` 最有价值的部分 |
| **长期治理** | 处理文档漂移、技术债、系统性偏差 | 通常不是它的核心强项，往往需要额外机制补足 |

这样回头看，为什么很多人会觉得两者"很像"就很好解释了：`Superpowers` 的确抓住了 `Harness` 视角里最有感知的一段，尤其是**前馈 + 强反馈回路**。

所以别把它简单写成 `Superpowers = Harness Engineering`。更准确的说法是：**`Superpowers` 把一部分前馈机制和一条很强的反馈回路，收进了 AI 编码场景里一套可直接上手的工作流。**

这也是为什么很多人第一次用 `Superpowers`，会明显感觉 AI 没那么"毛躁"了：不是模型突然更聪明，而是**它被放进了一条更讲纪律的工作流里。**

## 那两者到底该怎么选

真到落地阶段，通常不是二选一，而是先看你眼前到底是哪一层问题。

| 你的角色 | 通常更适合先抓什么 |
|---------|--------------------|
| **日常用 AI 写业务代码的开发者** | 先抓 `Superpowers` 这类工作流纪律，收益往往最直接 |
| **团队负责人** | 先用 `Superpowers` 管住编码流程，再补团队级规范和上下文文档 |
| **做 Agent 平台、基础设施或长期治理的人** | 更需要上升到 `Harness Engineering` 的视角去设计完整控制体系 |

如果你现在最急的是把 AI 写代码的跑偏率降下来，先看 `Superpowers`，回报通常最快。

如果你在搭的是会持续运行、会调工具、会跨多步执行任务的 Agent 系统，`Superpowers` 只够解决局部执行纪律，还是得把视角扩展到 `Harness Engineering`。

## 结论

把话收回来，其实就一句：`Superpowers` 和 `Harness Engineering` 确实有关，但不是一个层级的概念。

`Superpowers` 更像拿来就能用的 AI 编码工作流，重点是把一次任务里的跑偏、漏测和抢跑压下来；`Harness Engineering` 更像更大的工程视角，关心的是 Agent 的上下文、约束、反馈，以及它在长期运行里的稳定性。

所以更实用的理解不是"谁替代谁"，而是：**你可以先用 `Superpowers` 把当下的编码协作管住，再用 `Harness Engineering` 的视角去看哪些控制机制还没补齐。** 换个角度说，`Superpowers` 本身就可以被看作 `Harness` 在 AI 编码场景里的一种具体实现——它覆盖了前馈和反馈回路，但不等于 `Harness` 的全部。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
