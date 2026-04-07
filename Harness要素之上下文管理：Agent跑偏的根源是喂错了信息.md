# Harness 要素之上下文管理：Agent 跑偏的根源是喂错了信息

> Agent 跑偏，很多时候不是因为它笨，而是因为它看到的信息不对。要么塞了太多无关内容导致注意力被稀释，要么关键信息压根没给到，要么长任务跑到后半段上下文已经脏得不可收拾。上下文管理要解决的，就是让 Agent 在正确的时间看到正确的信息。

## 核心判断

一句话分清：

- **塞满上下文 = 把整个图书馆搬到桌上让人写论文。** 信息是全了，但找不到重点，注意力被稀释。
- **按需检索 = 给一张书架索引卡，需要什么自己去取。** 桌面干净，每次只看当前任务需要的信息。

上下文窗口是 Agent 的工作记忆，跟人的工作台一样——不是越大越好，而是越干净越高效。

## 上下文管理不当会出什么事

| 问题 | 表现 | 根因 |
|------|------|------|
| 上下文过载 | Agent 回复越来越草率，开始忽略细节 | 窗口接近上限，模型在压力下走捷径 |
| 关键信息遗失 | Agent 忘了 10 轮对话前给的架构约束 | 旧信息被新信息挤出了有效注意范围 |
| 信息污染 | Agent 把 A 任务的上下文混进 B 任务 | 多任务共享同一个会话 |
| 提前停止 | 长任务跑到后半段，Agent 突然宣布"已完成" | 上下文焦虑——模型感知到窗口快满了，仓促收尾 |
| 幻觉增加 | Agent 开始编造不存在的函数名和 API | 正确信息被噪音淹没，模型靠猜 |

这些问题都不是模型能力不足——同一个模型，上下文管理做好了，表现就稳定；管不好，就一塌糊涂。

## 上下文管理在 Harness 架构中的位置

约束管"能做什么"，沙箱管"在哪里做"，上下文管理管"看到什么"。三者分别控制 Agent 的行为边界、执行环境和信息环境。

```text
┌─────────────────────────────────────────┐
│            Harness 约束层               │
├─────────────────────────────────────────┤
│            沙箱隔离层                    │
├─────────────────────────────────────────┤
│         上下文管理（信息环境边界）          │
│  AGENTS.md │ 按需检索 │ 结构化交接 │ 上下文重置 │
├─────────────────────────────────────────┤
│       反馈回路 │ 可观测性                 │
├─────────────────────────────────────────┤
│          Agent（执行层）                │
└─────────────────────────────────────────┘
```

## 上下文管理的四种落地形态

### 1. AGENTS.md：给 Agent 一份精准的项目说明书

Agent 每次启动新会话，对你的项目几乎一无所知。如果不主动告诉它技术栈、规范、禁区、验证方式，它就会按自己的"默认世界观"来做事。

`AGENTS.md` 就是专门给 Agent 看的项目说明书。但关键不是写得多，而是写得准。

```markdown
# AGENTS.md

## 项目概览
Next.js 14 全栈项目，App Router + TypeScript 严格模式。

## 技术选型（不可更改）
- 框架：Next.js 14（App Router，禁止 Pages Router）
- 样式：Tailwind CSS（禁止 CSS Modules）
- ORM：Prisma + PostgreSQL
- 测试：Vitest + Testing Library

## 目录结构
- app/：页面和 API 路由
- lib/：业务逻辑和工具函数
- components/：UI 组件
- prisma/：数据库 schema

## 禁区
- .env、secrets/ → 绝对不要读取或修改
- prisma/migrations/ → 不要手动编辑迁移文件
- package-lock.json → 不要手动修改

## 验证命令（改完代码必须执行）
1. pnpm typecheck
2. pnpm lint
3. pnpm test
```

一个关键原则：**AGENTS.md 只放最关键的规则，详细文档放到 `docs/` 目录让 Agent 按需查阅。**

