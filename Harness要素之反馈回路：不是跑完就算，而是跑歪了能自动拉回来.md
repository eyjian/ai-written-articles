# Harness 要素之反馈回路：不是跑完就算，而是跑歪了能自动拉回来

> Agent 写完代码说"已完成"，但测试没跑、lint 没过、接口对不上。你靠人去检查，一次两次还行，十次百次就扛不住了。反馈回路要解决的，就是让系统自动发现 Agent 跑歪了，自动告诉它哪里错了，自动让它修——人只管最后兜底。

## 核心判断

一句话分清：

- **没有反馈回路 = 考试不批卷只交卷。** Agent 做完就算完，对不对全凭运气。
- **有反馈回路 = 交卷后自动批改、打回重做、直到及格。** Agent 每一步的输出都被验证，不合格就重来。

人工 Code Review 是反馈回路的一种，但它太慢、太贵、不可持续。Harness 级的反馈回路是自动化的——机器验证、机器反馈、机器驱动重试。

## 没有反馈回路会出什么事

| 问题 | 表现 | 根因 |
|------|------|------|
| 提前宣布胜利 | Agent 说"测试通过"，实际没执行测试命令 | 没有验证机制，Agent 的自我报告不可信 |
| 错误累积 | 第一步的小错误传递到后续步骤，最后结果全偏 | 每一步的输出没有被检查就直接作为下一步的输入 |
| 反复撞墙 | Agent 失败后重试同样的方法，连续失败 3-5 次 | 没有"换条路试试"的机制 |
| 质量退化 | 前几次输出还行，后面越来越差 | 没有持续的质量信号，Agent 无从校准 |

核心矛盾是：Agent 有很强的生成能力，但自我评估能力很弱。让 Agent 自己判断自己做得好不好，就像让学生自己给自己打分——大概率会"放水"。

## 反馈回路在 Harness 架构中的位置

约束管"能做什么"，沙箱管"在哪里做"，上下文管"看到什么"，反馈回路管"做得对不对"。它是 Agent 执行质量的最后一道自动化防线。

```text
┌─────────────────────────────────────────┐
│  约束 │ 沙箱 │ 上下文管理                 │
├─────────────────────────────────────────┤
│         反馈回路（质量闭环）               │
│  自动验证 │ 执行-评审分离 │ 循环检测 │ 经验沉淀 │
├─────────────────────────────────────────┤
│            可观测性                       │
├─────────────────────────────────────────┤
│          Agent（执行层）                │
└─────────────────────────────────────────┘
```

## 反馈回路的四种落地形态

### 1. 自动化验证：最低成本、最高收益的第一层

最基础的反馈回路：Agent 改了代码 → 自动跑 typecheck / lint / test → 不过就把错误信息反馈给 Agent → Agent 修复 → 再验证 → 直到通过或升级人工。

```python
# auto_verify.py
"""自动化验证：Agent 每次修改代码后自动触发"""

import subprocess

class AutoVerifier:
    CHECKS = [
        ("typecheck", "pnpm typecheck"),
        ("lint", "pnpm lint"),
        ("test", "pnpm test --run"),
    ]

    def verify(self, changed_files: list[str]) -> dict:
        results = {}
        all_passed = True

        for name, command in self.CHECKS:
            result = subprocess.run(
                command.split(),
                capture_output=True, text=True, timeout=120,
            )
            passed = result.returncode == 0
            results[name] = {
                "passed": passed,
                "stdout": result.stdout[-1000:] if passed else "",
                "stderr": result.stderr[-1000:] if not passed else "",
            }
            if not passed:
                all_passed = False

        return {"all_passed": all_passed, "checks": results}

    def format_feedback(self, results: dict) -> str:
        """把验证结果格式化成 Agent 能理解的反馈"""
        if results["all_passed"]:
            return "所有检查通过。"

        feedback_parts = ["以下检查未通过，请修复后重试：\n"]
        for name, check in results["checks"].items():
            if not check["passed"]:
                feedback_parts.append(f"## {name} 失败")
                feedback_parts.append(f"```\n{check['stderr']}\n```")
        return "\n".join(feedback_parts)
```

