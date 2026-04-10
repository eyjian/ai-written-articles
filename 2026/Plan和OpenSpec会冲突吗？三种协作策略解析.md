# Plan和OpenSpec会冲突吗？三种协作策略解析

问：确认下，执行采用openspec还是什么？我看到了你的plan，会同openspec的task.md冲突吗？

答：好问题。让我理清这两者的关系。

---

## Plan vs OpenSpec：它们是什么关系？

```
┌─────────────────────────────────────────────────────────────┐
│                     两个系统的定位                            │
│                                                             │
│  CodeBuddy Plan（当前 plan）          OpenSpec               │
│  ─────────────────────                ────────               │
│  • IDE 内置的任务管理                  • 你项目里安装的 skill  │
│  • 跟踪"做了没"                       • 管理"设计决策"        │
│  • todo 列表 + 执行状态               • proposal + design    │
│  • 执行完就消失                         + specs + tasks      │
│  • 不产出文件到项目中                  • 产出 .md 文件到项目中 │
│                                                             │
│  类比：便签纸上的待办清单              类比：项目的设计文档库   │
└─────────────────────────────────────────────────────────────┘
```

### 会冲突吗？

**不会冲突**，但有重叠。具体来说：

```
当前 Plan 的 todolist:
  1. 创建 OpenSpec proposal     ← 调用 OpenSpec
  2. 创建 tech-profiles.json
  3. 创建 ARCHITECTURE.md + SKILL.md
  4. 创建 6 个 agent prompt
  5. 创建协调者 rd-team.md
  6. 验证 + 审查

OpenSpec 创建后会产出:
  openspec/changes/ai-rd-team/
  ├── proposal.md              ← 提案（为什么做、做什么）
  ├── design.md                ← 设计（怎么做）
  ├── specs/                   ← 规格说明
  └── tasks.md                 ← 任务列表 ← ⚠️ 这里和 Plan 有重叠
```

**重叠点在 `tasks.md`**：
- Plan 的 todolist 有 6 个 todo
- OpenSpec 的 tasks.md 也会生成一份任务列表

### 三种执行策略

```
策略 A：Plan 主导，OpenSpec 仅记录设计决策
─────────────────────────────────────────
Plan todolist 驱动执行
OpenSpec 只产出 proposal.md + design.md（不产出 tasks.md）
✅ 简单，不重复
❌ OpenSpec 的 apply-change 工作流用不上

策略 B：OpenSpec 主导，Plan 退役
─────────────────────────────────────────
用 /opsx:propose 创建完整提案（含 tasks.md）
用 /opsx:apply 逐个执行 task
Plan 不再使用
✅ 纯 OpenSpec 工作流，设计和执行一体
❌ 当前 Plan 已经做好了，弃之可惜

策略 C：Plan 驱动，第一步调 OpenSpec 固化设计，后续 Plan 接管执行
─────────────────────────────────────────
Plan todo#1 → 调 openspec-propose 生成 proposal.md + design.md
Plan todo#2~6 → 直接执行（不依赖 OpenSpec tasks.md）
OpenSpec 作为"设计档案馆"，不驱动执行
✅ 设计有记录，执行不重复
✅ 当前 Plan 无需重做
```

---

## 我的建议：**策略 C**

原因：
1. **Plan 已经做好了**，里面的 6 个 todo 粒度合适，直接用
2. **OpenSpec 的价值在设计记录**——proposal.md 和 design.md 留在项目里，是有价值的设计文档
3. **不需要 OpenSpec 的 tasks.md**——因为 Plan 的 todolist 已经覆盖了，两份任务列表会让人迷惑
4. 后续如果别人想了解"为什么这么设计"，看 `openspec/changes/ai-rd-team/proposal.md` 就行

执行流程：

```
Plan todo#1: /opsx:propose → 生成 proposal.md + design.md
                             （记录设计决策，不生成 tasks.md，
                              或生成后不用它驱动执行）
Plan todo#2: 创建 tech-profiles.json
Plan todo#3: 创建 ARCHITECTURE.md + SKILL.md
Plan todo#4: 创建 6 个 agent prompt
Plan todo#5: 创建协调者 rd-team.md
Plan todo#6: 验证 + 审查
```

**你觉得这个策略可以吗？还是你更倾向纯 OpenSpec 驱动（策略 B）？**
