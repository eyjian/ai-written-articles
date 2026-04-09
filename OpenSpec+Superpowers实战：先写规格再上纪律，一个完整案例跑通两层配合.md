# OpenSpec + Superpowers 实战：先写规格再上纪律，一个完整案例跑通两层配合

> OpenSpec 管"做什么"，Superpowers 管"怎么做"。只有 OpenSpec，spec 写得再漂亮，Agent 照样走野路子；只有 Superpowers，纪律再好，需求一漂所有代码白写。两层配合才能堵住从规格到代码之间最容易断裂的那个环节。本文用一个完整案例，从写 spec 到 TDD 实现到验证闭环，把两层怎么配合跑一遍。

有人跟我说：他用 OpenSpec 写了一份详细的接口规格，字段名、返回格式、验收标准全定好了。让 Agent 按 spec 写代码，Agent 确实读了——但写出来的代码接口路径改了、字段名偷偷换了、验收标准里的边界校验一条没实现。"spec 不是白写了吗？"

问题不在 spec，而在**spec 和代码之间没有纪律约束**。Agent 读了 spec，但开发过程中没有任何机制"逼"它按 spec 来——不写测试就直接上手，不跑验证就宣称完成。

这就是 OpenSpec 和 Superpowers 需要配合的原因：**OpenSpec 定规格，Superpowers 逼执行。** 缺了前者，纪律没有锚点；缺了后者，规格没有执行力。

---

## 先给判断

赶时间就带走这两句：

- **OpenSpec 解决"做什么"的对齐问题**——需求、接口、验收标准写成结构化文档，所有 Agent 共享同一份规格。
- **Superpowers 解决"怎么做"的纪律问题**——TDD、计划拆解、代码审查、完成前验证，让 Agent 按工程最佳实践执行。

两层的关系用一句话说：**OpenSpec 是施工图纸，Superpowers 是施工规范手册。图纸告诉你盖什么样的楼，规范手册告诉你怎么盖才不塌。**

---

## 一、先看问题：只用其中一个，哪里会断

### 只有 OpenSpec，没有 Superpowers

你把 spec 写得很详细：接口路径 `/api/password/check`，入参 `password: string`，返回 `{ strength: "weak" | "medium" | "strong", reasons: string[] }`，验收标准包括 5 条边界规则。

Agent 读了 spec，然后——

- **直接上手写实现**，没有先写测试。写完的代码"看起来对"，但验收标准里 5 条只满足了 3 条。
- **接口字段名偷偷改了**。spec 里写的是 `strength`，Agent 觉得 `level` 更好听，就自己换了。
- **说"完成了"就交差**。你问它"验收标准都通过了吗"，它说"是的"——但其实它根本没跑验证。

这不是 Agent 不聪明，而是**它的开发流程里没有强制环节来确保它按 spec 来**。它读了 spec，但读了不等于遵守了。

### 只有 Superpowers，没有 OpenSpec

你给 Agent 加载了 TDD 技能、代码审查技能、完成前验证技能。纪律拉满。

Agent 确实先写了测试——但测试是它自己根据对需求的理解写的。你的需求是"密码长度不够 8 位就算 weak"，Agent 理解成"密码不含数字就算 weak"。测试通过了，代码也通过了，但和你要的东西完全不一样。

**纪律是有了，但纪律的锚点漂了。** 因为没有一份结构化的 spec 把需求锁死，Agent 的 TDD 是在对着自己理解的需求做驱动，不是对着你确认过的需求。

### 一张表看清各自盲区

| 场景 | OpenSpec 能管住吗 | Superpowers 能管住吗 |
|------|:---:|:---:|
| 需求漂移（Agent 自己改了需求） | ✅ spec 里锁死了 | ❌ 纪律管不住需求本身 |
| 接口字段不一致 | ✅ spec 里定义了 | ❌ TDD 只管测试通不通过 |
| 开发过程跳步骤（不写测试直接写代码） | ❌ spec 不管开发流程 | ✅ TDD 强制先写测试 |
| 代码写完没验证就说"完成了" | ❌ spec 不管执行纪律 | ✅ verification 逼它先验再交 |
| 验收标准不明确 | ✅ spec 里有验收标准 | ❌ Superpowers 没有需求定义功能 |
| 测试只测了开心路径 | ❌ spec 只定义标准，不管测试覆盖 | ✅ TDD + 审查会检查覆盖 |