在编排层里接入：

```python
# 编排层在 Agent 完成 execute 阶段后自动触发验证
verifier = AutoVerifier()

def post_execute(agent, modified_files):
    results = verifier.verify(modified_files)

    if results["all_passed"]:
        return "verify_passed"

    # 不过就反馈给 Agent，让它修
    feedback = verifier.format_feedback(results)
    agent.receive_feedback(feedback)
    return "retry_execute"
```

这一步看起来简单，但它消灭了最大的一类问题："Agent 说完成了但实际没验证过"。

### 2. Generator-Evaluator 分离：让一个 Agent 干活，另一个 Agent 找茬

Agent 自己评价自己的工作，天然倾向于"宽容自夸"。更可靠的做法是把生成和评估拆成两个独立的 Agent。

Anthropic 在 2026 年 3 月的博客中详细描述了这种模式：Generator 负责生成代码，Evaluator 负责审查，两者形成对抗循环。

```python
# gen_eval_loop.py
"""Generator-Evaluator 对抗循环"""

class GeneratorEvaluatorLoop:
    MAX_ITERATIONS = 5

    def __init__(self, generator, evaluator):
        self.generator = generator
        self.evaluator = evaluator

    async def run(self, task: str) -> dict:
        for i in range(self.MAX_ITERATIONS):
            # Generator 生成
            output = await self.generator.generate(task)

            # Evaluator 评审（独立会话，不共享 Generator 的上下文）
            review = await self.evaluator.evaluate(
                task_description=task,
                output=output,
                criteria=[
                    "功能是否完整实现",
                    "是否有明显 bug",
                    "是否遵循项目规范",
                    "边界情况是否处理",
                ],
            )

            print(f"第 {i+1} 轮: 得分 {review.score}/10")

            if review.score >= 8:
                return {"output": output, "iterations": i + 1, "final_review": review}

            # 评审不通过，把反馈给 Generator
            task = f"{task}\n\n上一轮评审反馈：\n{review.feedback}"

        # 超过最大轮次，升级人工
        return {"output": output, "iterations": self.MAX_ITERATIONS,
                "escalated": True, "final_review": review}
```

关键点：
- Evaluator 用独立会话，不和 Generator 共享上下文——避免"互相包庇"
- 评审标准是结构化的，不是让 Evaluator "看看怎么样"
- 超过最大轮次自动升级人工，不会无限循环

### 3. 循环失败检测：同一条路走不通，强制换方向

Agent 还有一个常见问题：失败后重试同样的方法，连续撞墙。传统后端的熔断器思路在这里完全适用。

```python
# loop_detector.py
"""循环失败检测：连续失败就强制换路径"""

from collections import defaultdict

class LoopDetector:
    def __init__(self, max_same_failures: int = 3):
        self.max_same_failures = max_same_failures
        self.failure_log = defaultdict(list)

    def record_failure(self, action: str, error: str):
        self.failure_log[action].append(error)

    def should_force_pivot(self, proposed_action: str) -> bool:
        """连续相同失败超过阈值，强制换方向"""
        return len(self.failure_log[proposed_action]) >= self.max_same_failures

    def get_pivot_suggestion(self, failed_action: str) -> str:
        errors = self.failure_log[failed_action]
        return (
            f"动作 '{failed_action}' 已连续失败 {len(errors)} 次。"
            f"最近一次错误：{errors[-1]}\n"
            f"请换一种方法。禁止再使用相同的方法重试。"
        )
```

在编排层接入：

