---
name: agent-debate
description: Use when encountering design decisions with multiple valid approaches, architectural tradeoffs, or when two implementation strategies need objective comparison during development
---

# Agent Debate

## Overview

When a design decision has multiple valid approaches, dispatch two agents to independently analyze and debate, then use a judge agent to select the best approach.

**Core principle:** Two independent analyses + structured debate produces better decisions than a single agent's judgment.

## When to Use

- Architecture decision with multiple valid options
- API design choices (pagination strategy, data format, etc.)
- Algorithm/performance tradeoff decisions
- Two different implementation strategies both seem reasonable
- User explicitly requests a debate ("debate this")
- Plan or agent signals `[DEBATE: topic]` or `NEEDS_DEBATE: topic`

**Don't use when** only one correct approach exists, or the decision is trivial.

## Trigger Modes

| Mode | Signal | Action |
|------|--------|--------|
| **Checkpoint** | Plan marks `[DEBATE: topic]` | Launch debate when reaching this step |
| **Agent-initiated** | Agent signals `NEEDS_DEBATE: topic` | Pause execution, launch debate on the topic |
| **User-initiated** | User says "debate this" or "辩论" | Launch debate on the current decision point |

## Cost Estimation (run BEFORE launching debate)

Agent count is determined by decision complexity (see Agent Count Calculation below). Each phase consumes:

| Phase | Agent Calls | Parallelizable |
|-------|-------------|----------------|
| Propose | N | Yes — dispatch together |
| Critique | N | Yes — dispatch together |
| Rebuttal | N | Yes — dispatch together |
| Judge | 1 | No — needs all prior output |
| **Total** | **3N+1** | 3 parallel rounds + 1 final |

**Minimum 3 sequential rounds** before a decision is reached. For a 2-agent debate: 7 calls. For a 5-agent debate: 16 calls.

For trivial decisions where one approach is clearly correct, skip the debate and explain why.

## The Debate Flow

```
1. IDENTIFY — narrow to a specific, scoped question, write a Debate Brief
   Bad:  "What queue should we use?"
   Good: "Redis Streams vs RabbitMQ for payment callback async processing?"

2. SIZE — calculate required agent count from breadth and depth scores
   Determine how many proponent agents are needed (2–5)
   Estimate total agent calls for the full debate

3. CONFIRM — present the plan to the caller before launching
   "This debate needs N agents, ~X calls across Y rounds. Proceed?"
   Caller may override agent count, angle assignments, or evaluation criteria
   Caller's decision is final — if they say 2 agents, use 2 agents

4. PROPOSE — dispatch N agents in parallel
   Each agent argues from a DIFFERENT assigned angle
   Each proposes a concrete solution addressing the debate question

5. CRITIQUE — each agent reads all other proposals (parallel)
   Identify weaknesses, gaps, unvalidated assumptions, risks
   Acknowledge genuine strengths fairly

6. REBUTTAL — dispatch N agents in parallel
   Each agent responds to critiques of their own proposal
   Defend valid points, concede genuine weaknesses

7. JUDGE — fresh agent
   Reads full debate record → selects winner → explains reasoning → outputs implementation plan

8. OUTPUT — save winning implementation plan to docs/debate/yyyy-MM-dd-{topic}.md
   Return the document path and implementation plan to the caller for next steps
```

## Debate Brief Template

Before launching any agents, the orchestrator MUST write a Debate Brief. This scopes the debate and prevents drift.

```markdown
## Debate Brief

**Question:** <one specific, answerable question>

**Context:**
- Project tech stack: <language, framework, existing infra>
- Relevant constraints: <latency requirements, scale, team expertise, deadlines>
- Existing patterns in codebase: <what's already in use, what would be inconsistent>

**Option A:** <name and one-line summary>
**Option B:** <name and one-line summary>

**Evaluation Criteria (ranked):**
1. <top priority: e.g., correctness under concurrent writes>
2. <second priority: e.g., operational simplicity>
3. <third priority: e.g., performance at 10k msg/s>

**Scope boundaries:** <what is OUT of scope for this debate>

**Angle assignments:**
- Agent A argues for Option A from: <e.g., simplicity, maintainability, team familiarity>
- Agent B argues for Option B from: <e.g., scalability, fault tolerance, future-proofing>
```

The Debate Brief is included in each agent's prompt as PROJECT CONTEXT.

## Agent Count Calculation