这张表的信号很清楚：**两者各管一半，重叠很少，互补很强。**

---

## 二、两层怎么配合：从规格到代码的完整链路

把两层放在一起，一个完整的开发周期是这样的：

```text
┌─────────────────────────────────────────────────┐
│  Phase 1：规格定义（OpenSpec 主导）                │
│  propose → spec → 锁定需求、接口、验收标准        │
└────────────────────┬────────────────────────────┘
                     │ spec 文档作为输入
                     ▼
┌─────────────────────────────────────────────────┐
│  Phase 2：需求澄清（Superpowers 的 brainstorming） │
│  Agent 对着 spec 提问，确认边界和歧义              │
└────────────────────┬────────────────────────────┘
                     │ 澄清后的 spec
                     ▼
┌─────────────────────────────────────────────────┐
│  Phase 3：任务拆解（Superpowers 的 writing-plans） │
│  把 spec 里的验收标准拆成可执行的开发步骤          │
└────────────────────┬────────────────────────────┘
                     │ 任务计划
                     ▼
┌─────────────────────────────────────────────────┐
│  Phase 4：TDD 实现（Superpowers 的 TDD）          │
│  对着 spec 的验收标准写测试 → 写实现 → 跑测试     │
└────────────────────┬────────────────────────────┘
                     │ 代码 + 通过的测试
                     ▼
┌─────────────────────────────────────────────────┐
│  Phase 5：验证闭环（OpenSpec 的 verify）           │
│  拿 spec 的验收标准逐条核对实际产出               │
│  通过 → 归档；不通过 → 打回 Phase 4               │
└─────────────────────────────────────────────────┘
```

**关键交接点**：

- **Phase 1 → Phase 2**：spec 文档是 brainstorming 的输入。Agent 不是凭空猜需求，而是对着一份明确的 spec 提问。
- **Phase 3 → Phase 4**：任务拆解时，每个步骤都要对应 spec 里的某条验收标准。不是"我觉得该先做什么"，而是"spec 要求什么，我就先测什么"。
- **Phase 4 → Phase 5**：TDD 写的测试来自 spec 的验收标准，verify 对的也是这套标准。**锚点始终是同一份 spec，不会漂。**

---

## 三、实战案例：用 OpenSpec + Superpowers 做一个密码强度检测模块

这个案例故意选得很小——一个密码强度检测函数。小到一个人半小时能手写完，但足够把两层配合的每一步都跑到。

### Step 1：用 OpenSpec 写 spec

先创建 spec 文件，把需求、接口和验收标准锁死。这里用 YAML 做简化演示——实际 OpenSpec 有自己的 spec 模板和 CLI 工具链（`openspec propose`、`openspec spec`、`openspec verify`），格式和目录结构可能不完全一样，但核心思路一致：

```yaml
# openspec/specs/password-checker.yaml

title: 密码强度检测模块
status: approved

description: |
  提供一个函数，输入密码字符串，返回强度等级和判定原因。
  用于注册页面的前端实时提示和后端二次校验。

interface:
  function: check_strength
  input:
    - name: password
      type: string
      required: true
  output:
    strength: "weak" | "medium" | "strong"
    reasons: string[]

acceptance_criteria:
  - id: AC-1
    description: 密码长度小于 8 位，判定为 weak
    example:
      input: "abc"
      expected_strength: weak
      expected_reason_contains: "长度不足"

  - id: AC-2
    description: 密码只包含字母，判定为 medium
    example:
      input: "abcdefgh"
      expected_strength: medium
      expected_reason_contains: "缺少数字"

  - id: AC-3
    description: 密码只包含数字，判定为 medium
    example:
      input: "12345678"
      expected_strength: medium
      expected_reason_contains: "缺少字母"

  - id: AC-4
    description: 密码包含字母和数字但长度不足 12 位，判定为 medium
    example:
      input: "abc12345"
      expected_strength: medium
      expected_reason_contains: "长度未达到强密码标准"

  - id: AC-5
    description: 密码包含字母和数字且长度 ≥ 12 位，判定为 strong
    example:
      input: "myPassword12"
      expected_strength: strong
```

