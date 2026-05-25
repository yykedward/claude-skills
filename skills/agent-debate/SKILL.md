---
name: agent-debate
description: Use when encountering design decisions with multiple valid approaches, architectural tradeoffs, or when two implementation strategies need objective comparison during development
---

# Agent Debate

## Overview

When a design decision has multiple valid approaches, dispatch N agents to independently analyze from different angles, run structured adversarial rounds (critique + rebuttal), then use a judge to select the best approach.

**Core principle:** Multiple independent analyses + structured debate > single agent judgment.

**Adaptive depth:** After adversarial rounds, measure structural divergence between proposals. High divergence → fast judge (proposals are clearly distinct, critique already stress-tested them). Low divergence → deep judge with 3-judge ensemble (proposals may share blind spots). This gives proportional scrutiny — cheaper for simple decisions, deeper for complex ones.

**Cost range:** 3N+2 (fast/standard) to 3N+4 (deep). Deep path only triggers on low divergence.

## When to Use

- Architecture decision with multiple valid options
- API design, algorithm/performance tradeoffs, implementation strategy comparisons
- User says "debate this" / "辩论", or plan/agent signals `[DEBATE: topic]` / `NEEDS_DEBATE: topic`

**Skip when** only one correct approach exists, or the decision is trivial.

## Trigger Modes

| Mode | Signal | Action |
|------|--------|--------|
| Checkpoint | Plan marks `[DEBATE: topic]` | Launch debate at that step |
| Agent-initiated | Agent signals `NEEDS_DEBATE: topic` | Pause, launch debate |
| User-initiated | "debate this" / "辩论" | Launch on current decision |

## The Debate Flow (9 phases)

```
1. IDENTIFY — write a Debate Brief scoping one specific question
2. SIZE     — calculate agent count: breadth + depth - 1 (clamped 2–5)
3. CONFIRM  — present plan (agents, angles, cost range) to caller; WAIT for approval
4. PROPOSE  — N agents in parallel, each from a different angle
5. CRITIQUE — N agents in parallel, each critiques all other proposals
6. REBUTTAL — N agents in parallel, each defends against critiques of their proposal
7. DIVERGE  — 1 Haiku call: measure structural similarity on 3 axes → route depth
8. JUDGE    — fast/standard/deep path based on divergence
9. OUTPUT   — save to docs/debate/yyyy-MM-dd-{topic}.md, return to caller
```

## Debate Brief

Write BEFORE launching any agents. Included in every agent's prompt as project context.

```markdown
**Question:** <one specific, answerable question>

**Context:** <tech stack, constraints, existing patterns>

**Evaluation Criteria (ranked):**
1. <top priority>
2. <second priority>
3. <third priority>

**Scope boundaries:** <what is OUT of scope>

**Angle assignments:**
- Agent #1 argues from: <angle>
- Agent #2 argues from: <angle>
```

## Agent Count (Breadth × Depth)

**Breadth** — how many distinct viable options?
| 1 | Binary (A vs B) | 2 | Multiple options (A vs B vs C) | 3 | Open design space |

**Depth** — how far-reaching is the impact?
| 1 | Local (single module) | 2 | Cross-cutting (multiple services) | 3 | Systemic (architecture) |

**Formula:** `agents = breadth + depth - 1` → clamp to [2, 5]

| B | D | N | When |
|---|---|---|------|
| 1 | 1 | 2 | Simple binary, local |
| 1 | 2 | 2 | Binary, cross-cutting |
| 2 | 1 | 2 | Multi-option, local |
| 2 | 2 | 3 | Multi-option, cross-cutting |
| 2 | 3 | 4 | Multi-option, systemic |
| 3 | 2 | 4 | Open design, cross-cutting |
| 3 | 3 | 5 | Open design, systemic |

**Angle pool for 3+ agents:** Simplicity/DX, Performance/Scale, Reliability/Correctness, Flexibility/Evolvability, Operations/Cost. Pick N distinct angles.

## Confirmation Gate

Before launching, present and **wait for explicit approval:**

```
## Debate Plan
**Question:** <from brief>
**Agent count:** N
**Angles:** <one per agent>
**Estimated cost:** 3N+2 to 3N+4 calls (fast/standard: 3N+2, deep: 3N+4, triggered by low divergence only)

Proceed with this plan?
```

Caller may override agent count, angles, criteria priority, judge depth, or skip debate entirely. **Caller's decision is final.**

## Adaptive Depth (Divergence Routing)

After REBUTTAL, run 1 Haiku call to measure structural divergence on 3 axes:
- **Decision alignment** — same actions recommended? (0–100)
- **Rationale alignment** — same reasons/evidence? (0–100)
- **Risk surface overlap** — same tradeoffs/risks identified? (0–100)

