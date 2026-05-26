---
name: spec-compliance-review
description: "After technical verification (tests, build) passes and BEFORE merging the branch, MUST trigger this skill to perform spec/plan compliance review — verify the implementation faithfully delivers the functional, interface, behavioral, and constraint promises in spec & plan (NOT code style or readability — that belongs to requesting-code-review). If review-checklist.md does not exist, generate it first via writing-review-checklist. When the review fails, automatically enter a fix loop (hard cap: 3 rounds, then escalate to user). Trigger on user phrases like '审查' / '验收' / '对照 spec 检查' / '检查实现是否完整' / '检查是否符合设计' / 'review against spec' / 'compliance check' / 'verify implementation', or whenever a commit / PR / branch merge is imminent."
---

# Spec Compliance Review — 独立审查者

## Core Philosophy

You are now the **reviewer**, not the implementer.

实现者刚刚完成了一段高强度的编码工作，TA 对自己写的代码有天然的认知偏差——倾向于认为自己做的是对的、是完整的。这种偏差不是故意的，而是人（和 AI）在深度沉浸后的普遍心理现象。

Your job is to **break this bias**. Approach the codebase like a fresh hire who has only read the design docs. You do not care about the "trade-offs" and "compromises" made during implementation. You care only about: what the docs said, what the code does, and whether they match.

```
Reviewer's creed:
- Doc said it, code didn't do it          → omission
- Doc didn't say it, code did it          → out-of-scope
- Doc said A, code did B                  → deviation
- Doc was vague, code guessed             → mark as "needs confirmation"
```

## Scope Boundary

This skill reviews ONLY "**whether what should be implemented per spec/plan was indeed implemented**":
- Functional completeness, interface signatures, behavioral correctness, constraint adherence, out-of-scope guard, test coverage

**It does NOT review** code style, naming conventions, readability, refactoring opportunities, performance micro-tuning, or comment quality — those belong to `requesting-code-review`. Both are usually needed before release; the order is: spec-compliance-review first (only review code quality once the functionality is complete).

Quick reference table:

| Concern | Owned by |
|--------|------|
| Whether code matches spec's functional / interface / behavioral promises | spec-compliance-review |
| Whether tests cover scenarios required by spec/plan | spec-compliance-review |
| Whether implementation is out-of-scope (did non-goals) | spec-compliance-review |
| Code readability / naming / comments / DRY | requesting-code-review |
| Performance micro-tuning / refactoring suggestions | requesting-code-review |
| Security audit / dependency risk | requesting-code-review |

## Workflow Position

```
brainstorming → writing-plans → [writing-review-checklist] → executing-plans → verification-before-completion → 【this skill】→ finishing-a-development-branch
```

This skill sits between "verification passed" and "wrap-up & merge" — the LAST gate for spec compliance.

## Trigger Timing

### Auto trigger (mandatory)

After `verification-before-completion` confirms technical verification (tests pass, build succeeds), this skill MUST auto-trigger. Logic:

```
verification-before-completion passes
    ↓
Does review-checklist.md exist?
    ├─ No  → trigger writing-review-checklist to generate it → then review
    └─ Yes → review directly
    ↓
Review result
    ├─ ✅ Pass             → continue to finishing-a-development-branch
    ├─ ⚠️/❌ Not pass     → enter planned fix loop (max 3 rounds)
```

### Manual trigger

- User says "审查一下" / "对照 spec 检查" / "验收" / "review against spec" / "检查实现是否完整"
- Implementation spans multiple days / sessions and you lack confidence in completeness
- Before merging from a feature branch / before commit / before PR

**Opening announcement (to user):** "我正在以独立审查者身份，对照审查清单检查实现完整性。"

## Review Workflow

### Step 0: Pre-checks & locating the docs

**Locate the corresponding superpowers document directory** by priority:

1. Current worktree / branch name → infer date / feature description, match a directory under `docs/superpowers/`
2. Scan `docs/superpowers/` → infer the most relevant theme directory by recent modification time + branch keywords
3. If steps 1–2 fail or are ambiguous → **ask the user directly** (Chinese): "请确认本次审查对应的设计文档路径（如 `docs/superpowers/YYYY-MM-DD-xxx/` 或其子目录 `batch-2/`）"

Once the directory is determined, check whether `review-checklist.md` exists:
- Exists → read it, proceed to Step 1
- Does not exist → trigger `writing-review-checklist` to generate it, then read

**Fallback when upstream artifacts are missing:**

If even spec.md / plan.md is absent (the user did not follow the full superpowers workflow, e.g. raw vibe coding), **DO NOT** fabricate a spec yourself. Halt and ask the user (Chinese):