After writing the Debate Brief, calculate how many proponent agents the debate needs. Use two dimensions:

### Breadth — how many distinct viable options or paradigms exist?

| Score | Description | Example |
|-------|-------------|---------|
| 1 | Binary choice (A vs B) | Redis Streams vs RabbitMQ |
| 2 | Multiple clear options (A vs B vs C) | JWT vs Opaque Token vs PASETO for auth |
| 3 | Open design space (many permutations, no fixed options) | "How should we redesign our data pipeline?" |

### Depth — how far-reaching is the decision's impact?

| Score | Description | Example |
|-------|-------------|---------|
| 1 | Local — affects a single module/component | Pick a date library for one service |
| 2 | Cross-cutting — affects multiple services or layers | Pick an RPC serialization format across 5 services |
| 3 | Systemic — affects overall architecture or tech stack direction | Pick primary database for a new platform |

### Formula

```
agent_count = breadth_score + depth_score - 1
             clamped to range [2, 5]
```

| Breadth | Depth | Agents | Rationale |
|---------|-------|--------|-----------|
| 1 | 1 | **2** | Simple binary choice, local impact |
| 1 | 2 | **2** | Binary but cross-cutting — 2 perspectives sufficient |
| 2 | 1 | **2** | Multiple options, local — still constrained |
| 2 | 2 | **3** | Multiple options, cross-cutting — need a third angle |
| 2 | 3 | **4** | Multiple options, systemic — high stakes demand diversity |
| 3 | 2 | **4** | Open design, cross-cutting — need multiple perspectives |
| 3 | 3 | **5** | Open design, systemic — maximum coverage needed |

### Angle Assignment for 3+ Agents

With 3+ agents, angles must be distributed to avoid overlap. Use distinct dimensions:

- **Simplicity / DX** — ease of use, learning curve, team familiarity
- **Performance / Scale** — throughput, latency, resource efficiency
- **Reliability / Correctness** — fault tolerance, consistency, edge case safety
- **Flexibility / Evolvability** — extensibility, future-proofing, ecosystem
- **Operations / Cost** — monitoring, incident response, infrastructure spend

Pick N angles from this pool based on the evaluation criteria in the brief. Each agent gets exactly one angle.

## Confirmation Gate

Before launching any debate agents, the orchestrator MUST present a summary to the caller and get explicit approval.

### What to Present

```
## Debate Plan

**Question:** <from brief>
**Agent count:** N proponent agents
**Angles:** <one per agent>
**Estimated cost:** 3N+1 = X total agent calls across 4 rounds

Proceed with this plan?
```

### Caller Responses

| Response | Action |
|----------|--------|
| "Yes" / "Proceed" / "Go ahead" | Launch the debate |
| "Use M agents instead" | Recalculate with caller's number; reassign angles if needed |
| "Change angle X to Y" | Adjust angle assignment, then proceed |
| "Skip debate, just do X" | Abort debate, follow caller's direction |
| "Change criteria priority" | Update the brief, recalculate if needed |

**Caller's decision always takes highest priority.** If the caller overrides the agent count, use their number — the calculation is a recommendation, not a mandate.

## Prompt Templates

### Proponent

Dispatch N Agent() calls in parallel (N = calculated agent count). Each gets:

```
You are Agent #<number> of <N> in a structured design debate.

DEBATE TOPIC: <specific question>

PROJECT CONTEXT:
<Debate Brief contents — include the full brief>

YOUR ANGLE: <assigned angle, unique among all agents>

OTHER AGENTS' ANGLES:
- Agent #1 argues from: <angle>
- Agent #2 argues from: <angle>
- ...

TASK:
1. Independently analyze the problem using the project context
2. Propose a concrete solution with specifics (not hand-waving)
3. Argue why your approach wins when evaluated from YOUR angle — don't try to cover every angle
4. Address how your approach performs on the ranked evaluation criteria from the brief
5. Preemptively address likely counterarguments from the OTHER angles listed above
6. Acknowledge any genuine weaknesses of your approach (be honest)

Output your full proposal with reasoning. Label it "AGENT #<number> PROPOSAL".
```

### Critique

Feed each agent ALL other agents' proposals (verbatim):