这份 spec 做了三件关键的事：

1. **接口锁死了**——函数名、参数名、返回结构不容商量
2. **验收标准锁死了**——5 条，每条带输入输出示例
3. **边界明确了**——什么算 weak、什么算 medium、什么算 strong，不留解释空间

### Step 2：用 Superpowers 的 brainstorming 澄清边界

在让 Agent 动手之前，先加载 brainstorming 技能，让它对着 spec 提问。Agent 可能会问很多问题，这里列两个最关键的：

```text
Agent 提问：
1. spec 里没有提到特殊字符（如 !@#$），包含特殊字符是否影响强度判定？
2. 空字符串怎么处理？返回 weak 还是直接报错？

你的回答：
1. 当前版本不考虑特殊字符，后续迭代再加。
2. 空字符串返回 weak，reasons 里写"密码为空"。
```

这一步的价值：**Agent 不再是按自己的理解猜，而是对着 spec 里写了的和没写的分别确认。** 确认完的边界回填到 spec 的补充说明里。

### Step 3：用 Superpowers 的 writing-plans 拆解任务

Agent 加载 writing-plans 技能后，对着 spec 的 5 条验收标准拆出开发计划：

```text
开发计划：

任务 1：搭建项目结构
  - 创建 password_checker.py 和 tests/test_password.py
  - 确认测试框架（pytest）

任务 2：实现 AC-1（长度不足 8 位 → weak）
  - 写失败测试 → 最小实现 → 跑测试

任务 3：实现 AC-2（只有字母 → medium）
  - 写失败测试 → 最小实现 → 跑测试

任务 4：实现 AC-3（只有数字 → medium）
  - 写失败测试 → 最小实现 → 跑测试

任务 5：实现 AC-4（字母+数字但长度不足 12 位 → medium）
  - 写失败测试 → 最小实现 → 跑测试

任务 6：实现 AC-5（字母+数字且长度 ≥ 12 位 → strong）
  - 写失败测试 → 最小实现 → 跑测试

任务 7：补充边界用例
  - 空字符串 → weak
  - 多条 reasons 同时命中

任务 8：代码审查 + 完成前验证
```

注意这份计划的特点：**每个任务直接对应 spec 里的一条验收标准。** 不是 Agent 自己想到什么做什么，而是 spec 说要验什么，计划就先测什么。

### Step 4：用 Superpowers 的 TDD 实现代码

按计划逐条推进。以 AC-1 为例：

**RED——先写一个会失败的测试：**

```python
# tests/test_password.py
def test_ac1_short_password_is_weak():
    """AC-1: 密码长度小于 8 位，判定为 weak"""
    from password_checker import check_strength
    result = check_strength("abc")
    assert result["strength"] == "weak"
    assert any("长度不足" in r for r in result["reasons"])
```

跑一下。失败了——因为 `password_checker.py` 还不存在。这就对了。

**GREEN——写最小实现让测试通过：**

```python
# password_checker.py
def check_strength(password: str) -> dict:
    reasons = []

    if len(password) < 8:
        reasons.append("长度不足 8 位")
        return {"strength": "weak", "reasons": reasons}

    return {"strength": "medium", "reasons": reasons}
```

再跑测试。通过了。

**继续推进 AC-2 到 AC-5**，每一条都是同样的节奏：先写失败测试（直接从 spec 的 example 翻译过来）→ 最小实现 → 跑通 → 下一条。

最终实现可能是这样的：

