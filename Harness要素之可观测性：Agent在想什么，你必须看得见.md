# Harness 要素之可观测性：Agent 在想什么，你必须看得见

> Agent 跑了两个小时，最后告诉你"任务完成"。你问它中间做了什么决策、为什么选了这条路径、哪一步开始偏的——它说不清楚，你也查不到。可观测性要解决的，就是让 Agent 的每一步决策、每一次工具调用、每一个中间状态都有迹可查。出了问题不用猜，直接看。

## 核心判断

一句话分清：

- **不可观测的 Agent = 黑箱赌博机。** 结果好就用，结果差就重跑，但你永远不知道为什么好、为什么差。
- **可观测的 Agent = 透明流水线。** 每一步都有记录，出了问题精确定位到哪个 Agent、哪一步、哪次调用偏了。

"能跑起来"和"能看清楚它在干什么"之间，差着一整套工程基建。

## 没有可观测性会出什么事

| 问题 | 表现 | 调试成本 |
|------|------|---------|
| 结果错了但不知道哪里错的 | 只看到最终输出有问题，不知道是第几步偏的 | 只能从头重跑，反复试 |
| 间歇性失败 | 同样的任务有时成功有时失败，找不到规律 | 靠猜，或者加大量 print 调试 |
| 性能退化 | Agent 越来越慢，不知道是哪个工具调用卡住了 | 无法定位瓶颈 |
| 成本失控 | LLM 调用费用暴涨，不知道是哪个环节在浪费 token | 只能看总账单，看不到明细 |
| 复盘无据 | 想优化 Agent 的行为，但没有历史数据可以分析 | 全凭记忆和直觉 |

一个关键认知：可观测性不是"加日志"。日志是最基础的一层，但远远不够。你需要的是完整的调用链路、结构化的状态快照和可回溯的决策记录。

## 可观测性在 Harness 架构中的位置

可观测性是 Harness 五要素中最"基础"的一个——它不直接控制 Agent 的行为，但它让其他四个要素的执行状态变得透明。约束拦截了多少次、沙箱里发生了什么、上下文用了多少、反馈回路循环了几轮——这些信息都需要可观测性来呈现。

```text
┌─────────────────────────────────────────┐
│  约束 │ 沙箱 │ 上下文管理 │ 反馈回路       │
├─────────────────────────────────────────┤
│          可观测性（全局透视层）             │
│  调用追踪 │ 状态快照 │ 决策日志 │ 指标监控   │
├─────────────────────────────────────────┤
│          Agent（执行层）                │
└─────────────────────────────────────────┘
```

## 可观测性的四种落地形态

### 1. 调用链追踪：每一步都有迹可查

Agent 的每次 LLM 调用、每次工具使用、每次阶段跳转，都记录成一条结构化的 trace 记录。出了问题，顺着 trace 就能找到是哪一步开始偏的。