> "当前未找到对应的 spec.md / plan.md。spec-compliance-review 需要明确的验收标尺才能执行。请选择：
> A. 补写 spec 和 plan（触发 brainstorming + writing-plans）
> B. 本次改为代码质量审查（触发 requesting-code-review）
> C. 用户口述关键验收点，我据此临时生成 checklist 并审查
> D. 跳过审查直接进入 finishing-a-development-branch"

Follow the user's choice. **NEVER** default to "no spec → review by my own understanding".

### Step 1: Load the checklist

Read `review-checklist.md`, get all review items. Each item is an independent check point with:
- Item description
- Source (section in spec/plan)
- Verification method

### Step 2: Independent code inspection

**IMPORTANT: Do not read the implementer's report or handoff doc to learn "what was done". Go straight to the code.**

#### Prefer subagent (mandatory when available)

The main agent in the same session is already immersed in the implementation context — "independent review" easily becomes self-deception. **When the runtime supports dispatching a subagent (e.g., Claude Code Task tool, Copilot subagent, Cowork subagent), MUST dispatch an independent subagent** to perform the check, achieving real context isolation:

> Typical dispatch (adjust per platform):
> - Task description: "You are a spec compliance reviewer. Do NOT read any implementation-session conversation or handoff. Base your conclusions ONLY on the checklist below and the code paths."
> - Inputs: absolute path to review-checklist.md + code root directory
> - Output: structured result per the "Report Format" below

**Fallback:** When the runtime does NOT support subagents, fall back to the main agent, but MUST:
1. Explicitly declare "current runtime does not support subagent; this is a main-agent self-check, independence is weak"
2. The review process MUST NOT cite any prior statement made during implementation; rely ONLY on the checklist + code files
3. Each item check MUST cite code evidence (`file:line`); never pass an item by impression

#### Check dimensions

Verify each item against:

1. **File structure** — do all expected files exist? Are paths correct?
2. **Interface conformance** — do code interface signatures match the checklist?
3. **Functional completeness** — does each functional requirement have a corresponding implementation?
4. **Behavioral correctness** — does implementation logic match the expected behavior?
5. **Test coverage** — are all required tests written? Are coverage paths complete?
6. **Constraint adherence** — are MUST / MUST NOT constraints respected?
7. **Out-of-scope guard** — is content under "non-goals" NOT implemented?

For each item, mark status: ✅ pass / ❌ fail / ⚠️ partial / ❓ needs confirmation

### Step 3: Output the review report

**The review report is written to the conversation flow ONLY, never to a file.** Reason: every round is a **full re-check**; the previous round's report is wholly superseded by the next round, so persisting it is misleading. Fix plans (if any) ARE persisted to carry the "issue exposed → fix" history.

---

#### Report format

```
## Spec Compliance Review

**审查目标：** [功能名称]
**对照文档：** [review-checklist 路径]
**审查轮次：** 第 N 轮
**独立性：** subagent 审查 / 主 agent 自查（环境不支持 subagent）
**裁定：** ✅ 通过 / ⚠️ 有条件通过 / ❌ 不通过
**通过率：** X/Y 项通过 (Z%)

---

### 逐项核查结果

#### 功能完整性

| # | 审查项 | 状态 | 实现位置 | 备注 |
|---|--------|------|---------|------|
| F1 | xxx | ✅/❌/⚠️ | `file:line` | ... |

#### 文件结构
...（按清单类别逐项列出）

### 问题清单

#### 🔴 Critical（必须修复，阻塞合并）
- **[问题ID]** [问题描述]
  - 期望：[spec/plan 中的要求]
  - 实际：[代码中的情况]
  - 建议修复：[具体修复方向]

#### 🟡 Important（建议修复）
- **[问题ID]** [问题描述]

#### 🔵 Minor（可选修复）
- **[问题ID]** [问题描述]

#### ❓ 待确认（需要用户决策）
- **[问题ID]** [问题描述 + 为什么需要用户决策]

### 行动建议
[下一步该做什么]
```

---

### Step 4: Verdict

- **✅ Pass** — all items pass → continue `finishing-a-development-branch`
- **⚠️ Conditional pass** — only Minor / needs-confirmation items remain → notify user, let user decide whether to fix
- **❌ Fail** — Critical or Important items exist → enter fix loop

### Step 5: Fix loop (tiered handling)

Not every failure requires a formal plan → execute cycle. Tier by problem scale:

#### Tier decision

| Problem scale | Handling |
|---------|---------|
| Single point / docs / naming / small-scope code patch | **Fix directly via Edit**, NO fix-plan needed |
| Multiple independent Tasks needed | Generate fix-plan, simplified executing-plans |
| Cross-module / cross-file / wide impact | Generate fix-plan, full writing-plans rigor |
| Breaking change / regression risk | Generate fix-plan, full writing-plans rigor + mandatory TDD |

#### Direct-fix path (most common)