```python
# password_checker.py
def check_strength(password: str) -> dict:
    """优先级顺序直接来自 spec 的验收标准编号：AC-1 到 AC-5"""
    reasons = []

    # AC-边界：空字符串（brainstorming 阶段确认的补充规则）
    if not password:
        return {"strength": "weak", "reasons": ["密码为空"]}

    # AC-1：长度不足 8 位，直接判 weak（最高优先级）
    if len(password) < 8:
        reasons.append("长度不足 8 位")
        return {"strength": "weak", "reasons": reasons}

    # AC-2 / AC-3：检查字符类型
    has_letter = any(c.isalpha() for c in password)
    has_digit = any(c.isdigit() for c in password)

    if not has_digit:
        reasons.append("缺少数字")
    if not has_letter:
        reasons.append("缺少字母")

    if not has_letter or not has_digit:
        return {"strength": "medium", "reasons": reasons}

    # AC-4：字母+数字都有，但长度不足 12 位
    if len(password) < 12:
        reasons.append("长度未达到强密码标准（建议 12 位以上）")
        return {"strength": "medium", "reasons": reasons}

    # AC-5：字母+数字且长度 ≥ 12 位
    return {"strength": "strong", "reasons": reasons}
```

### Step 5：用 OpenSpec 的 verify 验证产出

代码写完、测试全绿以后，用 OpenSpec 的 verify 流程逐条核对：

```text
验证报告：

AC-1 ✅ 输入 "abc" → strength: weak, reasons 包含 "长度不足"
AC-2 ✅ 输入 "abcdefgh" → strength: medium, reasons 包含 "缺少数字"
AC-3 ✅ 输入 "12345678" → strength: medium, reasons 包含 "缺少字母"
AC-4 ✅ 输入 "abc12345" → strength: medium, reasons 包含 "长度未达到强密码标准"
AC-5 ✅ 输入 "myPassword12" → strength: strong

接口校验：
  - 函数名 check_strength ✅
  - 入参 password: string ✅
  - 返回结构 { strength, reasons } ✅

结论：全部通过，归档。
```

这一步的价值：**验证的锚点是 spec，不是 Agent 自己的判断。** 它不是"觉得做完了"就完了，而是对着 spec 逐条打勾。

### Step 6：闭环——如果不通过怎么办

假设 AC-4 没通过——输入 `"abc12345"` 返回了 `strong` 而不是 `medium`。

verify 报告里会标红：

```text
AC-4 ❌ 输入 "abc12345" → 期望 strength: medium，实际 strength: strong
```

这时候不需要人介入判断"到底哪错了"。流程自动打回 Phase 4：

1. Agent 拿到验证失败报告
2. 定位到 AC-4 对应的测试——发现测试本身写对了，但实现逻辑里漏了长度 < 12 的判断
3. 修复实现 → 跑测试 → 通过 → 重新 verify → 全部通过 → 归档

**这就是闭环的意义：verify 不只是"走个流程"，而是整条链路的最后一道门禁。不通过就不算完。**

---

## 四、两层配合的操作检查清单

每次用 OpenSpec + Superpowers 做开发，对着这张清单过一遍：

| # | 检查项 | 哪一层负责 | 判断标准 |
|---|--------|:---------:|---------|
| 1 | spec 文件存在且状态为 approved | OpenSpec | 有文件、有需求、有接口、有验收标准 |
| 2 | 验收标准每一条都有输入输出示例 | OpenSpec | 可以直接翻译成测试用例 |
| 3 | Agent 在写代码前做了 brainstorming | Superpowers | 有提问记录、有边界确认 |
| 4 | 开发计划里每个任务对应 spec 的某条标准 | Superpowers | 计划和 spec 有明确映射 |
| 5 | 每条验收标准都有对应的测试用例 | Superpowers | 测试文件里有 AC-1 到 AC-N 的测试 |
| 6 | 测试是先写后实现（RED → GREEN） | Superpowers | 测试先失败、再通过 |
| 7 | 代码写完做了代码审查 | Superpowers | 有审查记录 |
| 8 | 最终用 spec 逐条 verify | OpenSpec | 验证报告里每条标准都有 ✅ 或 ❌ |
| 9 | verify 不通过的项已打回并修复 | 两层配合 | 重新跑通测试 + 重新 verify |