```python
# tracer.py
"""调用链追踪：记录 Agent 的每一步操作"""

import json
import time
import uuid
from pathlib import Path
from datetime import datetime

class Tracer:
    def __init__(self, task_id: str = None):
        self.task_id = task_id or str(uuid.uuid4())[:8]
        self.traces = []
        self.start_time = time.time()

    def record(self, event_type: str, data: dict):
        """记录一个事件"""
        self.traces.append({
            "task_id": self.task_id,
            "timestamp": datetime.now().isoformat(),
            "elapsed_ms": int((time.time() - self.start_time) * 1000),
            "event_type": event_type,
            "data": data,
        })

    def record_llm_call(self, model: str, messages: list, response: str,
                         tokens_in: int, tokens_out: int, duration_ms: int):
        self.record("llm_call", {
            "model": model,
            "messages_count": len(messages),
            "prompt_preview": str(messages[-1])[:200],
            "response_preview": response[:200],
            "tokens_in": tokens_in,
            "tokens_out": tokens_out,
            "duration_ms": duration_ms,
        })

    def record_tool_call(self, tool: str, args: dict, result: str, success: bool):
        self.record("tool_call", {
            "tool": tool,
            "args": args,
            "result_preview": result[:500],
            "success": success,
        })

    def record_phase_transition(self, from_phase: str, to_phase: str, reason: str):
        self.record("phase_transition", {
            "from": from_phase,
            "to": to_phase,
            "reason": reason,
        })

    def record_constraint_hit(self, constraint: str, action: str, allowed: bool):
        self.record("constraint_check", {
            "constraint": constraint,
            "action": action,
            "allowed": allowed,
        })

    def save(self, output_dir: str = ".harness/traces"):
        path = Path(output_dir)
        path.mkdir(parents=True, exist_ok=True)
        filepath = path / f"{self.task_id}.json"
        filepath.write_text(json.dumps(self.traces, ensure_ascii=False, indent=2))
        return str(filepath)

    def summary(self) -> dict:
        """生成执行摘要"""
        llm_calls = [t for t in self.traces if t["event_type"] == "llm_call"]
        tool_calls = [t for t in self.traces if t["event_type"] == "tool_call"]
        constraints = [t for t in self.traces if t["event_type"] == "constraint_check"]

        total_tokens = sum(
            t["data"]["tokens_in"] + t["data"]["tokens_out"] for t in llm_calls
        )
        failed_tools = [t for t in tool_calls if not t["data"]["success"]]
        blocked = [t for t in constraints if not t["data"]["allowed"]]

        return {
            "task_id": self.task_id,
            "total_duration_ms": int((time.time() - self.start_time) * 1000),
            "llm_calls": len(llm_calls),
            "total_tokens": total_tokens,
            "tool_calls": len(tool_calls),
            "failed_tool_calls": len(failed_tools),
            "constraint_blocks": len(blocked),
        }
```

### 2. 状态快照：任务进行到哪了，一目了然

长任务运行过程中，需要在关键节点保存状态快照。快照不是日志——日志是流水账，快照是某个时刻的完整状态切面。

```python
# snapshot.py
"""状态快照：在关键节点保存 Agent 的完整状态"""

import json
from datetime import datetime
from pathlib import Path

class SnapshotManager:
    def __init__(self, output_dir: str = ".harness/snapshots"):
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)

    def capture(self, task_id: str, phase: str, agent_state: dict) -> str:
        """在阶段切换时自动捕获快照"""
        snapshot = {
            "task_id": task_id,
            "timestamp": datetime.now().isoformat(),
            "phase": phase,
            "state": {
                "modified_files": agent_state.get("modified_files", []),
                "test_results": agent_state.get("test_results"),
                "context_utilization": agent_state.get("context_utilization"),
                "pending_issues": agent_state.get("pending_issues", []),
                "tool_call_count": agent_state.get("tool_call_count", 0),
            },
        }

        filename = f"{task_id}_{phase}_{datetime.now().strftime('%H%M%S')}.json"
        filepath = self.output_dir / filename
        filepath.write_text(json.dumps(snapshot, ensure_ascii=False, indent=2))
        return str(filepath)

    def list_snapshots(self, task_id: str) -> list[dict]:
        """列出某个任务的所有快照，按时间排序"""
        snapshots = []
        for f in sorted(self.output_dir.glob(f"{task_id}_*.json")):
            snapshots.append(json.loads(f.read_text()))
        return snapshots

    def diff(self, snap_a: dict, snap_b: dict) -> dict:
        """对比两个快照的差异"""
        return {
            "phase_change": f"{snap_a['phase']} → {snap_b['phase']}",
            "new_modified_files": [
                f for f in snap_b["state"]["modified_files"]
                if f not in snap_a["state"]["modified_files"]
            ],
            "context_utilization_change":
                (snap_b["state"].get("context_utilization", 0) -
                 snap_a["state"].get("context_utilization", 0)),
            "new_issues": [
                i for i in snap_b["state"]["pending_issues"]
                if i not in snap_a["state"]["pending_issues"]
            ],
        }
```

### 3. 决策日志：Agent 为什么选了这条路