This is **structural similarity**, not quality judgment. Run **after** adversarial rounds — proposals that looked similar post-PROPOSE often diverge once critique exposes different assumptions.

| Divergence | Path | Judge | Total | Rationale |
|------------|------|-------|-------|-----------|
| HIGH (decision < 40 or risk < 50) | Fast | 1 judge | 3N+2 | Adversarial rounds already tested differences |
| MEDIUM | Standard | 1 judge | 3N+2 | Default baseline |
| LOW (decision > 80 and rationale > 70) | Deep | 3 judges + blind scoring | 3N+4 | Convergent proposals may share blind spots |
| Uncertain (confidence < 50) | Fallback | 1 judge | 3N+2 | Degrade gracefully |

**Deep path:** 3 independent JUDGE instances with fresh context, blind scoring (no authorship labels), divergence-aware prompt ("these proposals converged — look harder for hidden disagreements and shared blind spots"). Aggregate: majority wins; if all 3 disagree, report "NO CONSENSUS" with all reasonings.

**Graceful degradation:** classifier fails → standard path. Deep judges disagree → majority rules, report disagreement in OUTPUT. User overrides depth → use user's choice. Failure mode is "not optimal," never "wrong answer."

## Prompt Templates

### Proponent (N in parallel)

```
You are Agent #<number> of <N> in a structured design debate.

DEBATE TOPIC: <question>
PROJECT CONTEXT: <full Debate Brief>
YOUR ANGLE: <assigned angle>
OTHER AGENTS' ANGLES: <list all N angles>

TASK:
1. Analyze independently and propose a concrete solution with specifics
2. Argue from YOUR angle — don't try to cover every angle
3. Address how your approach performs on the ranked evaluation criteria
4. Preemptively address likely counterarguments from other angles
5. Acknowledge genuine weaknesses honestly

Label output: "AGENT #<number> PROPOSAL"
```

### Critique (N in parallel)

```
CRITIQUE PHASE — You are Agent #<number>.

YOUR ANGLE: <angle>
YOUR PROPOSAL: <your original proposal>

=== OTHER AGENT PROPOSALS ===
<verbatim proposals from all other agents, labeled by number and angle>

For EACH other proposal, identify:
1. Weaknesses or gaps in reasoning
2. Unvalidated assumptions
3. Risks or edge cases missed
4. Strengths you acknowledge (be fair)

Output one critique section per other agent. Critique only — no new solutions.
```

### Rebuttal (N in parallel)

```
REBUTTAL PHASE — You are Agent #<number>.

YOUR ORIGINAL PROPOSAL: <full proposal>

=== CRITIQUES OF YOUR PROPOSAL ===
<verbatim critiques from ALL other agents>

For each point raised:
1. Valid critique → concede and explain mitigation
2. Misses context → explain why with specific reasoning
3. Wrong → refute with evidence or logic

Group by critic. No entirely new arguments. Respond directly.
```

### Divergence Check (1 Haiku call)

```
Structural divergence classifier. Measure similarity, NOT quality.

DEBATE TOPIC: <question>

=== ALL PROPOSALS ===
<all N proposals>

=== ALL REBUTTALS ===
<all rebuttals>

Score each axis 0–100:
1. DECISION ALIGNMENT — same/convergent actions? (0=completely different, 100=identical)
2. RATIONALE ALIGNMENT — same reasons/evidence/values? (0=different, 100=identical)
3. RISK SURFACE OVERLAP — same tradeoffs/risks/edge cases? (0=different, 100=identical)

OUTPUT ONLY:
decision_alignment: <0–100>
rationale_alignment: <0–100>
risk_overlap: <0–100>
confidence: <0–100>
```

Routing: LOW = decision > 80 AND rationale > 70 → DEEP. HIGH = decision < 40 OR risk < 50 → FAST. Else STANDARD. Confidence < 50 → STANDARD.

### Judge (Standard / Fast)

```
You are the Judge. Final and binding decision.

DEBATE TOPIC: <question>
NUMBER OF AGENTS: N
JUDGE MODE: <standard | fast>
EVALUATION CRITERIA (ranked): <from brief>

DIVERGENCE: decision=<score>, rationale=<score>, risk=<score>

=== AGENT PROPOSALS ===
<all N proposals, verbatim>

=== CRITIQUES ===
<all critiques>

=== REBUTTALS ===
<all rebuttals>

OUTPUT:
WINNER: [Agent #N / HYBRID (Agent #X + Agent #Y)]

REASONING:
- Score on each criterion, key deciding factors
- Which arguments survived critique, which collapsed
- Why winner beats each loser

IMPLEMENTATION PLAN:
- Architecture, key components, data flow, migration path, risk mitigation
```

