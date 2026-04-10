# Harness 要素之约束：不是写在 prompt 里的建议，而是编进流程的硬卡控

> 给 Agent 配了一堆 rule，它还是会偷偷扩需求、跳过测试、改不该改的文件。问题不在 rule 写得不够多，而在这些 rule 只是"嘱咐"，不是"门禁"。约束这个要素要解决的，就是把嘱咐变成门禁。

## 核心判断

prompt 级 rule 和 Harness 级约束的差距不是程度差异，是机制差异：

- **prompt 级 rule = 口头嘱咐。** 模型大概率会遵守，但没有机制保证它一定遵守。违反了，流程照样往下走。
- **Harness 级约束 = 电子围栏。** 不满足就不让进入下一阶段。Agent 不是"被建议不要越界"，而是根本过不去。

口头嘱咐再多，也抵不过一道物理门禁。

## 没有硬约束的 Agent 会出什么事

先看几个真实场景，都是 prompt 里写了 rule 但实际没拦住的：

| 场景 | prompt 里写了什么 | 实际发生了什么 |
|------|-----------------|---------------|
| 需求边界 | "只做需求范围内的功能" | Agent 顺手加了一个"优化建议"功能，没人要求 |
| 接口契约 | "前后端必须按约定的字段名开发" | 开发 Agent 觉得 `userName` 比 `user_name` 更好，自己改了 |
| 代码安全 | "不要修改 .env 和 secrets 目录" | Agent 为了"修复配置问题"直接写了 .env |
| 测试验证 | "改完代码必须跑测试" | Agent 说"已验证，测试通过"，实际根本没执行测试命令 |

这些不是模型笨。恰恰相反，模型"太聪明"了——它会自作主张地优化、会合理化自己的越界行为、会在上下文压力下走捷径。四个场景的共同特点是：Agent 不是不知道规则，而是没有机制在执行层面拦住它。

prompt 级 rule 管得住常规情况，管不住边界情况。而生产环境里，出事的往往就是边界情况。

## 约束在 Harness 架构中的位置

Harness Engineering 有五个核心要素：约束、沙箱隔离、上下文管理、反馈回路、可观测性。约束是第一个，也是最基础的一个——其他四个要素都建立在"Agent 的行为边界已经被定义清楚"这个前提上。

```text
┌─────────────────────────────────────────┐
│            Harness 约束层               │
│  阶段门禁 │ 工具权限 │ 文件ACL │ 验证卡控  │
├─────────────────────────────────────────┤
│      沙箱 │ 上下文 │ 反馈 │ 可观测       │
├─────────────────────────────────────────┤
│          Workflow（编排层）              │
├─────────────────────────────────────────┤
│          Agent（执行层）                │
└─────────────────────────────────────────┘
```

约束层在最上面，意味着它是所有执行动作的第一道检查。Agent 发出任何动作——调用工具、写文件、切换阶段——都必须先过约束层的门禁。

## 约束的四种落地形态

约束不是一个笼统的概念。在实际工程中，它至少有四种不同的落地形态，每种管的事不一样。

### 1. 阶段门禁：控制任务节奏

复杂任务不应该一口气跑到底。把任务拆成阶段（调研 → 计划 → 执行 → 验证），用状态机控制阶段跳转，是最基本的约束。

关键不在于"建议 Agent 先做计划再执行"，而在于**调研阶段根本拿不到写文件的工具**。

```python
# 编排层代码（由开发者维护，Agent 不可修改）
class PhaseGate:
    """阶段门禁：每个阶段只暴露允许的工具"""

    PHASE_TOOLS = {
        "research": ["read_file", "search_code", "list_files"],
        "plan":     ["read_file", "create_plan"],
        "execute":  ["read_file", "write_file", "run_command"],
        "verify":   ["run_tests", "run_lint", "run_typecheck"],
    }

    ALLOWED_TRANSITIONS = {
        "research": ["plan"],
        "plan":     ["execute", "research"],
        "execute":  ["verify"],
        "verify":   ["execute", "plan"],  # 验证不过可以回退
    }

    def get_tools(self, phase: str) -> list[str]:
        return self.PHASE_TOOLS[phase]

    def can_transition(self, current: str, target: str) -> bool:
        return target in self.ALLOWED_TRANSITIONS.get(current, [])
```

编排层调用大模型时，只把当前阶段允许的工具传进去：

