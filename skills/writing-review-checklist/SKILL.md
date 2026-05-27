---
name: writing-review-checklist
description: "MUST trigger immediately after writing-plans saves plan.md to generate review-checklist.md — this is the mandatory gate between 'plan is written' and 'start coding'. The checklist is the SOLE acceptance ruler for the downstream spec-compliance-review skill: it verifies whether the implementation faithfully delivers the functional/interface/behavioral/constraint promises in spec & plan (NOT code style or readability — that belongs to requesting-code-review). Trigger on user phrases like 'write the review checklist' / 'generate review checklist' / 'prepare acceptance criteria' / 'we can start implementing now' / 'we can start coding now' / 'let's begin implementation' / 'write the checklist' / 'prepare acceptance criteria', or whenever a fresh plan.md is detected. If the user attempts to skip the checklist and jump straight into implementation, this skill MUST intercept first."
---

# Writing Review Checklist — Generate the Acceptance Checklist

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
- User says "write the review checklist" / "generate review checklist" / "prepare acceptance criteria"
- When `spec-compliance-review` runs without a corresponding checklist, it will invoke this skill first

**Interception trigger (CRITICAL):**

When plan.md is ready but checklist is not yet generated, the following user expressions MUST be intercepted to run this skill first, BEFORE entering implementation:
- "we can start implementing now" / "we can start coding now" / "let's begin implementation" / "execute the plan"
- "use executing-plans" / "use subagent-driven-development"
- Any utterance semantically implying "skip the checklist, jump to implementation"

**Opening announcement scripts (apply in order):**

1. **If triggered via interception** (user intended to skip the checklist), first say the interception line:
   > "Before implementation begins, the superpowers workflow requires me to generate `review-checklist.md` first as the acceptance ruler for the later review. This takes about one minute, and skipping it would leave `spec-compliance-review` without a reliable basis."

2. **Then (always) announce the start:**
   > "I am generating the acceptance checklist from the spec and plan now."

## Inputs

Read from the corresponding feature directory under `docs/superpowers/YYYY-MM-DD-<feature-name>/`:
- `spec.md` — design specification
- `plan.md` — implementation plan
- Other supplementary documents (if any)

## Output Location & Naming

**Strictly follow the directory & naming rules in workspace `AGENTS.md`:**

- Main task (no sub-task split) → `docs/superpowers/YYYY-MM-DD-<feature-name>/review-checklist.md`
- Sub-task / sub-process → keep files flat in the same feature directory using a scoped filename, e.g. `docs/superpowers/YYYY-MM-DD-<feature-name>/batch-2-review-checklist.md`
- Do NOT create second-level directories for `batch`, `phase`, `task`, or `fix-round`

**Built-in anti-bloat mechanism:** When a large requirement is split into multiple sub-tasks, each sub-task owns its own scoped plan + checklist pair in the same feature directory. A single checklist should not exceed the verifiable promises of its phase. If one checklist tries to cover the entire mega-requirement, the split granularity is wrong — go back to writing-plans and re-split.

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

Categorize all items into the following sections (use the English names below as the actual section headers in the output file):

- **Functional Completeness** — every requirement in the spec is implemented
- **File Structure** — expected files exist at the correct paths
- **Interface Conformance** — code interfaces match documented signatures
- **Behavioral Correctness** — logic matches the description
- **Test Coverage** — required tests are in place
- **Constraint Adherence** — explicit MUST / MUST NOT requirements are respected
- **Out-of-Scope Guard** — content under "non-goals" is NOT implemented

For each item, annotate two fields:
- **Source**: spec section / plan Task N Step M
- **Verification Method**: file existence / code search / signature comparison / behavioral test / human confirmation

### Step 5: Write to file

## Item Granularity Guidelines

**Good item characteristics** (verifiable + atomic):
- ✅ "ToolRegistry exposes a `register(name: string, executor: Executor): void` interface"
- ✅ "The return shape of `run_shell` includes the `stdout`, `stderr`, and `exitCode` fields"
- ✅ "After approval is rejected, the task status changes to `cancelled` and does not emit the `taskCompleted` event"

**Bad item characteristics** (drop or split):
- ❌ "The system should run correctly" → not verifiable
- ❌ "The code should be readable and maintainable" → belongs to code review
- ❌ "The implementation should be elegant and performant" → subjective, not a spec promise
- ❌ "Complete all functionality and pass all tests" → too coarse, must be split
- ❌ "Planner, Orchestrator, and ToolRegistry work together" → compound, must be split

**Items to drop:**
- Implementation details (variable names, private method names, formatting)
- "Reasonable guesses" not explicitly required by spec/plan
- Generic engineering virtues (DRY, SOLID, sufficient comments)

