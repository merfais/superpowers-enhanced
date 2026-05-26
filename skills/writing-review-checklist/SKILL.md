---
name: writing-review-checklist
description: "MUST trigger immediately after writing-plans saves plan.md to generate review-checklist.md — this is the mandatory gate between '写完 plan' and '开始写代码'. The checklist is the SOLE acceptance ruler for the downstream spec-compliance-review skill: it verifies whether the implementation faithfully delivers the functional/interface/behavioral/constraint promises in spec & plan (NOT code style or readability — that belongs to requesting-code-review). Trigger on user phrases like '写审查清单' / '生成 review checklist' / '准备验收标准' / '可以开始实现了' / '可以写代码了' / '开干吧' / 'write the checklist' / 'prepare acceptance criteria', or whenever a fresh plan.md is detected. If the user attempts to skip the checklist and jump straight into implementation, this skill MUST intercept first."
---

# Writing Review Checklist — 生成验收审查清单

## Purpose

Right after the implementation plan (plan.md) is finalized, transform every **functional promise** in spec & plan into an itemized, individually verifiable acceptance checklist. This checklist becomes the SOLE source of truth for the downstream `spec-compliance-review` skill.

**Scope boundary:**
- This checklist verifies ONLY "whether functions / interfaces / behaviors / constraints are implemented as promised in spec & plan"
- It does NOT verify code style, readability, naming aesthetics, or comment conventions — those belong to `requesting-code-review`

Why generate the checklist BEFORE implementation?
- During implementation, developers gradually "habituate" to the current state and lose sensitivity to omissions
- Writing the checklist up front gives the future reviewer an unbiased ruler, immune to implementation drift
- If the checklist cannot even be written, the spec/plan itself is not concrete enough

## Trigger Timing

**Auto trigger (mandatory):**
- Immediately after `writing-plans` completes and saves plan.md
- Workflow: `brainstorming → writing-plans → 【this skill】→ implementation`

**Manual trigger:**
- User says "写审查清单" / "生成 review checklist" / "准备验收标准"
- When `spec-compliance-review` runs without a corresponding checklist, it will invoke this skill first

**Interception trigger (CRITICAL):**

When plan.md is ready but checklist is not yet generated, the following user expressions MUST be intercepted to run this skill first, BEFORE entering implementation:
- "可以开始实现了" / "可以写代码了" / "开干吧" / "执行 plan"
- "用 executing-plans" / "用 subagent-driven-development"
- Any utterance semantically implying "skip the checklist, jump to implementation"

**Opening announcement scripts (apply in order):**

1. **If triggered via interception** (user intended to skip the checklist), first say the interception line:
   > "在进入实现前，按 superpowers 工作流我需要先生成 review-checklist.md，作为后续验收的标尺。这一步预计 1 分钟，跳过会导致 spec-compliance-review 阶段无据可依。"

2. **Then (always) announce the start:**
   > "我正在根据 spec 和 plan 生成验收审查清单。"

## Inputs

Read from the corresponding superpowers document directory:
- `spec.md` — design specification
- `plan.md` — implementation plan
- Other supplementary documents (if any)

## Output Location & Naming

**Strictly follow the directory & naming rules in workspace `AGENTS.md`:**

- Filename MUST be `review-checklist.md`. NEVER encode batch/phase info in the filename
- Batch/phase MUST be expressed via subdirectories:
  - Main task (no sub-task split) → `docs/superpowers/YYYY-MM-DD-功能描述/review-checklist.md`
  - Sub-task / sub-process → `docs/superpowers/YYYY-MM-DD-功能描述/<batch-2|phase-2|task-a>/review-checklist.md`
- How to decide whether it's a sub-task:
  - If the current plan lives in a subdirectory → write checklist in that subdirectory
  - If plan lives in main directory but describes an independent phase → upgrade to subdirectory
  - Otherwise → main directory