```python
# 编排层在调用 LLM 时只传当前阶段的工具
response = client.chat.completions.create(
    model="claude-sonnet-4-20250514",
    messages=[...],
    tools=gate.get_tools(current_phase),  # 研究阶段就没有 write_file
)
```

Agent 不是"被告知不要在调研阶段改代码"，而是**根本看不到改代码的工具**。这就是硬约束和软约束的本质区别。

### 2. 文件权限：控制 Agent 能碰什么

Agent 能读写文件，就必须有权限边界。高风险文件不能靠 prompt 来保护。

```python
class FileACL:
    """文件访问控制：写在代码里的禁区，不是写在 prompt 里的嘱咐"""

    DENY_WRITE = [
        ".env", ".env.*",
        "secrets/", "config/production/",
        ".git/", ".github/workflows/",
        "package-lock.json", "pnpm-lock.yaml",
    ]

    DENY_READ = [
        "secrets/api-keys.json",
    ]

    def check(self, path: str, operation: str) -> bool:
        deny_list = self.DENY_WRITE if operation == "write" else self.DENY_READ
        for pattern in deny_list:
            if pattern.endswith("/") and path.startswith(pattern):
                return False
            if path == pattern or path.endswith(pattern):
                return False
        return True
```

把这段逻辑放在编排层的工具调用拦截器里，Agent 每次调用 `write_file` 之前都会被检查。不在白名单里的路径，直接拒绝，不给 Agent 解释的机会。

### 3. 验证卡控：防止"嘴上说完成了，实际没做"

Agent 最常见的一类问题是"提前宣布胜利"——声称任务已完成，但实际上没有执行关键步骤。

```python
class CompletionValidator:
    """完成验证：Agent 说完成了，编排层不信，自己检查"""

    def validate(self, agent_output, tool_call_log):
        issues = []

        # 说完成了但没有任何工具调用？
        if agent_output.claims_done and not tool_call_log.has_calls:
            issues.append("声称完成但没有工具调用记录")

        # 改了代码但没跑测试？
        if tool_call_log.has_file_writes and not tool_call_log.has_test_runs:
            issues.append("修改了文件但未运行测试")

        # 说测试通过但没有实际的测试输出？
        if agent_output.claims_tests_passed and not tool_call_log.has_test_output:
            issues.append("声称测试通过但无测试执行记录")

        return issues
```

这段逻辑跑在编排层，不依赖 Agent 的自我报告。Agent 说"我已经测试过了"没用——编排层看的是工具调用日志里有没有真正执行过 `run_tests`。

### 4. 自定义 Linter 规则：让约束变成自动化检查

前三种约束都跑在编排层。第四种更进一步——把项目级别的架构约束编码成 linter 规则，接入 CI 流程，无论是人写的代码还是 Agent 写的代码，都必须通过。

以 ESLint 自定义规则为例。假设项目要求：**所有 API 调用必须通过统一的 `apiClient` 模块，不允许直接使用 `fetch` 或 `axios`**。

```javascript
// eslint-rules/no-direct-fetch.js
module.exports = {
  meta: {
    type: "problem",
    docs: {
      description: "禁止直接使用 fetch/axios，必须通过 apiClient",
      // 关键：错误信息不只是报错，还解释为什么和怎么改
      // 这样 Agent 收到 lint 错误后能自行修正
    },
    messages: {
      noDirectFetch:
        "禁止直接使用 {{ name }}。请使用 lib/apiClient 模块。" +
        "原因：统一错误处理、鉴权和请求日志。" +
        "示例：import { apiClient } from '@/lib/apiClient'",
    },
  },
  create(context) {
    return {
      CallExpression(node) {
        if (node.callee.name === "fetch") {
          context.report({ node, messageId: "noDirectFetch", data: { name: "fetch" } });
        }
      },
      ImportDeclaration(node) {
        if (node.source.value === "axios") {
          context.report({ node, messageId: "noDirectFetch", data: { name: "axios" } });
        }
      },
    };
  },
};
```

配套的 `.eslintrc.js`：

```javascript
// .eslintrc.js
module.exports = {
  rules: {
    "no-direct-fetch": "error",
  },
};
```

配套的 GitHub Actions CI 配置：

```yaml
# .github/workflows/lint.yml
name: Lint Gate
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx eslint --max-warnings 0 'src/**/*.{ts,tsx}'
```

