# Harness 要素之可观测性：Agent 在想什么，必须看得见

> Agent 跑了两个小时，最后返回"任务完成"。想查它中间做了什么决策、为什么选了这条路径、哪一步开始偏的——它说不清楚，日志里也找不到。
>
> 可观测性要解决的就是这个问题：让 Agent 的决策过程、工具调用和中间状态都有迹可查。出了问题不用猜，直接看。

## 核心判断

一句话分清：

- **不可观测的 Agent = 蒙着眼跑。** 结果好就用，结果差就重跑，但永远不知道为什么好、为什么差。
- **可观测的 Agent = 开着仪表盘跑。** 每一步都有记录，出了问题精确定位到哪个 Agent、哪一步、哪次调用偏了。

"能跑起来"和"能看清楚它在干什么"之间，差着一整套工程基建。

## 没有可观测性会出什么事

| 问题 | 表现 | 调试成本 |
|------|------|----------|
| 结果错了但不知道哪里错 | 只看到最终输出有问题，不知道是第几步偏的 | 需要完整重跑任务，无法定位断点 |
| 间歇性失败 | 同样的任务有时成功有时失败，找不到规律 | 缺少复现条件，只能盲目添加调试代码 |
| 性能退化 | Agent 越来越慢，不知道是哪个工具调用卡住了 | 无法定位瓶颈，缺少各环节耗时数据 |
| 成本失控 | LLM 调用费用暴涨，不知道是哪个环节在浪费 token | 只有总账单，无法拆分到单次调用粒度 |
| 复盘无据 | 想优化 Agent 的行为，但没有历史数据可分析 | 无决策记录可回溯，只能凭直觉调优 |

这些问题的共同点是：出了事只能靠猜、靠重跑。

可观测性不是"加日志"。日志是最基础的一层，但远不够。完整的调用链路、结构化的状态快照、可回溯的决策记录——三者缺一不可。

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

## Agent 可观测性与传统可观测性的关系

传统服务的可观测性已经有成熟的三大支柱（three pillars）：Traces（调用链）、Metrics（指标）、Logs（日志）。Agent 可观测性继承了这套基本理念，但因为 Agent 有一个传统服务没有的维度——**决策**——所以需要额外的手段。

| 维度 | 传统服务可观测性 | Agent 可观测性 | 差异点 |
|------|----------------|---------------|--------|
| **调用链追踪** | 请求在微服务间的传播链路 | LLM 调用 + 工具调用 + 阶段跳转的完整链路 | Agent 的调用链包含非确定性的 LLM 推理步骤 |
| **日志** | 结构化事件流，按时间线记录 | 同样是事件流，但需要记录 prompt/response 等大文本 | 单条日志的信息密度和体积远高于传统日志 |
| **指标监控** | QPS、延迟、错误率等聚合指标 | token 消耗、工具调用失败率、约束拦截次数等 | 指标维度从"资源消耗"扩展到"行为质量" |
| **状态快照** | 类似 checkpoint，但不常用 | 某个时刻的完整状态切面（文件、上下文、待办） | Agent 任务时间长，必须能从任意节点恢复现场 |
| **决策日志** | 无对应概念 | 记录"为什么选这条路"而非"做了什么" | Agent 独有，传统服务不需要记录决策理由 |

一句话概括：传统三板斧 Agent 全都要，但光有这三样还不够。**状态快照**解决长任务的现场恢复问题，**决策日志**解决"Agent 为什么这么干"的归因问题。

## 可观测性的四种落地形态

### 1. 调用链追踪：每一步都有迹可查