如果是 monorepo，分层放：

```text
项目根目录/AGENTS.md          ← 全局规则
├── packages/web/AGENTS.md    ← Web 端专属规则
└── packages/api/AGENTS.md    ← API 端专属规则
```

Agent 在不同子目录工作时，读取到的规则更精准，不会被无关信息干扰。

### 2. 按需检索：教 Agent 自己去找信息

把所有文档一股脑塞进上下文，效果往往不如教 Agent 按需查阅。

原因很简单：上下文窗口是有限的。塞满了文档，留给任务本身的空间就少了。而且大段的背景信息会稀释模型对当前任务关键细节的注意力。

```python
# context_provider.py
"""按需上下文提供器：Agent 需要什么信息，调用工具去取"""

import json
from pathlib import Path

class ContextProvider:
    """不是一次塞满，而是让 Agent 按需检索"""

    def __init__(self, docs_dir: str = "docs"):
        self.docs_dir = Path(docs_dir)
        self.index = self._build_index()

    def _build_index(self) -> dict:
        """构建文档索引，让 Agent 知道有哪些文档可查"""
        index = {}
        for f in self.docs_dir.rglob("*.md"):
            key = f.stem.lower().replace("-", "_")
            index[key] = str(f)
        return index

    def get_available_docs(self) -> list[str]:
        """Agent 可以先调用这个工具，看看有哪些文档可查"""
        return list(self.index.keys())

    def get_doc(self, topic: str) -> str:
        """Agent 需要某个主题的信息时，调用这个工具取"""
        path = self.index.get(topic)
        if not path:
            return f"没有找到关于 {topic} 的文档。可用文档：{list(self.index.keys())}"
        return Path(path).read_text()
```

把 `get_available_docs` 和 `get_doc` 注册为 Agent 的工具。Agent 开始任务时先看索引，需要什么查什么，不用的信息不进上下文。

### 3. 结构化交接：长任务跑到一半，怎么接力

长任务里最大的问题是上下文越来越脏。会话越长，早期的重要信息越容易被"推"到模型注意力的边缘。

与其让一个会话死扛到底，不如在合适的时候做一次结构化交接——总结当前进度，启动新会话继续。

```python
# handoff.py
"""结构化交接：老会话总结，新会话接力"""

import json
from datetime import datetime

class HandoffManager:
    def generate_handoff(self, agent_session) -> dict:
        """让当前 Agent 生成一份结构化交接文档"""
        handoff = {
            "timestamp": datetime.now().isoformat(),
            "task": agent_session.original_task,
            "completed": agent_session.completed_steps,
            "current_state": {
                "modified_files": agent_session.get_modified_files(),
                "test_status": agent_session.last_test_result,
                "open_issues": agent_session.known_issues,
            },
            "next_steps": agent_session.planned_next_steps,
            "warnings": agent_session.accumulated_warnings,
        }
        return handoff

    def load_handoff(self, handoff: dict) -> str:
        """把交接文档转换成新会话的初始 prompt"""
        return f"""你正在接手一个进行中的任务。

## 原始任务
{handoff['task']}

## 已完成的步骤
{json.dumps(handoff['completed'], ensure_ascii=False, indent=2)}

## 当前状态
- 修改过的文件：{handoff['current_state']['modified_files']}
- 测试状态：{handoff['current_state']['test_status']}
- 已知问题：{handoff['current_state']['open_issues']}

## 下一步计划
{json.dumps(handoff['next_steps'], ensure_ascii=False, indent=2)}

## 注意事项
{json.dumps(handoff['warnings'], ensure_ascii=False, indent=2)}

请从下一步计划开始继续执行。"""
```

交接文档的关键是结构化——不是让 Agent 写一段"总结"，而是按固定字段输出：做了什么、当前状态、下一步、注意事项。新会话拿到这份文档，就能在一个干净的上下文里继续工作。