```
CRITIQUE PHASE — You are Agent #<number>. Read all other proposals critically.

YOUR ANGLE: <your assigned angle>
YOUR PROPOSAL: <your original proposal for reference>

=== OTHER AGENT PROPOSALS ===
<verbatim outputs from all other agents, labeled by agent number and angle>

For EACH other proposal, identify:
1. Weaknesses or gaps in their reasoning
2. Unvalidated assumptions
3. Risks or edge cases they missed
4. Strengths you acknowledge (be fair — don't dismiss valid points)

Output one critique section per other agent. Critique only — do not propose new solutions.
```

### Rebuttal

Feed each agent ALL critiques of their own proposal:

```
REBUTTAL PHASE — You are Agent #<number>. Your proposal has been critiqued by other agents.

YOUR ORIGINAL PROPOSAL:
<your full proposal>

=== CRITIQUES OF YOUR PROPOSAL ===
<verbatim critiques from ALL other agents, labeled by agent number>

For each point raised against your proposal:
1. If the critique is valid: concede and explain how it could be mitigated
2. If the critique misses context: explain why with specific reasoning
3. If the critique is wrong: refute with evidence or logic

Address critiques grouped by critic. DO NOT introduce entirely new arguments.
Respond to the critiques directly.
```

### Judge

Dispatch a fresh Agent() with the complete debate record:

```
You are the Judge. Your decision is final and binding.

DEBATE TOPIC: <original question from brief>
NUMBER OF AGENTS: N

EVALUATION CRITERIA (ranked):
<copied from Debate Brief>

=== AGENT #1 PROPOSAL (angle: <angle>) ===
...
=== AGENT #2 PROPOSAL (angle: <angle>) ===
...
<repeat for all N agents>

=== CRITIQUES ===
<all critique outputs, labeled by critic and target>

=== REBUTTALS ===
<all rebuttal outputs, labeled by agent>

OUTPUT FORMAT:

WINNER: [Agent #N / HYBRID (Agent #X + Agent #Y)]

REASONING:
- How each agent's approach scored on each evaluation criterion
- Key deciding factors — which criteria were most decisive?
- Which arguments survived critique and which collapsed
- Why the winning approach is better than each losing approach

IMPLEMENTATION PLAN:
<concrete, actionable description ready for the writing-plans skill>
- Architecture: <high-level structure>
- Key components: <what to build>
- Data flow: <how data moves>
- Migration path: <if replacing existing system>
- Risk mitigation: <biggest risks and how to handle them>
```

## Decision Logic

### Picking a Winner

| Scenario | Action |
|----------|--------|
| One approach dominates on all criteria | Declare that agent's approach winner |
| Different approaches win on different criteria | Weigh by criteria ranking; top criteria carry more weight |
| Multiple approaches are equally valid | Declare HYBRID — combine the best parts of each |
| All approaches are flawed | Judge explains why none work, proposes a new option |
| N > 2 and some approaches are clearly inferior | Eliminate them explicitly in reasoning, compare the remaining |

### HYBRID Decisions

A HYBRID winner means combining complementary strengths from both proposals. The judge MUST specify:

1. **Which parts** come from which approach (be specific, not "take the best of both")
2. **How they integrate** — what's the interface between them?
3. **Why hybrid is better** — what would be lost by picking only A or only B?

Example: "Use Agent A's data model (simpler schema, better query patterns) with Agent B's retry mechanism (exponential backoff + DLQ). The data model and retry logic are orthogonal — combining them has no downside."

### Early Termination

Debate phases can be truncated when:

| Condition | Action |
|-----------|--------|
| Both proposals converge on the same solution | Skip to JUDGE immediately; confirm alignment |
| One proposal has a fatal, irreparable flaw exposed in CRITIQUE | Skip REBUTTAL; go to JUDGE, note the fatal flaw |
| Both proposals fail to address a hard constraint from the brief | Abort debate; refine the brief or explore a different option space |
| The decision is time-sensitive and 2+ rounds have been completed | JUDGE can be called early with available evidence |

When terminating early, explain which condition was met so the user understands the shortcut.

### Debate State Tracking

The orchestrator tracks debate state through these artifacts:

```
debate-brief.md              — written in IDENTIFY, fixed for the debate duration
debate-plan.md               — agent count, angles, cost estimate (presented to caller)
agent-N-proposal.md          — one per agent, captured from Propose phase
agent-N-critiques-all.md     — one per agent, each critiques all other proposals
agent-N-rebuttal.md          — one per agent, responds to all critiques of their proposal
judge-decision.md            — final output from Judge phase
```