这条 CI 规则的效果是：无论是人还是 Agent 提交的代码，只要直接用了 `fetch` 或 `axios`，PR 就合不进去。Agent 收到 lint 报错后，错误信息里已经写明了为什么不行、该怎么改——这让它能自行修正，形成一个约束 + 反馈的闭环。

## 不写编排层也能落地：用 CodeBuddy / Claude Code 的 Hooks

上面四种约束的代码都是跑在独立编排层里的。但如果用的是 CodeBuddy CLI（v1.16.0+）或 Claude Code，有些约束可以直接通过 **Hooks** 落地，不需要自己写编排层程序。

Hooks 是 CodeBuddy 和 Claude Code 都支持的机制——在 Agent 调用工具之前（`PreToolUse`）或之后（`PostToolUse`）自动执行一段 shell 脚本。脚本可以检查工具参数、拦截调用、甚至修改参数。

有一点需要先说清楚：**Hooks 不是"把脚本放到某个目录就自动生效"的机制。** CodeBuddy 真正识别的是 `.codebuddy/settings.json` 里的 `hooks` 配置，其中 `command` 字段指定要执行的脚本路径。脚本本身放在哪里都行——放 `.codebuddy/hooks/` 下只是一种组织习惯，不是 CodeBuddy 的固定约定。Claude Code 同理，配置文件在 `.claude/settings.json`。换句话说：**写了脚本但没在 `settings.json` 里配置，等于没写。**

先看一张能力对照表，分清哪些能用 Hooks 做、哪些必须自己写编排层：

| 约束类型 | Rules（prompt 级） | Hooks（工具拦截级） | 独立编排层（API 调用级） |
|---------|-------------------|-------------------|----------------------|
| 文件禁写 | 软约束（嘱咐） | **硬约束（拦截）** | 硬约束 |
| 命令拦截 | 软约束 | **硬约束** | 硬约束 |
| 阶段门禁（状态机） | 不支持 | 部分模拟（状态文件） | **原生支持** |
| 动态工具集控制 | 不支持 | 不支持 | **原生支持** |
| 完成验证 | 不支持 | PostToolUse 部分支持 | **原生支持** |
| 搭建成本 | 零（写 markdown） | 低（写 shell 脚本） | 中（写 Python 程序） |

### 用 Hooks 落地文件权限（直接替代 FileACL）

上面那段 `FileACL` 的 Python 代码，在 CodeBuddy 里可以用一个 shell 脚本 + 一行配置替代。

**第一步：写拦截脚本**

```bash
#!/bin/bash
# .codebuddy/hooks/file-acl.sh
# 拦截对敏感文件的写入——效果等同于前面的 FileACL 类

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path')

# 禁写名单
DENY_PATTERNS=(".env" "secrets/" ".git/" "config/production/" "pnpm-lock.yaml")

for pattern in "${DENY_PATTERNS[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    jq -n --arg reason "禁止写入 $FILE_PATH —— 该路径在保护名单中" '{
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason: $reason
      }
    }'
    exit 0
  fi
done

exit 0  # 不在禁写名单，放行
```

**第二步：配置 Hook**

CodeBuddy 的配置文件在 `.codebuddy/settings.json`，Claude Code 在 `.claude/settings.json`，格式一样：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$CODEBUDDY_PROJECT_DIR\"/.codebuddy/hooks/file-acl.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

`matcher` 用正则匹配工具名——`Write|Edit` 表示只要 Agent 调用写文件或编辑文件的工具，就先过这个脚本。脚本返回 `permissionDecision: "deny"` 时，**工具调用被直接拦截**，拒绝原因会反馈给 Agent。

效果和前面的 Python `FileACL` 完全一致：Agent 试图写 `.env`，hook 直接拦死，不是靠 prompt 嘱咐。

### 用 Hooks 落地命令拦截

同样的机制也可以拦截危险命令：

```bash
#!/bin/bash
# .codebuddy/hooks/block-dangerous-commands.sh

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

# 拦截高危命令
if echo "$COMMAND" | grep -qE "rm\s+-rf|drop\s+database|truncate\s+table"; then
  jq -n --arg cmd "$COMMAND" '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: ("危险命令被拦截: " + $cmd)
    }
  }'
  exit 0
fi

exit 0
```