调用链记录了"做了什么"，决策日志记录"为什么这么做"。让 Agent 在每个关键决策点输出结构化的决策理由。

```python
# decision_log.py
"""决策日志：记录 Agent 的决策理由，不只记录动作"""

import json
from pathlib import Path
from datetime import datetime

class DecisionLogger:
    def __init__(self, output_dir: str = ".harness/decisions"):
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)
        self.decisions = []

    def log_decision(self, task_id: str, decision_point: str,
                      options: list[str], chosen: str, reason: str):
        """记录一个决策点"""
        entry = {
            "task_id": task_id,
            "timestamp": datetime.now().isoformat(),
            "decision_point": decision_point,
            "options_considered": options,
            "chosen": chosen,
            "reason": reason,
        }
        self.decisions.append(entry)
        return entry

    def save(self, task_id: str):
        filepath = self.output_dir / f"{task_id}_decisions.json"
        filepath.write_text(json.dumps(self.decisions, ensure_ascii=False, indent=2))
```

在编排层里，当 Agent 面临选择时（比如 Evaluator 打回后 Generator 是优化当前方案还是换一个方向），记录下来：

```python
decision_logger.log_decision(
    task_id="fix-login-bug",
    decision_point="评审未通过后的策略选择",
    options=["优化当前实现", "换用 session-based 认证", "重构登录组件"],
    chosen="优化当前实现",
    reason="评审反馈指出的是边界情况处理缺失，核心逻辑没问题，不需要换方向",
)
```

这些记录在复盘时极其有价值——不只是看到结果，而是理解 Agent 的决策链路。

### 4. 指标监控：系统级的健康度仪表盘

单个任务的追踪之上，还需要系统级的聚合指标。这些指标帮助你判断 Harness 整体是否健康。

```python
# metrics.py
"""指标聚合：系统级的健康度监控"""

import json
from pathlib import Path
from collections import defaultdict

class MetricsAggregator:
    def __init__(self, traces_dir: str = ".harness/traces"):
        self.traces_dir = Path(traces_dir)

    def aggregate(self) -> dict:
        """聚合所有任务的指标"""
        stats = defaultdict(list)

        for trace_file in self.traces_dir.glob("*.json"):
            traces = json.loads(trace_file.read_text())
            task_id = traces[0]["task_id"] if traces else "unknown"

            llm_calls = [t for t in traces if t["event_type"] == "llm_call"]
            tool_calls = [t for t in traces if t["event_type"] == "tool_call"]
            constraints = [t for t in traces if t["event_type"] == "constraint_check"]

            total_tokens = sum(
                t["data"]["tokens_in"] + t["data"]["tokens_out"]
                for t in llm_calls
            )
            failed_tools = sum(1 for t in tool_calls if not t["data"]["success"])
            blocked = sum(1 for t in constraints if not t["data"]["allowed"])

            stats["tokens_per_task"].append(total_tokens)
            stats["tool_failures_per_task"].append(failed_tools)
            stats["constraint_blocks_per_task"].append(blocked)
            stats["llm_calls_per_task"].append(len(llm_calls))

        return {
            "total_tasks": len(stats["tokens_per_task"]),
            "avg_tokens_per_task": self._avg(stats["tokens_per_task"]),
            "avg_tool_failures": self._avg(stats["tool_failures_per_task"]),
            "avg_constraint_blocks": self._avg(stats["constraint_blocks_per_task"]),
            "avg_llm_calls": self._avg(stats["llm_calls_per_task"]),
        }

    def _avg(self, values: list) -> float:
        return round(sum(values) / len(values), 2) if values else 0

    def print_dashboard(self):
        metrics = self.aggregate()
        print("=" * 50)
        print("Harness 健康度仪表盘")
        print("=" * 50)
        print(f"总任务数:          {metrics['total_tasks']}")
        print(f"平均 token 消耗:   {metrics['avg_tokens_per_task']}")
        print(f"平均 LLM 调用次数: {metrics['avg_llm_calls']}")
        print(f"平均工具调用失败:  {metrics['avg_tool_failures']}")
        print(f"平均约束拦截次数:  {metrics['avg_constraint_blocks']}")
        print("=" * 50)
```