For a 3-agent debate, this is: brief + plan + 3 proposals + 3 critiques + 3 rebuttals + 1 decision = 12 artifacts.

Each artifact is passed verbatim to the next phase. Never summarize or paraphrase — the full text is what makes the debate valuable.

## Output & Handoff

After the judge produces a decision and implementation plan, save the result as a document and then hand off to implementation skills.

### Step 1: Save Decision Document

Write the judge's full decision (WINNER, REASONING, IMPLEMENTATION PLAN) to the project's `docs/debate/` directory.

**Directory:** `{project_root}/docs/debate/` (create if it doesn't exist)

**Filename format:** `{yyyy-MM-dd}-{topic-slug}.md`

Derive the topic slug from the debate question:
- Lowercase and replace spaces / special characters with hyphens
- Keep it concise (under ~50 characters)
- Example: "Redis Streams vs RabbitMQ for payment callbacks" → `redis-streams-vs-rabbitmq`

Full path example: `docs/debate/2026-05-25-redis-streams-vs-rabbitmq.md`

**Document template:**

```markdown
# Debate Decision: {original debate question}

**Date:** {yyyy-MM-dd}
**Winner:** {Agent #N or HYBRID (Agent #X + Agent #Y)}
**Angles considered:** {list of angles, one per agent}

## Reasoning

{judge's full reasoning — how each approach scored, key deciding factors,
which arguments survived critique, why winner beats each loser}

## Implementation Plan

{judge's full implementation plan — architecture, key components, data flow,
migration path, risk mitigation}
```

### Step 2: Return to Caller

After saving the document, return the result to the calling skill or user. Do NOT assume the next step — the caller decides what to do with the output.

**Return payload:**

```
Debate complete.

**Winner:** {Agent #N or HYBRID}
**Decision document:** docs/debate/{yyyy-MM-dd}-{topic-slug}.md
**Implementation plan summary:** {brief summary of the plan}

The implementation plan in the decision document is ready for downstream skills (e.g., writing-plans, superpowers:writing-plans, executing-plans) if the caller chooses to proceed with implementation.
```

This keeps agent-debate composable: any skill (writing-plans, superpowers:writing-plans, superpowers:executing-plans, or user directly) can consume the output and decide what to do next.

### Common Downstream Paths

| Caller | Typical next step |
|--------|-------------------|
| User directly | User reviews the document and decides whether to implement |
| writing-plans | Reads the decision document, invokes writing-plans for the plan |
| superpowers:writing-plans | Reads the decision document, produces implementation plan |
| superpowers:executing-plans | Reads the plan from the document and executes |
| Other skill | Each skill handles the output according to its own workflow |

### Step 3: Retain Debate Artifacts

The debate artifacts (brief, proposals, critiques, rebuttals, decision) serve as the rationale documentation — reference them in the PR description so reviewers understand WHY the approach was chosen.

### Standalone Mode

If the user only wants the debate result without implementation, stop after Step 1 (save the document). The decision document in `docs/debate/` serves as the authoritative record of the decision.

### If Judge Declares HYBRID

The orchestrator MUST reconcile the two partial approaches before writing the document. Read both original proposals, extract the winning parts as specified by the judge, produce a unified design, then save the document and return the result to the caller.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Vague debate topic | Narrow to a specific question with clear tradeoffs |
| Both agents get same angle | Assign different perspectives (simplicity vs performance, minimal vs flexible) |
| Skipping critique phase | The debate IS the value; critique is where weaknesses surface |
| Judge gets summary instead of full record | Provide verbatim debate transcript |
| Debating when one approach is clearly wrong | Don't debate — just explain why and move on |
| Launching debate without cost check | Estimate agent calls first; skip debate for trivial decisions |
| Continuing debate after convergence | If both agents agree in proposals, skip to JUDGE immediately |
| HYBRID without integration rules | Judge must specify exactly which parts combine and how |
| Summarizing agent output between phases | Always pass verbatim; summarization loses detail that matters in critique |
| No Debate Brief before launching | Write the brief first — it prevents scope drift and ensures fair angle assignment |
| Launching without caller confirmation | Always present the Debate Plan and get approval; caller may have different preferences |
| Always defaulting to 2 agents | Calculate breadth/depth scores; complex systemic decisions need 3–5 agents |
| Over-assigning agents for simple decisions | Breadth 1 + Depth 1 = 2 agents; don't waste calls on straightforward choices |

## Complete Example

This is an abbreviated illustration of the debate structure. Real outputs would be 5-10x longer.

### IDENTIFY — Debate Brief

```markdown
**Question:** Redis Streams vs RabbitMQ for payment callback async processing at ~5k msg/s peak?

**Context:**
- Tech stack: Go services, PostgreSQL, Redis already in use
- Constraints: at-least-once delivery, <2s p99 processing latency, ops team of 2
- Existing patterns: Redis caching layer exists, no message queue in production

**Evaluation Criteria (ranked):**
1. Correctness — message loss/replay behavior under node failure
2. Operational simplicity — can a 2-person ops team manage this at 3am?
3. Performance — sustained throughput at 5k msg/s with headroom

**Scope boundaries:** Message queue choice only; payment processing logic is out of scope
```

### SIZE — Agent Count Calculation

```
Breadth score: 1 (binary choice — Redis Streams vs RabbitMQ)
Depth score:   2 (cross-cutting — affects payment service, monitoring, ops runbooks)
Agent count:   1 + 2 - 1 = 2 agents
Total calls:   3×2 + 1 = 7 agent calls across 4 rounds
```

### CONFIRM — Present to Caller

```
## Debate Plan

**Question:** Redis Streams vs RabbitMQ for payment callback async processing at ~5k msg/s peak?
**Agent count:** 2
**Angles:**
  - Agent #1: operational simplicity, leverage existing infra
  - Agent #2: message durability guarantees, mature routing patterns
**Estimated cost:** 7 agent calls across 4 rounds

Proceed with this plan?
```

### Agent #1 Proposal (angle: operational simplicity) — abbreviated

```
WIN FOR CORRECTNESS: Redis Streams with consumer groups + explicit ACK gives
at-least-once semantics. Pending Entries List (PEL) tracks unacknowledged messages.
If a consumer dies, another claims its pending entries via XCLAIM — no message loss.

WIN FOR OPERATIONS: Redis is already in our stack. Ops team knows Redis. No new
service to monitor, patch, or wake up for. One less moving part in the incident
response playbook.

WEAKNESS: Streams are not a dedicated MQ. No built-in dead-letter queue.
Long messages (>1MB) hurt Redis latency for other use cases. Mitigation:
cap message size at 64KB, build a simple DLQ pattern with XREADGROUP + retry counter.
```

### Agent #2 Proposal (angle: message durability) — abbreviated

```
WIN FOR CORRECTNESS: RabbitMQ's publisher confirms + consumer acknowledgments
are battle-tested for financial workloads. Dead Letter Exchanges (DLX) are native.
Message TTL, queue length limits, and overflow behavior are configurable per queue.

WIN FOR PERFORMANCE: RabbitMQ handles 20k+ msg/s on modest hardware. Separate
broker means queue workload doesn't compete with cache workload — no noisy neighbor.

WEAKNESS: New infrastructure. Ops team needs RabbitMQ training. Cluster recovery
after network partitions requires understanding of split-brain handling.
```

### Judge Decision (abbreviated)

```
WINNER: Agent #1 (Redis Streams)

REASONING:
- Correctness (criteria #1): Both achieve at-least-once. Agent #2's DLX is more mature,
  but Agent #1's PEL + XCLAIM is sufficient for payment callbacks with explicit ACK.
  Agent #1's weakness was acknowledged and mitigation is viable. Slight edge to #2, but
  not decisive.
- Operations (criteria #2): This is the deciding factor. A 2-person ops team
  cannot absorb RabbitMQ's cluster management overhead. Agent #2 never fully
  refuted this — their rebuttal argued "RabbitMQ is stable once configured"
  which misses the 3am failure scenario. Decisive win for Agent #1.
- Performance (criteria #3): 5k msg/s is within Redis Streams comfort zone.
  Agent #2's throughput advantage is real but irrelevant at this scale. Agent #1 wins by
  meeting requirements without overprovisioning.

IMPLEMENTATION PLAN:
- Create a `streams` package in the payment service
- Use XADD for publishing, XREADGROUP for consuming in a consumer group
- Implement XCLAIM-based claim mechanism for failed consumer recovery
- Add message size validation (64KB cap) at publish boundary
- Build a retry-counter pattern using a secondary stream as DLQ
- Write a runbook section for "Redis Streams consumer lag alert"
```