配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$CODEBUDDY_PROJECT_DIR\"/.codebuddy/hooks/block-dangerous-commands.sh"
          }
        ]
      }
    ]
  }
}
```

### 阶段门禁为什么 Hooks 做不到

前面的 `PhaseGate` 涉及三件事：维护全局状态（当前在哪个阶段）、根据阶段动态控制可用工具集、校验阶段跳转合法性。

Hooks 是**无状态的**——每次触发都是独立执行一个 shell 命令，不维护"当前处于 research 还是 execute 阶段"这样的全局状态。更关键的是，Hooks 不能动态改变 Agent 可用的工具列表（只能逐次拦截或放行），而阶段门禁的核心是**研究阶段根本不给 Agent 看到 write_file 这个工具**。

折中方案是用状态文件模拟——hook 脚本每次执行时读取一个文件来判断当前阶段，但 Agent 本身也能修改这个文件，所以不如独立编排层可靠。

如果需要严格的阶段门禁，就得用下面"实战操作"章节里的独立编排层方案。

### 怎么选

- **只需要文件禁写、命令拦截** → 直接用 Hooks，几个 shell 脚本就够
- **需要阶段门禁、动态工具集、完成验证** → 写独立编排层（下面的实战操作章节）
- **两者都要** → 编排层做阶段门禁和完成验证，Hooks 做文件禁写作为额外防线（两层防御比一层靠谱）

## 实战操作：从零搭一个最小约束 Harness（独立编排层方案）

上面 Hooks 能解决文件权限和命令拦截，但阶段门禁和完成验证需要自己写编排层。下面用一个完整的最小示例，把四种约束串起来。场景是：让 Agent 修复一个 bug，但必须按阶段走、不能碰高风险文件、改完必须跑测试。

### 前置准备

```bash
mkdir harness-constraint-demo && cd harness-constraint-demo
python3 -m venv .venv && source .venv/bin/activate
pip install openai  # 或其他 LLM SDK
```

### 编排层完整代码

```python
# orchestrator.py
"""
最小约束 Harness 示例：
- 阶段门禁：research → plan → execute → verify
- 文件权限：.env / secrets / .git 禁写
- 完成验证：改了代码必须跑测试，不跑不算完成
"""

import json
from enum import Enum

# ---------- 阶段门禁 ----------

class Phase(Enum):
    RESEARCH = "research"
    PLAN = "plan"
    EXECUTE = "execute"
    VERIFY = "verify"
    DONE = "done"

PHASE_TOOLS = {
    Phase.RESEARCH: ["read_file", "search_code", "list_files"],
    Phase.PLAN:     ["read_file", "create_plan"],
    Phase.EXECUTE:  ["read_file", "write_file", "run_command"],
    Phase.VERIFY:   ["run_tests", "run_lint"],
}

TRANSITIONS = {
    Phase.RESEARCH: [Phase.PLAN],
    Phase.PLAN:     [Phase.EXECUTE, Phase.RESEARCH],
    Phase.EXECUTE:  [Phase.VERIFY],
    Phase.VERIFY:   [Phase.EXECUTE, Phase.DONE],
}

# ---------- 文件权限 ----------

DENY_WRITE_PATTERNS = [".env", "secrets/", ".git/", "node_modules/"]

def check_file_permission(path: str, operation: str) -> bool:
    if operation != "write":
        return True
    for pattern in DENY_WRITE_PATTERNS:
        if pattern.endswith("/") and path.startswith(pattern):
            return False
        if path == pattern:
            return False
    return True

# ---------- 完成验证 ----------

def validate_completion(tool_calls: list[dict]) -> list[str]:
    has_writes = any(c["tool"] == "write_file" for c in tool_calls)
    has_tests = any(c["tool"] in ("run_tests", "run_lint") for c in tool_calls)
    issues = []
    if has_writes and not has_tests:
        issues.append("修改了文件但未执行测试，请先运行 run_tests")
    return issues

# ---------- 编排主循环 ----------