Agent 的 LLM 调用、工具使用、阶段跳转，都记录成结构化的 trace。出了问题，顺着 trace 就能找到是哪一步开始偏的。

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
            "tool_call_count_change":
                (snap_b["state"].get("tool_call_count", 0) -
                 snap_a["state"].get("tool_call_count", 0)),
            "test_results_change": {
                "before": snap_a["state"].get("test_results"),
                "after": snap_b["state"].get("test_results"),
            },
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
from collections import Counter

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

    def query_by_decision_point(self, keyword: str) -> list[dict]:
        """按关键词检索决策记录"""
        return [d for d in self.decisions if keyword in d["decision_point"]]

    def analyze(self) -> dict:
        """分析决策模式：哪些选项被选过、哪些从未被选"""
        chosen_counts = Counter(d["chosen"] for d in self.decisions)
        all_options = set()
        for d in self.decisions:
            all_options.update(d["options_considered"])
        never_chosen = all_options - set(chosen_counts.keys())
        return {
            "total_decisions": len(self.decisions),
            "chosen_distribution": dict(chosen_counts),
            "never_chosen_options": list(never_chosen),
        }

    def save(self, task_id: str):
        filepath = self.output_dir / f"{task_id}_decisions.json"
        filepath.write_text(json.dumps(self.decisions, ensure_ascii=False, indent=2))
