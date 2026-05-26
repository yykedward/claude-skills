---
name: design-debate-review
description: Review a plan or prompt before execution to identify hidden decision points, return the original text annotated with [Debate topic] markers, and optionally invoke agent-debate in parallel for each
---

# Design Debate Review

## Overview

Before you submit a plan or prompt for execution, throw it here first. This skill scans the content, finds where decisions are implicit or ambiguous, and returns:

1. **Annotated original text** — the plan/prompt with `[Debate <topic>]` markers at each decision point
2. **Decision point list** — what needs debate and why

From there, the user decides which topics to actually debate before executing.

**Core principle:** Surface hidden decisions in a plan before it goes live — the earlier you catch them, the less rework.

## When to Use

- You've drafted a plan or prompt and want to find decision gaps before submitting
- You want to stress-test a plan before committing resources to it
- User says "review this plan" / "审查计划" / "find debate points in this prompt"

**Note:** This skill does not assume a fixed position in any workflow. Use it on any plan or prompt whenever you want to audit for decision risks.

## Process (4 Steps)

### 1. REVIEW — Scan for Decision Points

Read the plan/prompt carefully. Screen for disputes against these dimensions:

| Dimension | Judgment Criteria |
|-----------|-------------------|
| **Impact on existing design** | Does it change data models, API contracts, or module boundaries? |
| **Constraints on future work** | Is the choice hard to revert? Will it limit future extensibility? |
| **Multiple viable options** | Are there 2+ approaches, each reasonable but with different tradeoffs? |
| **Hidden assumptions** | Does the plan rely on unverified assumptions? |

**Exclusion criteria (no debate needed):**
- Only one clearly correct approach exists
- Project already has established conventions, no room for dispute
- Decision scope is trivial (single function internals)

### 2. EXTRACT — Annotate and List

For each decision point, do two things:

**a) Annotate the original plan** — insert `[Debate <topic>]` at the exact spot where the decision arises:

```
Original:
The system caches user sessions in Redis.

Annotated:
The system caches user sessions in Redis [Debate session cache: Redis standalone vs Cluster vs in-memory].
```

**b) Define each topic** as a one-sentence, specific, answerable question:
- Not "which database to use", but "should user balances use MySQL row locks or Redis Lua scripts for concurrency safety?"
- Each topic independent of others where possible (enables parallel debate)

Annotate dependencies:

```
Topic A: <one sentence>  [independent]
Topic B: <one sentence>  [depends on A's result]
Topic C: <one sentence>  [independent]
```

### 3. OUTPUT — Return Results to User

Present two things:

**a) Annotated original plan** — the full plan/prompt with `[Debate <topic>]` markers inline.

**b) Decision point list:**

```
## Identified N Decision Points

| # | Location | Topic | Why Debate |
|---|----------|-------|-------------|
| 1 | Para X | <Topic A> | <one sentence> |
| 2 | Para Y | <Topic B> | <one sentence> |

**Parallel group:** Topics 1, 3 can run concurrently
**Sequential group:** Topic 2 depends on Topic 1
```

Then ask whether to proceed with debate, skip some, or skip all. The user may also adjust topic wording.

### 4. EXECUTE — Run Debates (Optional)

Only if the user confirms. They decide which topics and in what order.

- **Independent topics:** Invoke `yykedward-skills:agent-debate` in parallel, one per topic
- **Dependent topics:** Wait for prerequisite debate, pass its conclusion as context

Each debate output is saved under `docs/debate/`.

After all debates complete, summarize:

```
## Debate Results Summary

| # | Topic | Conclusion | Key Rationale |
|---|-------|------------|---------------|
| 1 | <Topic> | <Approach X> | <one sentence> |
| 2 | <Topic> | <Approach Y> | <one sentence> |
```

## Output

The primary deliverable is the **annotated original plan** + **decision point list**. Debate execution is optional and user-driven.

Whether to commit generated `docs/debate/` files is up to the user.