def run(task: str):
    phase = Phase.RESEARCH
    tool_calls = []

    while phase != Phase.DONE:
        allowed = PHASE_TOOLS.get(phase, [])
        print(f"\n{'='*50}")
        print(f"当前阶段: {phase.value}")
        print(f"可用工具: {allowed}")
        print(f"{'='*50}")

        # 这里替换为真实的 LLM 调用
        # response = client.chat.completions.create(
        #     model="...",
        #     messages=[{"role": "user", "content": task}],
        #     tools=build_tool_definitions(allowed),
        # )

        # 模拟 Agent 的工具调用（实际场景中从 LLM 响应中解析）
        # ...

        # 阶段跳转检查
        next_phase = Phase.PLAN  # 替换为 Agent 请求的下一阶段
        if next_phase not in TRANSITIONS.get(phase, []):
            print(f"拒绝非法跳转: {phase.value} → {next_phase.value}")
            continue

        # 如果是 verify → done，先检查完成条件
        if next_phase == Phase.DONE:
            issues = validate_completion(tool_calls)
            if issues:
                print(f"完成验证失败: {issues}")
                phase = Phase.EXECUTE  # 打回执行阶段
                continue

        phase = next_phase

    print("\n任务完成，所有约束检查通过")

if __name__ == "__main__":
    run("修复 src/login.tsx 中的登录 bug")
```

### 运行效果

```text
==================================================
当前阶段: research
可用工具: ['read_file', 'search_code', 'list_files']
==================================================
# Agent 读取相关文件，分析问题 → 请求进入 plan

==================================================
当前阶段: plan
可用工具: ['read_file', 'create_plan']
==================================================
# Agent 输出修复计划 → 请求进入 execute

==================================================
当前阶段: execute
可用工具: ['read_file', 'write_file', 'run_command']
==================================================
# Agent 修改 src/login.tsx → 请求进入 verify
# 如果 Agent 试图写 .env → 被 check_file_permission 拦截

==================================================
当前阶段: verify
可用工具: ['run_tests', 'run_lint']
==================================================
# Agent 运行测试 → 如果改了代码但没跑测试 → validate_completion 打回
# 测试全部通过 → 允许进入 done

任务完成，所有约束检查通过
```

整个过程中，Agent 的行为被三层约束同时管住：阶段门禁控制节奏、文件权限控制范围、完成验证控制质量。任何一层不满足，流程就走不下去。

## 什么时候需要约束，什么时候不需要

不是所有场景都需要搭这么完整的约束体系。

| 场景 | 是否需要 Harness 级约束 | 原因 |
|------|----------------------|------|
| 写个一次性脚本 | 不需要 | 结果人工验一次就够 |
| 做个 demo 验证思路 | 通常不需要 | prompt 级 rule 就能管住 |
| 多 Agent 协作完成交付物 | 需要 | 任何一个 Agent 越界，整条链都会偏 |
| 输出进入生产环境 | 必须有 | 不可控的风险直接影响用户 |
| 长时间持续运行的流水线 | 必须有 | 累积的小偏差会变成大问题 |

判断标准很简单：**如果 Agent 做错了一步，能接受的修复成本有多高？** 成本低，prompt 级 rule 够用。成本高，就必须上硬约束。

## 约束与其他四个 Harness 要素的关系

约束不是孤立的。它和其他四个要素互为前提、互相加强：

- **约束 + 沙箱隔离**：约束定义"不能做什么"，沙箱保证"做了也影响不到外面"。两层防御比一层靠谱。
- **约束 + 上下文管理**：约束控制行为边界，上下文管理控制信息边界。Agent 只看到该看的、只做被允许做的。
- **约束 + 反馈回路**：约束拦住违规动作，反馈回路把拦截原因告诉 Agent 让它修正。没有反馈的约束只是死墙，有反馈的约束是导航护栏。
- **约束 + 可观测性**：约束执行了多少次、拦截了哪些动作、哪些 Agent 最容易触发拦截——这些数据能帮助判断约束设得对不对、够不够。

五个要素是一个整体。约束是基座，但只有约束也不够。后面四篇会分别展开。

## 延伸阅读

- Harness Engineering 的整体框架 → 《Harness Engineering 不是工作流：一个管编排一个管兜底》
- 沙箱隔离 → 《Harness 要素之沙箱隔离：让 Agent 只能在围栏里干活》
- 上下文管理 → 《Harness 要素之上下文管理：Agent 跑偏的根源是喂错了信息》
- 反馈回路 → 《Harness 要素之反馈回路：不是跑完就算，而是跑歪了能自动拉回来》
- 可观测性 → 《Harness 要素之可观测性：Agent 在想什么，必须看得见》

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