**Built-in anti-bloat mechanism:** When a large requirement is split into multiple sub-tasks, each sub-task owns its plan + its checklist; each checklist covers only its own phase. A single checklist should not exceed the verifiable promises of its phase. If one checklist tries to cover the entire mega-requirement, the split granularity is wrong — go back to writing-plans and re-split.

## Generation Workflow

### Step 1: Extract functional requirements from spec.md

For each section of spec.md:
- Each functional point or behavioral requirement → one acceptance item
- Each architectural constraint → one constraint check item
- Each "non-goal" → one out-of-scope check item
- Data flow definitions → one pipeline verification item
- Test strategy → one test coverage check item

### Step 2: Extract implementation commitments from plan.md

For each section of plan.md:
- Each Task's deliverables → one delivery check item
- File structure definitions → one file-existence check item
- Interface signatures → one interface-conformance check item
- Test cases → one test-existence check item

### Step 3: Deduplicate & merge

spec and plan may overlap (plan refines spec):
- Merge semantically identical items
- Keep the most concrete version (usually from plan)
- If plan refines spec, keep both (spec-level completeness + plan-level precision)

### Step 4: Categorize & annotate

Categorize all items into the following sections (use the **Chinese names** as the actual section headers in the output file; English glosses are for LLM comprehension only):

- **功能完整性** (Functional completeness) — every requirement in spec is implemented
- **文件结构** (File structure) — expected files exist at correct paths
- **接口合规** (Interface conformance) — code interfaces match documented signatures
- **行为正确性** (Behavioral correctness) — logic matches description
- **测试覆盖** (Test coverage) — required tests are in place
- **约束遵守** (Constraint adherence) — explicit MUST / MUST NOT respected
- **越界防护** (Out-of-scope guard) — content under "non-goals" is NOT implemented

For each item, annotate two fields (Chinese labels in the output, English glosses for comprehension):
- **来源** (Source): spec § section / plan Task N Step M
- **验证方式** (Verification method): file existence / code search / signature comparison / behavioral test / human confirmation

### Step 5: Write to file

## Item Granularity Guidelines

**Good item characteristics** (verifiable + atomic):
- ✅ "ToolRegistry 暴露 `register(name: string, executor: Executor): void` 接口"
- ✅ "`run_shell` 工具的返回值结构包含 `stdout`、`stderr`、`exitCode` 三个字段"
- ✅ "审批拒绝后任务状态变更为 `cancelled`，且不触发 `taskCompleted` 事件"

**Bad item characteristics** (drop or split):
- ❌ "系统应该正常运行" → not verifiable
- ❌ "代码应当可读、易维护" → belongs to code review
- ❌ "实现优雅、性能良好" → subjective, not a spec promise
- ❌ "完成所有功能并通过测试" → too coarse, must be split
- ❌ "Planner、Orchestrator、ToolRegistry 三者协同工作" → compound, must be split

**Items to drop:**
- Implementation details (variable names, private method names, formatting)
- "Reasonable guesses" not explicitly required by spec/plan
- Generic engineering virtues (DRY, SOLID, sufficient comments)

**Scale reference:** A single checklist usually contains 15–50 items. Fewer than 10 → likely missing items; more than 60 → likely too fine-grained or includes code-review concerns.

## Output Template