### 4. 上下文重置：当压缩不够用时，重新开始

上下文压缩（compaction）是一种常见做法——把长对话历史压缩成摘要。但压缩有损，压缩次数越多，丢失的细节越多。

更稳的做法是：**检测到上下文使用率超过阈值时，直接做一次完整的交接 + 重置。**

```python
# context_reset.py
"""上下文重置：不是压缩，而是交接 + 新会话"""

class ContextResetManager:
    MAX_UTILIZATION = 0.6  # 上下文使用率超过 60% 就考虑重置

    def should_reset(self, session) -> bool:
        utilization = session.token_count / session.max_tokens
        return utilization > self.MAX_UTILIZATION

    def reset(self, old_session, handoff_manager):
        # 1. 生成交接文档
        handoff = handoff_manager.generate_handoff(old_session)

        # 2. 持久化交接文档（写到文件系统，不依赖上下文记忆）
        handoff_path = f".harness/handoffs/{old_session.id}.json"
        with open(handoff_path, "w") as f:
            json.dump(handoff, f, ensure_ascii=False)

        # 3. 启动新会话
        new_session = create_fresh_session()

        # 4. 加载交接文档 + AGENTS.md
        new_session.load_context(handoff_manager.load_handoff(handoff))
        new_session.load_context(read_file("AGENTS.md"))

        return new_session
```

重置比压缩更"浪费"（需要重建上下文），但也更可靠——新会话是干净的，不会被历史对话中的噪音干扰。

Anthropic 在 2026 年 3 月的博客中提到，他们在早期（Sonnet 4.5 时期）大量使用这种上下文重置策略。随着模型能力增强（Opus 4.6），模型的长上下文检索能力和抗焦虑能力都有改善，对重置的依赖降低了。但在当前的工程实践中，对大多数团队来说，超过 60% 上下文使用率就考虑交接仍然是稳妥的做法。

## 实战操作：从零搭一个带上下文管理的任务运行器

下面用一个完整示例，把 AGENTS.md + 按需检索 + 结构化交接串起来。

### 项目结构

```text
harness-context-demo/
├── AGENTS.md                  # Agent 的项目说明书
├── docs/
│   ├── api-conventions.md     # API 规范文档
│   ├── testing-guide.md       # 测试指南
│   └── database-schema.md     # 数据库设计
├── .harness/
│   └── handoffs/              # 交接文档存放目录
├── src/
│   └── ...                    # 项目源码
└── context_runner.py          # 带上下文管理的任务运行器
```

### 任务运行器代码