这些指标能回答很多关键问题：
- **约束拦截次数在增加？** 可能 Agent 的 prompt 需要优化，也可能约束规则太严了
- **工具调用失败率在升高？** 可能沙箱环境有问题，或者工具 API 不稳定
- **token 消耗在暴涨？** 可能上下文管理出了问题，需要检查是否有无效的信息被反复塞入

## 实战操作：给编排层接上完整的可观测性

下面把上面四种能力串进一个完整的编排器里。

```python
# observable_orchestrator.py
"""带完整可观测性的编排器"""

import time
from tracer import Tracer
from snapshot import SnapshotManager
from decision_log import DecisionLogger

class ObservableOrchestrator:
    def __init__(self, task_id: str, task: str):
        self.task_id = task_id
        self.task = task
        self.tracer = Tracer(task_id)
        self.snapshots = SnapshotManager()
        self.decisions = DecisionLogger()
        self.phase = "research"

    def run(self):
        print(f"开始任务: {self.task_id}")

        while self.phase != "done":
            # 捕获阶段入口快照
            self.snapshots.capture(self.task_id, self.phase, self._get_state())

            # 执行当前阶段
            start = time.time()
            result = self._execute_phase()
            duration = int((time.time() - start) * 1000)

            # 记录执行结果
            self.tracer.record("phase_complete", {
                "phase": self.phase,
                "duration_ms": duration,
                "result": result,
            })

            # 决定下一步
            next_phase = self._decide_next(result)
            self.tracer.record_phase_transition(self.phase, next_phase, result)
            self.phase = next_phase

        # 保存所有追踪数据
        trace_path = self.tracer.save()
        self.decisions.save(self.task_id)

        # 输出摘要
        summary = self.tracer.summary()
        print(f"\n{'='*50}")
        print(f"任务完成: {self.task_id}")
        print(f"总耗时: {summary['total_duration_ms']}ms")
        print(f"LLM 调用: {summary['llm_calls']} 次")
        print(f"Token 消耗: {summary['total_tokens']}")
        print(f"工具调用: {summary['tool_calls']} 次 (失败: {summary['failed_tool_calls']})")
        print(f"约束拦截: {summary['constraint_blocks']} 次")
        print(f"追踪文件: {trace_path}")
        print(f"{'='*50}")

    def _execute_phase(self):
        # 实际的 LLM 调用 + 工具执行
        # 每次调用都通过 self.tracer 记录
        return "completed"

    def _decide_next(self, result):
        # 决策逻辑 + 记录决策理由
        return "done"

    def _get_state(self):
        return {"modified_files": [], "tool_call_count": 0}
```

### 运行效果

```text
开始任务: fix-login-bug

==================================================
任务完成: fix-login-bug
总耗时: 45200ms
LLM 调用: 8 次
Token 消耗: 12340
工具调用: 15 次 (失败: 2)
约束拦截: 1 次
追踪文件: .harness/traces/fix-login-bug.json
==================================================
```

出了问题时，打开 `.harness/traces/fix-login-bug.json` 就能看到完整链路：

```json
[
  {
    "task_id": "fix-login-bug",
    "timestamp": "2026-04-07T23:30:15",
    "elapsed_ms": 0,
    "event_type": "phase_transition",
    "data": {"from": "research", "to": "plan", "reason": "completed"}
  },
  {
    "task_id": "fix-login-bug",
    "timestamp": "2026-04-07T23:30:18",
    "elapsed_ms": 3200,
    "event_type": "tool_call",
    "data": {"tool": "write_file", "args": {"path": ".env"}, "result_preview": "Permission denied: .env is in protected list", "success": false}
  },
  {
    "task_id": "fix-login-bug",
    "timestamp": "2026-04-07T23:30:18",
    "elapsed_ms": 3201,
    "event_type": "constraint_check",
    "data": {"constraint": "file_acl", "action": "write .env", "allowed": false}
  }
]
```