```

在编排层里，Agent 每次面临分支选择时（比如 Evaluator 打回后，Generator 是优化当前方案还是换方向），都应该记录下来：

```python
decision_logger.log_decision(
    task_id="fix-login-bug",
    decision_point="评审未通过后的策略选择",
    options=["优化当前实现", "换用 session-based 认证", "重构登录组件"],
    chosen="优化当前实现",
    reason="评审反馈指出的是边界情况处理缺失，核心逻辑没问题，不需要换方向",
)
```

这些记录在复盘时价值很高——不只看到结果，还能还原 Agent 的完整决策链路。

### 4. 指标监控：系统级的健康度仪表盘

单个任务的追踪之上，还需要系统级的聚合指标，用来判断 Harness 整体是否健康。

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
            stats["tool_calls_per_task"].append(len(tool_calls))

        return {
            "total_tasks": len(stats["tokens_per_task"]),
            "avg_tokens_per_task": self._avg(stats["tokens_per_task"]),
            "avg_tool_failures": self._avg(stats["tool_failures_per_task"]),
            "avg_constraint_blocks": self._avg(stats["constraint_blocks_per_task"]),
            "avg_llm_calls": self._avg(stats["llm_calls_per_task"]),
            "avg_tool_calls_per_task": self._avg(stats["tool_calls_per_task"]),
        }

    def _avg(self, values: list) -> float:
        return round(sum(values) / len(values), 2) if values else 0

    def check_alerts(self, token_threshold: int = 20000,
                      failure_rate_threshold: float = 0.15,
                      constraint_block_threshold: int = 3) -> list[str]:
        """检查是否触发告警"""
        metrics = self.aggregate()
        alerts = []
        if metrics["avg_tokens_per_task"] > token_threshold:
            alerts.append(
                f"token 消耗过高: 平均 {metrics['avg_tokens_per_task']}，阈值 {token_threshold}")
        if metrics["avg_tool_calls_per_task"] > 0:
            failure_rate = metrics["avg_tool_failures"] / metrics["avg_tool_calls_per_task"]
            if failure_rate > failure_rate_threshold:
                alerts.append(
                    f"工具调用失败率过高: {failure_rate:.1%}，阈值 {failure_rate_threshold:.0%}")
        if metrics["avg_constraint_blocks"] > constraint_block_threshold:
            alerts.append(
                f"约束拦截频繁: 平均 {metrics['avg_constraint_blocks']} 次，阈值 {constraint_block_threshold}")
        return alerts

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
        self._state = {"modified_files": [], "tool_call_count": 0, "test_results": None,
                        "context_utilization": 0.0, "pending_issues": []}

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
        """执行当前阶段：模拟 LLM 调用和工具调用"""
        phase_actions = {
            "research": self._phase_research,
            "plan": self._phase_plan,       # 结构与 research 类似，省略详细实现
            "implement": self._phase_implement,
            "review": self._phase_review,   # 结构与 implement 类似，省略详细实现
        }
        action = phase_actions.get(self.phase, lambda: "skipped")
        return action()

    def _phase_research(self):
        """研究阶段：展示 LLM 调用 + 工具调用的追踪"""
        # 模拟 LLM 分析任务
        self.tracer.record_llm_call(
            model="claude-sonnet", messages=[{"role": "user", "content": self.task}],
            response="需要修复登录模块的 token 刷新逻辑", tokens_in=320, tokens_out=180, duration_ms=1200,
        )
        # 模拟工具调用：读取相关文件
        self.tracer.record_tool_call(
            tool="read_file", args={"path": "src/auth/login.py"},
            result="class LoginHandler:...", success=True,
        )
        self._state["tool_call_count"] += 1
        return "found_root_cause"

    def _phase_implement(self):
        """实现阶段：展示约束拦截 + 工具调用成功/失败的追踪"""
        # 模拟约束检查：尝试写受保护的文件
        self.tracer.record_constraint_hit(constraint="file_acl", action="write .env", allowed=False)
        # 记录被拦截导致的工具调用失败
        self.tracer.record_tool_call(
            tool="write_file", args={"path": ".env"},
            result="Blocked by file_acl: .env is in protected list", success=False,
        )
        # 模拟写允许的文件
        self.tracer.record_tool_call(
            tool="write_file", args={"path": "src/auth/login.py"},
            result="File written successfully", success=True,
        )
        self._state["modified_files"].append("src/auth/login.py")
        self._state["tool_call_count"] += 1
        self.tracer.record_llm_call(
            model="claude-sonnet", messages=[{"role": "user", "content": "实现修复"}],
            response="已添加 token 提前刷新逻辑", tokens_in=1200, tokens_out=860, duration_ms=3200,
        )
        return "implementation_done"

    # 以下为 _phase_plan 和 _phase_review 的简化版实现，
    # 展示追踪记录的核心模式：
    # 调用 tracer.record_llm_call() 记录 LLM 推理，
    # 调用 tracer.record_tool_call() 记录工具操作，
    # 在约束相关环节调用 tracer.record_constraint_hit()。

    def _phase_plan(self):
        self.tracer.record_llm_call(
            model="claude-sonnet", messages=[{"role": "user", "content": "制定修复方案"}],
            response="方案：在 refresh_token 过期前 5 分钟触发刷新", tokens_in=580, tokens_out=420, duration_ms=1800,
        )
        return "plan_ready"

    def _phase_review(self):
        self.tracer.record_tool_call(
            tool="run_tests", args={"path": "tests/test_auth.py"},
            result="4 passed, 0 failed", success=True,
        )
        self._state["tool_call_count"] += 1
        self._state["test_results"] = {"passed": 4, "failed": 0}
        return "review_passed"

    def _decide_next(self, result):
        """决定下一阶段，并记录决策理由"""
        phase_flow = {
            "research": ("plan", "research → plan",
                         ["直接实现", "先制定方案再实现"],
                         "先制定方案再实现",
                         "已定位到 root cause，需要先规划修复方案再动手"),
            "plan": ("implement", "plan → implement",
                     ["实现方案 A（提前刷新）", "实现方案 B（延长过期时间）"],
                     "实现方案 A（提前刷新）",
                     "提前刷新是更安全的做法，不依赖服务端配置变更"),
            "implement": ("review", "implement → review",
                          ["跳过审查直接完成", "运行测试 + 代码审查"],
                          "运行测试 + 代码审查",
                          "涉及认证模块，必须经过测试验证"),
            "review": ("done", "review → done",
                       ["标记完成", "追加集成测试"],
                       "标记完成",
                       "单元测试已覆盖核心逻辑，无需追加集成测试"),
        }
        if self.phase in phase_flow:
            next_p, dp, options, chosen, reason = phase_flow[self.phase]
            self.decisions.log_decision(
                task_id=self.task_id, decision_point=dp,
                options=options, chosen=chosen, reason=reason,
            )
            return next_p
        return "done"

    def _get_state(self):
        return dict(self._state)
```

### 运行效果