---

## 五、什么时候该用这套组合，什么时候没必要

不是所有任务都值得同时上两层。

| 场景 | 建议 |
|------|------|
| 写个一次性脚本 | 不需要。单 Agent + 简单 prompt 就够 |
| 做个原型验证 | 可能不需要 OpenSpec，但值得用 Superpowers 的 TDD |
| 开发一个要上线的模块 | 建议两层都用。spec 锁需求，TDD 锁质量 |
| 多 Agent 协作开发一个完整项目 | 强烈建议。没有 spec 多 Agent 各做各的，没有纪律代码质量不可控 |
| 需求本身就很模糊、还在探索阶段 | 先用 brainstorming 梳理，梳理完再决定要不要写正式 spec |

判断标准也可以简化：**如果你的任务"做错了返工成本很高"，就值得两层都上。**

---

## 六、常见错误

### 错误 1：spec 写了但 Agent 没真正读

最常见的情况：spec 文件放在 `openspec/specs/` 目录里，但 Agent 启动时的上下文里根本没加载它。Agent 是对着你聊天窗口里的口头描述在干活，spec 文件只是静静地躺在那。

**怎么避免**：在 Agent 的启动 prompt 或 Harness 配置里，明确指定"先读 `openspec/specs/` 目录下的 spec 文件，再开始任何开发动作"。不是建议，是硬约束。

### 错误 2：TDD 写了但没对着 spec 的验收标准写

Agent 确实先写了测试，但测试是它自己编的——不是从 spec 的验收标准翻译过来的。结果测试全绿，但和 spec 要求的不是一回事。

**怎么避免**：在任务拆解阶段就把映射关系定死——每个任务对应 spec 里的哪条验收标准，测试用例直接从 spec 的 example 翻译。

### 错误 3：verify 走了个形式

Agent 跑了 verify，报告里写着"全部通过"。你一看，它只验了 5 条里的 3 条，另外 2 条直接跳过了。

**怎么避免**：verify 的模板必须包含 spec 里的**所有**验收标准 ID。少一条就不算完成。一个简单的检测方法：从 spec 文件解析出 AC-1 到 AC-N 的完整列表，再和验证报告里出现的 AC-ID 做 diff——缺了哪条一目了然。如果工具层支持自动化，可以做成脚本：`spec 里有 5 条标准，报告里只验了 3 条 → 自动标记为未完成`。

---

## 结论

OpenSpec 和 Superpowers 解决的是同一条链路上的不同环节：**OpenSpec 把"做什么"锁死，Superpowers 把"怎么做"管住。** 只有 OpenSpec 的时候，规格写了但执行靠运气；只有 Superpowers 的时候，纪律有了但锚点会漂。

两层配合的核心就一句话：**spec 定义验收标准，TDD 对着验收标准写测试，verify 对着验收标准做验证。三个环节的锚点是同一份文档，不会漂。**

如果你目前最大的痛点是"Agent 写的代码和需求对不上"，先把 spec 写起来、把 TDD 跑起来，很多对不上的问题就会先少一半。用不着一步到位搭完整平台——从一个小模块开始，两层配合跑一遍闭环，感受到收益再往更大的项目推。

## 延伸阅读

这篇聚焦的是 OpenSpec + Superpowers 的两层配合。如果想了解更完整的工程化体系，可以看：

- OpenSpec、Superpowers 和 Harness 三层各管什么 → 《OpenSpec、Superpowers 和 Harness：AI 工程化开发的三层拼图》
- Superpowers 的完整技能体系和在 CodeBuddy 里怎么借 → 《Superpowers 是什么：它不是"给 AI 加智商"，而是给 AI 编程加纪律》
- Superpowers 和 Harness Engineering 的抽象层级差异 → 《Superpowers 和 Harness Engineering，到底差在哪》
- 用 CodeBuddy 搭最小 Harness 闭环 → 《别让 AI 裸奔：用 CodeBuddy 搭一个 Markdown 质检 Harness》

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