```python
# context_runner.py
"""带上下文管理的任务运行器：AGENTS.md + 按需检索 + 交接重置"""

import json
from pathlib import Path
from datetime import datetime

# ---------- 上下文加载 ----------

def load_agents_md() -> str:
    """每个会话启动时，首先加载 AGENTS.md"""
    path = Path("AGENTS.md")
    if path.exists():
        return path.read_text()
    return ""

def get_available_docs() -> list[str]:
    """供 Agent 调用：查看有哪些文档可检索"""
    docs = Path("docs")
    if not docs.exists():
        return []
    return [f.stem for f in docs.glob("*.md")]

def get_doc(topic: str) -> str:
    """供 Agent 调用：按需获取指定文档"""
    path = Path("docs") / f"{topic}.md"
    if path.exists():
        return path.read_text()
    return f"文档 {topic} 不存在。可用文档：{get_available_docs()}"

# ---------- 交接管理 ----------

def generate_handoff(task: str, completed: list, state: dict, next_steps: list) -> dict:
    return {
        "timestamp": datetime.now().isoformat(),
        "task": task,
        "completed": completed,
        "state": state,
        "next_steps": next_steps,
    }

def save_handoff(handoff: dict, session_id: str):
    path = Path(".harness/handoffs")
    path.mkdir(parents=True, exist_ok=True)
    with open(path / f"{session_id}.json", "w") as f:
        json.dump(handoff, f, ensure_ascii=False, indent=2)

def load_latest_handoff() -> dict | None:
    path = Path(".harness/handoffs")
    if not path.exists():
        return None
    files = sorted(path.glob("*.json"), key=lambda f: f.stat().st_mtime, reverse=True)
    if not files:
        return None
    with open(files[0]) as f:
        return json.load(f)

# ---------- 运行器 ----------

def build_initial_context() -> str:
    """构建会话初始上下文"""
    parts = []

    # 1. 加载 AGENTS.md
    agents_md = load_agents_md()
    if agents_md:
        parts.append(f"# 项目规则\n{agents_md}")

    # 2. 检查是否有交接文档
    handoff = load_latest_handoff()
    if handoff:
        parts.append(f"# 接力信息\n你正在接手一个进行中的任务。\n"
                     f"原始任务：{handoff['task']}\n"
                     f"已完成：{json.dumps(handoff['completed'], ensure_ascii=False)}\n"
                     f"下一步：{json.dumps(handoff['next_steps'], ensure_ascii=False)}")

    # 3. 告诉 Agent 有哪些文档可以按需查阅
    docs = get_available_docs()
    if docs:
        parts.append(f"# 可查阅文档\n以下文档可以按需查阅（调用 get_doc 工具）：\n"
                     f"{', '.join(docs)}")

    return "\n\n---\n\n".join(parts)

if __name__ == "__main__":
    context = build_initial_context()
    print("=== 初始上下文 ===")
    print(context)
    print(f"\n=== 上下文长度：{len(context)} 字符 ===")
```

### 运行效果

```bash
python context_runner.py
```

```text
=== 初始上下文 ===
# 项目规则
# AGENTS.md

## 项目概览
Next.js 14 全栈项目，App Router + TypeScript 严格模式。
...

---

# 可查阅文档
以下文档可以按需查阅（调用 get_doc 工具）：
api-conventions, testing-guide, database-schema

=== 上下文长度：423 字符 ===
```

Agent 启动时只拿到项目规则和文档索引，工作台是干净的。需要查 API 规范时调用 `get_doc("api-conventions")`，不需要的信息不进上下文。任务跑到一半上下文快满了，生成交接文档、启动新会话、加载交接继续——全程不丢关键信息。

## 什么时候需要上下文管理，什么时候不需要

| 场景 | 需不需要 | 原因 |
|------|---------|------|
| 单次简单问答 | 不需要 | 一轮对话就结束，没有上下文积累 |
| 短任务（< 5 轮对话） | 通常不需要 | 上下文压力不大 |
| 长任务（> 20 轮对话） | 需要 | 上下文退化效应开始显现 |
| 多 Agent 协作 | 需要 | 必须隔离各 Agent 的上下文，防止污染 |
| 跨会话的持续任务 | 必须有 | 没有交接机制，每个新会话都是从零开始 |

判断标准：**这个任务会不会在一个上下文窗口里就做完？** 如果会，不用管上下文。如果不会，就需要规划怎么分段、怎么交接、怎么重置。

## 上下文管理与其他四个 Harness 要素的关系

- **上下文管理 + 约束**：约束控制行为边界（能做什么），上下文管理控制信息边界（能看到什么）。两者配合，Agent 只在正确的信息和正确的权限下工作。
- **上下文管理 + 沙箱**：沙箱隔离了执行环境，上下文管理隔离了信息环境。Agent 既碰不到不该碰的文件，也看不到不该看的信息。
- **上下文管理 + 反馈回路**：交接文档本身就是一种反馈产物——记录了做了什么、结果如何、哪里有问题。新会话基于这些反馈继续优化。
- **上下文管理 + 可观测性**：上下文的使用率、交接次数、每次交接丢失了多少信息——这些都是可观测指标，帮助你判断上下文策略是否合理。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