```python
detector = LoopDetector(max_same_failures=3)

def handle_agent_action(agent, action, result):
    if result.failed:
        detector.record_failure(action, result.error)

        if detector.should_force_pivot(action):
            # 强制换方向
            suggestion = detector.get_pivot_suggestion(action)
            agent.receive_feedback(suggestion)
            return "force_pivot"

        # 还没到阈值，允许重试
        agent.receive_feedback(f"执行失败：{result.error}\n请修复后重试。")
        return "retry"

    return "continue"
```

### 4. 经验沉淀：每次犯错都变成系统改进

反馈回路最有价值的一层：把 Agent 的错误转化为持久化的经验，让同类问题不再重复发生。

```python
# lessons.py
"""经验沉淀：Agent 犯的错变成系统的改进"""

import json
from datetime import date
from pathlib import Path

class LessonsLearned:
    def __init__(self, path: str = ".harness/lessons.json"):
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self.lessons = self._load()

    def _load(self) -> list:
        if self.path.exists():
            return json.loads(self.path.read_text())
        return []

    def add(self, error: str, root_cause: str, fix: str, status: str = "pending"):
        self.lessons.append({
            "date": str(date.today()),
            "error": error,
            "root_cause": root_cause,
            "fix": fix,
            "status": status,  # pending / applied_to_agents_md / applied_to_ci
        })
        self.path.write_text(json.dumps(self.lessons, ensure_ascii=False, indent=2))

    def get_relevant_lessons(self, context: str) -> list:
        """给 Agent 的上下文里注入相关的历史教训"""
        return [l for l in self.lessons if l["status"] != "pending"]
```

形成闭环：

```text
Agent 犯错
→ 自动化验证捕获错误
→ 归因分析（是缺信息？缺约束？缺验证？）
→ 写入 lessons.json
→ 更新 AGENTS.md 或 CI 规则
→ 下次同类错误自动避免
```

## 实战操作：用 Playwright MCP 搭一个端到端反馈回路

自动化验证不只是跑 lint 和测试。对于 Web 应用，最有力的反馈是**像真实用户一样操作界面**。Anthropic 在实践中使用 Playwright MCP 让 Evaluator Agent 直接在浏览器里点击、输入、截图来验证功能。

下面搭一个最小的端到端验证反馈回路。

### 前置准备

```bash
mkdir harness-feedback-demo && cd harness-feedback-demo
npm init -y
npm install playwright @playwright/test
npx playwright install chromium
```

### 端到端验证脚本

```javascript
// e2e-verify.js
/**
 * 端到端验证：像真实用户一样操作页面，验证功能是否正常
 * Agent 修改代码后，编排层自动调用这个脚本
 */

const { chromium } = require("playwright");

async function verifyLogin(baseUrl) {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const results = [];

  try {
    // 测试 1：页面能正常加载
    await page.goto(`${baseUrl}/login`);
    const title = await page.title();
    results.push({
      test: "页面加载",
      passed: title.includes("Login"),
      detail: `页面标题: ${title}`,
    });

    // 测试 2：空表单提交应该显示错误
    await page.click('button[type="submit"]');
    const errorMsg = await page.textContent(".error-message").catch(() => null);
    results.push({
      test: "空表单验证",
      passed: errorMsg !== null,
      detail: errorMsg || "没有显示错误提示",
    });

    // 测试 3：正确凭证应该跳转
    await page.fill('input[name="email"]', "test@example.com");
    await page.fill('input[name="password"]', "password123");
    await page.click('button[type="submit"]');
    await page.waitForURL("**/dashboard", { timeout: 5000 }).catch(() => {});
    const currentUrl = page.url();
    results.push({
      test: "登录跳转",
      passed: currentUrl.includes("/dashboard"),
      detail: `当前URL: ${currentUrl}`,
    });

    // 截图留证
    await page.screenshot({ path: "output/login-test-result.png" });

  } finally {
    await browser.close();
  }

  return results;
}

async function main() {
  const baseUrl = process.argv[2] || "http://localhost:3000";
  const results = await verifyLogin(baseUrl);

  const allPassed = results.every((r) => r.passed);
  const output = { all_passed: allPassed, tests: results };

  console.log(JSON.stringify(output, null, 2));
  process.exit(allPassed ? 0 : 1);
}

main().catch((e) => {
  console.error(JSON.stringify({ error: e.message }));
  process.exit(2);
});
```