```markdown
# Review Checklist — [功能名称]

> 本文档由 writing-review-checklist 技能自动生成，作为 spec-compliance-review 的审查依据。
> 基于: spec.md (YYYY-MM-DD) / plan.md (YYYY-MM-DD)
> 覆盖范围: [主任务全量 / 子任务 batch-2 / phase-2 ...]

---

## 功能完整性

| # | 审查项 | 来源 | 验证方式 |
|---|--------|------|---------|
| F1 | [需求描述] | spec § [节名] | [如何验证] |
| F2 | ... | ... | ... |

## 文件结构

| # | 预期文件/目录 | 来源 |
|---|-------------|------|
| D1 | `path/to/file.ts` | plan Task N |

## 接口合规

| # | 接口/类型定义 | 预期签名 | 来源 |
|---|-------------|---------|------|
| I1 | `FunctionName` | `(param: Type) => ReturnType` | plan Task N |

## 行为正确性

| # | 行为描述 | 预期表现 | 来源 | 验证方式 |
|---|---------|---------|------|---------|
| B1 | [场景] | [预期结果] | spec § [节名] | [如何验证] |

## 测试覆盖

| # | 预期测试 | 覆盖场景 | 来源 |
|---|---------|---------|------|
| T1 | [测试描述] | [覆盖什么] | plan Task N / spec 测试策略 |

## 约束遵守

| # | 约束描述 | 类型 | 来源 |
|---|---------|------|------|
| C1 | [必须/禁止 做什么] | MUST / MUST NOT | spec § [节名] |

## 越界防护

| # | 非目标项 | 来源 |
|---|---------|------|
| X1 | [不应实现的内容] | spec 非目标 |

---

**统计：** 共 N 项审查点（功能 x 项 / 文件 x 项 / 接口 x 项 / 行为 x 项 / 测试 x 项 / 约束 x 项 / 越界 x 项）
```

> Note: The checklist carries NO status column. This checklist is the IMMUTABLE ruler — every spec-compliance-review pass performs a **full re-check** against it (incremental review would miss regressions on already-accepted items). Review results are written to the conversation flow, never back into this file.

## State Management Principles

- **Immutable**: once generated, do NOT tick checkboxes or update status fields on items
- **Full re-check every round**: spec-compliance-review re-verifies every item every round, never "only the changed ones"
- **Why**: incremental review cannot detect "new bugs introduced into already-accepted features"; only full re-runs provide regression safety

## Worked Example

Suppose spec says:
> "新增 `ToolRegistry`，提供 register / get / list 三个方法。register 注册同名工具时应抛出错误。"

And plan says:
> "Task 1: 创建 `packages/tools/src/registry.ts`，导出 `class ToolRegistry`，方法签名见接口。Task 2: 添加单元测试，至少覆盖：注册成功、重复注册抛错、get 不存在返回 undefined。"

Then the relevant checklist items:

```markdown
## 文件结构
| D1 | `packages/tools/src/registry.ts` | plan Task 1 |

## 接口合规
| I1 | `ToolRegistry.register` | `(name: string, executor: Executor) => void` | plan Task 1 |
| I2 | `ToolRegistry.get`      | `(name: string) => Executor \| undefined`    | plan Task 1 |
| I3 | `ToolRegistry.list`     | `() => string[]`                              | plan Task 1 |

## 行为正确性
| B1 | 重复注册同名工具 | 抛出错误 | spec § ToolRegistry | 单元测试断言 throws |
| B2 | get 不存在的工具名 | 返回 undefined | spec § ToolRegistry | 单元测试断言 |

## 测试覆盖
| T1 | 注册成功 case      | 正常注册一个工具后 list 包含其名 | plan Task 2 |
| T2 | 重复注册抛错 case  | 同名 register 第二次抛错        | plan Task 2 |
| T3 | get 不存在 case    | 返回 undefined                  | plan Task 2 |
```

Notice the pattern: each item checks exactly one thing; each can be confirmed within seconds via "open file / run test / inspect signature"; no subjective items.

## Quality Bar

- **No omissions** — every verifiable promise in spec/plan must appear in the checklist
- **Actionable** — every item has a clear verification method; the reviewer should know exactly how to check
- **Traceable** — every item annotates its source for fast back-tracing
- **Atomic** — each item checks exactly one thing; never merge multiple checks

## After Generation

After producing the checklist:
1. Tell the user the checklist is generated and give the path
2. Report the total item count (broken down by category)
3. Tell the user (Chinese): "审查清单已就绪，实现完成后使用 spec-compliance-review 技能执行验收审查。"
4. Continue the normal implementation flow (execute the plan)