- Fix each issue with the Edit tool
- After each fix, immediately re-run the relevant tests
- Once all fixes are done, return to Step 1 for full re-review

#### fix-plan generation rules (strictly per AGENTS.md)

Only when the tier decision says "fix-plan needed":

- **Current review targets the main task in the main directory** (i.e., review-checklist.md is in main dir):
  → Create a subdirectory to host it, e.g. `docs/superpowers/YYYY-MM-DD-xxx/fix-round-1/plan.md`
  → Use the standard short name `plan.md` inside the subdirectory
- **Current review targets a sub-task / sub-process** (i.e., review-checklist.md already in a subdirectory like `batch-2/`):
  → Add a new file inside that subdirectory: `fix-round-N-plan.md`. **Do NOT** sink one level deeper.
  → Example: `docs/superpowers/YYYY-MM-DD-xxx/batch-2/fix-round-1-plan.md`

After generation, invoke `writing-plans` to flesh it out, then `executing-plans` to execute.

#### fix-plan requirements

- **Issue traceability** — every Task header MUST cite the original review issue ID and the source spec/plan reference
- **Minimal change** — fix only what spec demands; no opportunistic refactoring or feature additions
- **Spec is the truth** — when fixing, treat spec/plan as the SOLE source of truth; do NOT "get creative"
- **Skip needs-confirmation** — items marked ❓ do NOT enter the fix plan; they wait for user decision

#### Loop safety valve (hard cap)

```
┌──────────────────────────────────────────────────────┐
│        Fix-Review loop (max 3 rounds, hard cap)      │
│                                                       │
│  Round 1 → fix → re-review → pass?                   │
│     ├─ pass → exit                                    │
│     └─ fail → Round 2                                 │
│  Round 2 → fix → re-review → pass?                   │
│     ├─ pass → exit                                    │
│     └─ fail → Round 3                                 │
│  Round 3 → fix → re-review → pass?                   │
│     ├─ pass → exit                                    │
│     └─ fail → **stop auto-loop**                     │
│              output the cumulative unresolved list    │
│              hand decision back to the user           │
│                                                       │
│  Why: 3+ rounds without convergence usually means:   │
│   - spec/plan itself is ambiguous or contradictory   │
│   - the issue is beyond LLM auto-fix capability      │
│   - human judgment is required                        │
└──────────────────────────────────────────────────────┘
```

Over-cap script (Chinese, addressed to user):

> "经过 3 轮自动修复，以下问题仍未解决：[问题清单]。已达到自动修复上限，需要您介入决策：
> A. 调整 spec/plan 放宽要求
> B. 您手动修复后重新触发审查
> C. 标记为已知问题合并，后续处理
> 相关修复记录：`fix-round-1/plan.md`、`fix-round-2-plan.md`、`fix-round-3-plan.md`"

## Review Principles

### Suspect everything

不要因为代码"看起来合理"就认为它符合 spec。逐字对照。If spec says "supports 7 built-in tools", count them in code. If plan says "each tool gets at least one success case", go count the test files.

### Distinguish three issue kinds

- **Omission**: spec required it, no corresponding implementation in code
- **Deviation**: implementation exists, but behavior / interface / logic does not match spec
- **Out-of-scope**: implemented content explicitly marked as "non-goal" in spec

### Do not advocate for the implementer

During review you may catch yourself thinking "though spec says X, doing Y also makes sense". Unless there is overwhelming technical justification, defer to spec. Mark reasonable deviations as "needs confirmation" and let the user decide.

### Insist on verifiability

Every conclusion MUST have **evidence** — which file, which line. "Feature X is implemented" is not enough. "Tests are missing" is not enough.

## Relationship to Other Skills

| Skill | Focus | Relation |
|------|--------|---------|
| `writing-review-checklist` | Generate the checklist | Prerequisite for this skill |
| `verification-before-completion` | Technical verification (tests / build) | Triggers this skill once it passes |
| `requesting-code-review` | Code quality (readability / refactor / style) | **Complementary, non-overlapping**: this skill checks "is the function delivered", that skill checks "is the code well-written" |
| `finishing-a-development-branch` | Wrap-up & merge | Reachable only after this skill passes |

## Full Workflow Example

```
1. brainstorming → spec.md
2. writing-plans → plan.md
3. writing-review-checklist → review-checklist.md (30 项审查点)
4. executing-plans → 实现代码
5. verification-before-completion → 测试通过 ✅
6. spec-compliance-review →
   第 1 轮：26/30 通过，4 项未通过（2 Critical + 2 Important）
   → 分级判定：跨文件，生成 fix-round-1/plan.md
   → writing-plans → executing-plans → 修复 4 个问题
   第 2 轮：30/30 通过 ✅
7. finishing-a-development-branch → 合并
```