### 编排层集成

```python
# feedback_orchestrator.py
"""集成端到端验证的反馈编排"""

import subprocess
import json

class E2EFeedbackLoop:
    MAX_FIX_ATTEMPTS = 3

    def run_e2e_verify(self, base_url: str = "http://localhost:3000") -> dict:
        result = subprocess.run(
            ["node", "e2e-verify.js", base_url],
            capture_output=True, text=True, timeout=30,
        )
        try:
            return json.loads(result.stdout)
        except json.JSONDecodeError:
            return {"all_passed": False, "error": result.stderr}

    def feedback_loop(self, agent, base_url: str):
        for attempt in range(self.MAX_FIX_ATTEMPTS):
            results = self.run_e2e_verify(base_url)

            if results.get("all_passed"):
                print(f"端到端验证通过（第 {attempt + 1} 次尝试）")
                return True

            # 构建反馈
            failed_tests = [t for t in results.get("tests", []) if not t["passed"]]
            feedback = "端到端验证未通过：\n"
            for t in failed_tests:
                feedback += f"- {t['test']}：{t['detail']}\n"
            feedback += "\n请修复以上问题后重试。"

            print(f"第 {attempt + 1} 次尝试失败，反馈给 Agent...")
            agent.receive_feedback(feedback)

        print("超过最大尝试次数，升级人工处理")
        return False
```

### 运行效果

```text
第 1 次尝试失败，反馈给 Agent...
端到端验证未通过：
- 空表单验证：没有显示错误提示
- 登录跳转：当前URL: http://localhost:3000/login

Agent 修复中...

第 2 次尝试失败，反馈给 Agent...
端到端验证未通过：
- 登录跳转：当前URL: http://localhost:3000/login

Agent 修复中...

端到端验证通过（第 3 次尝试）
```

Agent 不是靠"我觉得修好了"来判断，而是靠真实的浏览器操作来验证。页面没跳转就是没修好，不管 Agent 怎么说。

## 什么时候需要反馈回路，什么时候不需要

| 场景 | 需不需要 | 原因 |
|------|---------|------|
| 一次性文本生成 | 通常不需要 | 人看一眼就能判断 |
| 代码生成（需要运行） | 需要 | lint + test 是最基本的反馈 |
| 多步骤任务 | 需要 | 每步的输出都应该被验证后再进入下一步 |
| 面向用户的功能 | 需要 | 端到端验证比单元测试更真实 |
| 持续运行的 Agent 系统 | 必须有 | 没有自动反馈，错误会无限累积 |

判断标准：**Agent 的输出能不能被机器自动验证？** 能被验证的，就接自动反馈。不能被验证的（比如创意类任务），至少要有人工检查节点。

## 反馈回路与其他四个 Harness 要素的关系

- **反馈回路 + 约束**：约束是事前拦截（不让做），反馈回路是事后校验（做了检查对不对）。前者防越界，后者防质量滑坡。
- **反馈回路 + 沙箱**：Agent 在沙箱里的执行结果（退出码、日志、截图）是天然的反馈信号。沙箱提供了一个安全的验证环境。
- **反馈回路 + 上下文管理**：反馈信息本身需要被管理——哪些反馈要保留在上下文里，哪些可以丢弃，历史教训怎么注入新会话。
- **反馈回路 + 可观测性**：反馈回路的执行过程（验证了几次、每次的结果、最终是通过还是升级人工）就是可观测数据。没有可观测性，你不知道反馈回路本身有没有在正常工作。

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