以下是生产环境中一个典型的任务输出。四阶段编排器在实际任务中会产生更多的 LLM 调用和工具操作。注意：下面的数字是生产环境的参考值，与上方简化示例代码的实际运行输出不完全一致——简化版只模拟了少量调用，生产环境的调用次数和 token 消耗会高得多：

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
    "event_type": "constraint_check",
    "data": {"constraint": "file_acl", "action": "write .env", "allowed": false}
  },
  {
    "task_id": "fix-login-bug",
    "timestamp": "2026-04-07T23:30:18",
    "elapsed_ms": 3201,
    "event_type": "tool_call",
    "data": {"tool": "write_file", "args": {"path": ".env"}, "result_preview": "Blocked by file_acl: .env is in protected list", "success": false}
  }
]
```

一眼就看到：Agent 在 3.2 秒时试图写 `.env`，被约束层拦截了。不用猜，不用重跑，直接定位。

### 拿到 trace 之后怎么用

记录只是第一步，关键是能快速从 trace 中提取有价值的信息。几个常用的分析操作：

```bash
# 查看某次任务中所有被约束拦截的事件
jq '[.[] | select(.event_type == "constraint_check" and .data.allowed == false)]' \
  .harness/traces/fix-login-bug.json

# 按耗时排序，找出最慢的 LLM 调用
jq '[.[] | select(.event_type == "llm_call")] | sort_by(.data.duration_ms) | reverse | .[0:3]' \
  .harness/traces/fix-login-bug.json

# 统计工具调用的成功/失败分布
jq '[.[] | select(.event_type == "tool_call")] | group_by(.data.success) | map({success: .[0].data.success, count: length})' \
  .harness/traces/fix-login-bug.json
