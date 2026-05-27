---
name: spec-compliance-review
description: "After technical verification (tests, build) passes and BEFORE merging the branch, MUST trigger this skill to perform spec/plan compliance review — verify the implementation faithfully delivers the functional, interface, behavioral, and constraint promises in spec & plan (NOT code style or readability — that belongs to requesting-code-review). If review-checklist.md does not exist, generate it first via writing-review-checklist. When the review fails, automatically enter a fix loop (hard cap: 3 rounds, then escalate to user). Trigger on user phrases like 'review this against the spec' / 'run acceptance review' / 'check implementation completeness against the spec' / 'verify implementation completeness' / 'check whether this matches the design' / 'review against spec' / 'compliance check' / 'verify implementation', or whenever a commit / PR / branch merge is imminent."
---

# Spec Compliance Review — Independent Reviewer

## Core Philosophy

You are now the **reviewer**, not the implementer.

The implementer has just finished an intense coding session and naturally carries cognitive bias toward the code they wrote, tending to believe it is correct and complete. This bias is not malicious; it is a common human and AI pattern after deep immersion.

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

- User says "review this" / "check this against the spec" / "run acceptance review" / "review against spec" / "check whether the implementation is complete"
- Implementation spans multiple days / sessions and you lack confidence in completeness
- Before merging from a feature branch / before commit / before PR

**Opening announcement (to user):** "I am reviewing the implementation as an independent reviewer against the review checklist."

## Review Workflow

### Step 0: Pre-checks & locating the docs

**Locate the corresponding superpowers document directory** by priority:

1. Current worktree / branch name → infer date / feature description, match a directory under `docs/superpowers/`
2. Scan `docs/superpowers/` → infer the most relevant theme directory by recent modification time + branch keywords
3. If steps 1–2 fail or are ambiguous → **ask the user directly**: "Please confirm the design document path for this review (for example `docs/superpowers/YYYY-MM-DD-<feature-name>/`; if this is a scoped file, it may look like `batch-2-plan.md`)."

Once the directory is determined, check whether `review-checklist.md` exists:
- Exists → read it, proceed to Step 1
- Does not exist → trigger `writing-review-checklist` to generate it, then read

**Fallback when upstream artifacts are missing:**

If even `spec.md` / `plan.md` is absent (the user did not follow the full superpowers workflow, e.g. raw vibe coding), **DO NOT** fabricate a spec yourself. Halt and ask the user:

> "I could not find the corresponding `spec.md` / `plan.md`. `spec-compliance-review` requires a clear acceptance ruler before it can run. Please choose:
> A. Write the missing spec and plan first (trigger `brainstorming` + `writing-plans`)
> B. Switch this into a code quality review instead (trigger `requesting-code-review`)
> C. Tell me the key acceptance points verbally and I will generate a temporary checklist from them for this review
> D. Skip this review and go directly to `finishing-a-development-branch`"

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

**Review Target:** [Feature Name]
**Reference Document:** [review-checklist path]
**Review Round:** Round N
**Independence Mode:** subagent review / main-agent self-review (runtime does not support subagents)
**Verdict:** ✅ Pass / ⚠️ Conditional pass / ❌ Fail
**Pass Rate:** X/Y items passed (Z%)

---

### Item-by-Item Results

#### Functional Completeness

| # | Review Item | Status | Implementation Location | Notes |
|---|--------|------|---------|------|
| F1 | xxx | ✅/❌/⚠️ | `file:line` | ... |

#### File Structure
... (list each checklist category item by item)

### Issues

#### 🔴 Critical (must fix, blocks merge)
- **[Issue ID]** [Issue description]
  - Expected: [Requirement from spec/plan]
  - Actual: [What the code does]
  - Recommended fix: [Concrete repair direction]

#### 🟡 Important (recommended fix)
- **[Issue ID]** [Issue description]

#### 🔵 Minor (optional fix)
- **[Issue ID]** [Issue description]

#### ❓ Needs Confirmation (requires user decision)
- **[Issue ID]** [Issue description + why user input is required]

### Recommended Next Action
[What should happen next]
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

- **Current review targets the main task in the main directory** (i.e., `review-checklist.md` is the main checklist):
  → Create a flat scoped file in the same feature directory, e.g. `docs/superpowers/YYYY-MM-DD-<feature-name>/fix-round-1-plan.md`
- **Current review targets a sub-task / sub-process** (i.e., checklist file is already scoped like `batch-2-review-checklist.md`):
  → Create a flat scoped file in the same feature directory, e.g. `docs/superpowers/YYYY-MM-DD-<feature-name>/batch-2-fix-round-1-plan.md`

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

Over-cap script (addressed to the user):

> "After 3 automated fix rounds, the following issues are still unresolved: [issue list]. The automatic fix limit has been reached and I need your decision:
> A. Adjust the spec/plan to relax the requirement
> B. You fix it manually and then trigger the review again
> C. Merge while marking these as known issues to handle later
> Related fix records: `fix-round-1-plan.md`, `fix-round-2-plan.md`, `fix-round-3-plan.md`"

## Review Principles

### Suspect everything

Do not assume the code matches the spec just because it "looks reasonable". Compare it line by line. If the spec says "supports 7 built-in tools", count them in the code. If the plan says "each tool gets at least one success case", count the test files.

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
3. writing-review-checklist → review-checklist.md (30 checklist items)
4. executing-plans → implementation code
5. verification-before-completion → tests pass ✅
6. spec-compliance-review →
   Round 1: 26/30 passed, 4 items failed (2 Critical + 2 Important)
   → Tier decision: cross-file issue, generate `fix-round-1-plan.md`
   → writing-plans → executing-plans → fix the 4 issues
   Round 2: 30/30 passed ✅
7. finishing-a-development-branch → merge
```