**Scale reference:** A single checklist usually contains 15–50 items. Fewer than 10 → likely missing items; more than 60 → likely too fine-grained or includes code-review concerns.

## Output Template

```markdown
# Review Checklist — [Feature Name]

> This document is generated automatically by the `writing-review-checklist` skill and serves as the review basis for `spec-compliance-review`.
> Based on: `spec.md` (YYYY-MM-DD) / `plan.md` (YYYY-MM-DD)
> Coverage: [full main task / sub-task batch-2 / phase-2 ...]

---

## Functional Completeness

| # | Checklist Item | Source | Verification Method |
|---|--------|------|---------|
| F1 | [Requirement description] | spec section [name] | [How to verify] |
| F2 | ... | ... | ... |

## File Structure

| # | Expected File/Directory | Source |
|---|-------------|------|
| D1 | `path/to/file.ts` | plan Task N |

## Interface Conformance

| # | Interface/Type Definition | Expected Signature | Source |
|---|-------------|---------|------|
| I1 | `FunctionName` | `(param: Type) => ReturnType` | plan Task N |

## Behavioral Correctness

| # | Behavior Description | Expected Outcome | Source | Verification Method |
|---|---------|---------|------|---------|
| B1 | [Scenario] | [Expected result] | spec section [name] | [How to verify] |

## Test Coverage

| # | Expected Test | Covered Scenario | Source |
|---|---------|---------|------|
| T1 | [Test description] | [What it covers] | plan Task N / spec test strategy |

## Constraint Adherence

| # | Constraint Description | Type | Source |
|---|---------|------|------|
| C1 | [What must / must not happen] | MUST / MUST NOT | spec section [name] |

## Out-of-Scope Guard

| # | Non-goal Item | Source |
|---|---------|------|
| X1 | [What must not be implemented] | spec non-goals |

---

**Totals:** N checklist items in total (functional x / file x / interface x / behavior x / test x / constraint x / out-of-scope x)
```

> Note: The checklist carries NO status column. This checklist is the IMMUTABLE ruler — every spec-compliance-review pass performs a **full re-check** against it (incremental review would miss regressions on already-accepted items). Review results are written to the conversation flow, never back into this file.

## State Management Principles

- **Immutable**: once generated, do NOT tick checkboxes or update status fields on items
- **Full re-check every round**: spec-compliance-review re-verifies every item every round, never "only the changed ones"
- **Why**: incremental review cannot detect "new bugs introduced into already-accepted features"; only full re-runs provide regression safety

## Worked Example

Suppose spec says:
> "Add `ToolRegistry` with the methods `register`, `get`, and `list`. Registering a tool under an existing name must throw an error."

And plan says:
> "Task 1: Create `packages/tools/src/registry.ts` and export `class ToolRegistry`; method signatures are defined by the interface. Task 2: Add unit tests that cover at least successful registration, duplicate registration throwing an error, and `get` returning `undefined` for a missing tool."

Then the relevant checklist items:

```markdown
## File Structure
| D1 | `packages/tools/src/registry.ts` | plan Task 1 |

## Interface Conformance
| I1 | `ToolRegistry.register` | `(name: string, executor: Executor) => void` | plan Task 1 |
| I2 | `ToolRegistry.get`      | `(name: string) => Executor \| undefined`    | plan Task 1 |
| I3 | `ToolRegistry.list`     | `() => string[]`                              | plan Task 1 |

## Behavioral Correctness
| B1 | Registering a duplicate tool name | Throws an error | spec section ToolRegistry | Unit test asserts `throws` |
| B2 | Calling `get` with a missing tool name | Returns `undefined` | spec section ToolRegistry | Unit test assertion |

## Test Coverage
| T1 | Successful registration case | After registering one tool, `list` includes its name | plan Task 2 |
| T2 | Duplicate registration throws | Calling `register` a second time with the same name throws | plan Task 2 |
| T3 | Missing `get` case | Returns `undefined` | plan Task 2 |
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
3. Tell the user: "The review checklist is ready. After implementation is complete, use `spec-compliance-review` to run the acceptance review."
4. Continue the normal implementation flow (execute the plan)

## Required Next Step

Once `review-checklist.md` is generated, implementation may begin.

Before implementation starts, ensure an isolated execution workspace exists if needed by invoking `superpowers:using-git-worktrees`.

If the runtime supports subagents, the recommended next step is:
- `superpowers:using-git-worktrees` (if isolation is not already established)
- then `superpowers:subagent-driven-development`

Otherwise:
- `superpowers:using-git-worktrees` (if isolation is not already established)
- then `superpowers:executing-plans`

**Implementation choice message:**

"The review checklist has been generated and is ready. Next, confirm the isolated workspace setup before execution begins. Available execution options:
1. `subagent-driven-development` (recommended)
2. `executing-plans`
Please choose one."