### Judge (Deep — 3 in parallel)

Same as standard, plus:

```
JUDGE MODE: deep (proposals classified as HIGHLY CONVERGENT — potential shared blind spots)

ADDITIONAL INSTRUCTIONS:
1. Before evaluating, list 3 potential blind spots ALL proposals may share
2. Score each proposal per criterion (1–10) BEFORE comparing
3. Flag criteria where proposals score identically — genuine agreement or shared oversight?
4. If you find a shared blind spot, explain mitigation

You are Judge #<X> of 3. Your scores will be averaged with 2 other independent judges.
```

Aggregate: 3/3 or 2/3 agree → winner stands. All 3 disagree → "NO CONSENSUS", report all reasonings.

## Decision Logic

| Scenario | Action |
|----------|--------|
| One dominates all criteria | Declare winner |
| Different winners per criterion | Weigh by ranked criteria |
| Equally valid | HYBRID — combine best parts (specify which parts, how they integrate, why better) |
| All flawed | Explain why, propose new option |
| N > 2, some inferior | Eliminate explicitly, compare remainder |

## Early Termination

| Condition | Action |
|-----------|--------|
| Proposals converge on same solution | Skip to JUDGE |
| Fatal flaw exposed in CRITIQUE | Skip REBUTTAL, go to DIVERGE → JUDGE |
| All fail hard constraint from brief | Abort, refine brief |
| Time-sensitive, 2+ rounds done | JUDGE early with available evidence |

Explain which condition was met.

## Artifacts & Verbatin Rule

```
debate-brief.md → debate-plan.md → agent-N-proposal.md (×N) → agent-N-critiques-all.md (×N) → agent-N-rebuttal.md (×N) → divergence-check.md → judge-decision.md
```

**Never summarize or paraphrase between phases.** Full text verbatim is what makes the debate valuable.

## Output & Handoff

1. **Save** judge's decision (WINNER, REASONING, IMPLEMENTATION PLAN) to `{project_root}/docs/debate/{yyyy-MM-dd}-{topic-slug}.md`
2. **Return** to caller: winner, document path, plan summary. Caller decides next step.
3. **If HYBRID:** reconcile partial approaches before writing document — extract winning parts, produce unified design.
4. **Standalone:** if user only wants debate result, stop after Step 1.
5. **Do NOT git-add the output file.** The orchestrator must never stage or commit `docs/debate/` files. If the caller wants them in version control, they commit them manually.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Vague topic | Narrow to specific question with clear tradeoffs |
| Same angle for multiple agents | Assign distinct perspectives |
| Skipping critique | Critique is where weaknesses surface |
| Judge gets summary | Provide verbatim transcript |
| Debating trivial/one-sided decisions | Don't debate — explain and move on |
| No Debate Brief | Write brief first — prevents scope drift |
| No confirmation | Always present plan and get approval |
| Always 2 agents | Calculate breadth/depth; systemic needs 3–5 |
| Over-assigning for simple decisions | B1+D1 = 2 agents; don't waste calls |
| Divergence before adversarial rounds | Run AFTER critique+rebuttal (prevents false convergence) |
| Full model for divergence | Haiku for structural similarity; it's clustering, not reasoning |
| Deep path too often | Conservative thresholds, calibrate from shadow data |
| Ignoring classifier confidence | Confidence < 50 → fallback to standard |
| Deep judges share context | Each gets fresh context window for independence |

## Example (Abbreviated)

Real outputs are 5-10x longer.

**Brief:** "Redis Streams vs RabbitMQ for payment callbacks at ~5k msg/s peak?"
- Context: Go, PostgreSQL, Redis in use. 2-person ops team. At-least-once, <2s p99.
- Criteria: (1) Correctness under failure (2) Ops simplicity (3) Throughput at 5k msg/s.
- Scope: MQ choice only.

**Size:** B=1 (binary), D=2 (cross-cutting) → 2 agents. Cost: 3×2+2 to 3×2+4 = 8–10 calls.

**Angles:** Agent #1: operational simplicity + existing infra. Agent #2: message durability + mature routing.

**Divergence (post-rebuttal):** decision=25, rationale=30, risk=45, confidence=90 → HIGH → FAST path (8 calls).

**Judge:** WINNER = Agent #1 (Redis Streams). Ops simplicity decisive for 2-person team. Redis Streams meets correctness via consumer groups + PEL + XCLAIM. RabbitMQ's DLX advantage acknowledged but doesn't outweigh ops burden at this scale.

**Plan:** `streams` package, XADD/XREADGROUP/XCLAIM, 64KB message cap, retry-counter DLQ pattern, ops runbook section.