一眼就看到：Agent 在 3.2 秒时试图写 `.env`，被约束层拦截了。不用猜，不用重跑，直接定位。

## 什么时候需要可观测性，什么时候不需要

| 场景 | 需不需要 | 原因 |
|------|---------|------|
| 单次简单任务 | 不需要 | 结果直接看得见 |
| 调试阶段 | 需要 | 没有追踪数据就只能靠猜 |
| 多 Agent 协作 | 需要 | 必须能追踪到是哪个 Agent 出了问题 |
| 生产环境运行 | 必须有 | 故障定位、成本优化、性能调优都依赖可观测数据 |
| 持续优化 Agent 行为 | 必须有 | 没有历史数据就无法做数据驱动的优化 |

判断标准：**如果 Agent 出了问题，你能在 5 分钟内定位到原因吗？** 不能，就需要可观测性。

## 可观测性与其他四个 Harness 要素的关系

- **可观测性 + 约束**：记录约束拦截了什么、多少次、哪些 Agent 最容易触发拦截。这些数据帮助你优化约束规则——太严的放松，太松的收紧。
- **可观测性 + 沙箱**：监控沙箱的资源使用（CPU、内存、磁盘 IO），发现 Agent 的异常行为（比如无限循环、内存泄漏）。
- **可观测性 + 上下文管理**：追踪上下文使用率变化曲线、交接次数和交接质量。如果每次交接后 Agent 的表现都下降，说明交接文档设计有问题。
- **可观测性 + 反馈回路**：记录反馈循环的轮次、每轮的改进幅度、最终是通过还是升级人工。这些数据帮助你优化反馈策略和阈值。

## 系列收束

五篇文章到这里全部写完。回顾一下 Harness Engineering 的五个核心要素：

| 要素 | 管什么 | 核心机制 |
|------|-------|---------|
| **约束** | 能做什么 | 阶段门禁 + 文件权限 + 验证卡控 + CI 门禁 |
| **沙箱隔离** | 在哪里做 | 文件系统隔离 + 网络隔离 + 资源限制 + 会话隔离 |
| **上下文管理** | 看到什么 | AGENTS.md + 按需检索 + 结构化交接 + 上下文重置 |
| **反馈回路** | 做得对不对 | 自动验证 + Gen-Eval 分离 + 循环检测 + 经验沉淀 |
| **可观测性** | 发生了什么 | 调用追踪 + 状态快照 + 决策日志 + 指标监控 |

五个要素不是五个独立的功能，而是一个互相依赖的系统。约束定义边界，沙箱保证隔离，上下文管理喂对信息，反馈回路校验质量，可观测性让一切透明。缺了任何一个，系统都会有盲区。

不用一次全搭齐。先从最痛的地方开始——如果 Agent 经常越界就先补约束，如果结果不稳定就先补反馈回路，如果出了问题找不到原因就先补可观测性。一层一层加，每加一层都能把稳定性拉上一个台阶。

## 延伸阅读

本文是 Harness Engineering 五要素系列的最后一篇。其他四个要素：

- [Harness 要素之约束：不是写在 prompt 里的建议，而是编进流程的硬卡控](Harness要素之约束：不是写在prompt里的建议，而是编进流程的硬卡控.md)
- [Harness 要素之沙箱隔离：让 Agent 只能在围栏里干活](Harness要素之沙箱隔离：让Agent只能在围栏里干活.md)
- [Harness 要素之上下文管理：Agent 跑偏的根源是喂错了信息](Harness要素之上下文管理：Agent跑偏的根源是喂错了信息.md)
- [Harness 要素之反馈回路：不是跑完就算，而是跑歪了能自动拉回来](Harness要素之反馈回路：不是跑完就算，而是跑歪了能自动拉回来.md)

Claude Code 的 Harness 架构全景分析：

- [Claude Code 就是一个 Harness：从泄露的 51 万行源码看 Agent 的运行底座](Claude-Code就是一个Harness：从泄露源码看Agent的运行底座.md)

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