```

三条命令分别对应三种排查场景：约束配置是否合理、哪个环节最慢、工具链是否稳定。

## 现有工具生态：自建还是接入

前面的代码全部是自建方案，目的是把可观测性的核心机制讲透：trace 怎么串、snapshot 怎么切、决策日志怎么结构化。但到了生产环境，不一定需要从零造轮子——行业里已经有成熟的工具链可以直接接入。

**调用链追踪方面**，OpenTelemetry（OTel）是当前的行业标准。它定义了统一的 trace、metric、log 数据模型和传输协议，几乎所有主流语言都有 SDK。Agent 场景下的 LLM 调用、工具调用同样可以用 OTel 的 Span 来建模——一个 Agent 任务是一个顶层 Trace，每次 LLM 调用和工具调用分别是子 Span，阶段跳转用 Span 事件标记。好处是一旦接入 OTel，后端可以对接 Jaeger、Grafana Tempo 等已有的分布式追踪系统，不需要自己写存储和查询。

**Agent 专用可观测性平台方面**，LangSmith 和 LangFuse 是两个代表。它们专门针对 LLM 应用设计，内置了 prompt 版本管理、token 成本统计、调用链可视化、评估打分等功能。相比通用的 OTel 方案，这类平台对 Agent 场景的支持更精细——比如能直接看到每次 LLM 调用的 prompt/response 全文、自动计算不同模型的调用成本、支持基于 trace 的回放和对比分析。

**自建和接入不是二选一的关系。** 判断标准很直接：如果团队的 Agent 系统还在原型阶段，用本文的自建方案足够把核心机制跑通，也更容易根据自身需求定制；当 Agent 开始服务真实用户、任务量上来之后，trace 的存储、查询、可视化和告警会变成工程负担，这时候接入 OTel + 专用平台是更务实的选择。很多团队的做法是：核心的 trace 数据模型自己定义（保证和业务逻辑贴合），存储和展示层接入现有平台（避免重复造轮子）。

需要注意的是，不管用哪种方案，前面讲的四种落地形态（调用链追踪、状态快照、决策日志、指标监控）都是必须覆盖的。工具只是载体，观测的维度和粒度才是设计的重点。

## 什么时候需要可观测性，什么时候不需要

| 场景 | 需不需要 | 原因 |
|------|---------|------|
| 单次简单任务 | 不需要 | 结果直接看得见 |
| 调试阶段 | 需要 | 没有追踪数据就只能靠猜 |
| 多 Agent 协作 | 需要 | 必须能追踪到是哪个 Agent 出了问题 |
| 生产环境运行 | 必须有 | 故障定位、成本优化、性能调优都依赖可观测数据 |
| 持续优化 Agent 行为 | 必须有 | 没有历史数据就无法做数据驱动的优化 |

判断标准：**如果 Agent 出了问题，能快速定位到原因吗？还是只能从头重跑、靠猜？** 不能快速定位，就需要可观测性。

### 可观测性的开销意识

可观测性本身也有成本。每条 trace 占存储空间，每次 snapshot 有 IO 开销，LLM 的 prompt/response 原文记录会迅速膨胀存储。

粗略估算一下量级：一次包含 8 次 LLM 调用和 15 次工具调用的任务，trace 文件约 50-100 KB；全量记录 prompt/response 原文时，单条 LLM trace 可达 10-50 KB；一天处理 100 个任务就会产生约 100-500 MB 的 trace 数据。生产环境中需要考虑：

- **采样**：不是每次调用都需要完整记录，可以按比例采样
- **日志级别**：开发阶段全量记录，生产环境只记关键事件
- **存储轮转**：设置保留策略，定期清理过期的 trace 和 snapshot
- **敏感信息脱敏**：prompt 中可能包含用户数据，记录前需要脱敏处理

## 可观测性与其他四个 Harness 要素的关系

| 要素组合 | 记录什么 | 能发现什么 | 优化方向 |
|----------|---------|-----------|---------|
| **可观测性 + 约束** | 约束拦截了什么、多少次、哪些 Agent 最容易触发 | 规则太严导致正常操作被拦、规则太松导致越界未拦 | 调整约束粒度；对高频拦截的 Agent 优化 prompt |
| **可观测性 + 沙箱** | 沙箱的 CPU、内存、磁盘 IO 使用情况 | 无限循环、内存泄漏、资源耗尽 | 设置资源阈值告警（如内存 > 80% 触发主动终止） |
| **可观测性 + 上下文管理** | 上下文使用率曲线、交接次数和交接质量评分 | 每次交接后 Agent 表现下降，说明交接文档有缺失 | 优化交接模板；对上下文利用率持续偏低的任务触发告警 |
| **可观测性 + 反馈回路** | 反馈循环轮次、每轮改进幅度、最终结果（通过/升级人工） | 某类任务反复循环超过 N 轮仍无法通过 | 优化评估标准；对循环超阈值的任务自动升级人工介入 |

## 系列收束

到这里，Harness Engineering 五要素系列全部完成。回顾五个核心要素：

| 要素 | 管什么 | 核心机制 |
|------|-------|---------|
| **约束** | 能做什么 | 阶段门禁 + 文件权限 + 验证卡控 + CI 门禁 |
| **沙箱隔离** | 在哪里做 | 文件系统隔离 + 网络隔离 + 资源限制 + 会话隔离 |
| **上下文管理** | 看到什么 | 配置驱动 + 按需检索 + 结构化交接 + 上下文重置 |
| **反馈回路** | 做得对不对 | 自动验证 + Gen-Eval 分离 + 循环检测 + 经验沉淀 |
| **可观测性** | 发生了什么 | 调用追踪 + 状态快照 + 决策日志 + 指标监控 |

五个要素各管一件事，但彼此依赖：约束定义边界，沙箱保证隔离，上下文管理喂对信息，反馈回路校验质量，可观测性让前四者的运行状态透明可查。缺了任何一个，系统都会有盲区。

不用一次全搭齐。下面是一个分级实施路线，按痛点优先：

| 级别 | 包含什么 | 适用阶段 | 解决什么问题 |
|------|---------|---------|-------------|
| **Level 1** | 基础日志 + 调用链追踪 | 开发调试期 | 出了问题能看到发生了什么 |
| **Level 2** | + 状态快照 + 决策日志 | 多 Agent 协作 / 长任务 | 能回溯任意时刻的完整状态和决策理由 |
| **Level 3** | + 指标聚合 + 告警 | 生产环境 | 系统级健康度监控，异常自动触发告警 |

先从 Level 1 开始，每加一层都能把稳定性拉上一个台阶。如果只读了这一篇，建议从约束篇开始补全——五个要素的落地顺序和阅读顺序一致。

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
