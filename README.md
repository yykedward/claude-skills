# Claude Code Skills

A collection of skills that enhance the Claude Code development workflow.

## Skills

| Skill | Purpose |
|-------|---------|
| `agent-debate` | Launch structured multi-agent debate when multiple valid design approaches exist |

## Skill Details

### agent-debate

Use when encountering design decisions with multiple valid approaches, architectural tradeoffs, or when two implementation strategies need objective comparison during development.

**Core principle:** Two independent analyses + structured debate produces better decisions than a single agent's judgment.

**The Debate Flow (7 phases):**

```
1. IDENTIFY  — narrow to a specific, scoped question, write a Debate Brief
2. SIZE      — calculate required agent count from breadth and depth scores
3. CONFIRM   — present the plan to the caller before launching
4. PROPOSE   — dispatch N agents in parallel, each from a different angle
5. CRITIQUE  — each agent reads all other proposals, identifies weaknesses
6. REBUTTAL  — each agent responds to critiques of their own proposal
7. JUDGE     — fresh agent selects winner, explains reasoning, outputs implementation plan
```

**Agent count** is calculated from the decision's breadth (1-3) and depth (1-3):

```
agent_count = breadth_score + depth_score - 1  (clamped to [2, 5])
```

**Trigger modes:**
| Mode | Signal | Action |
|------|--------|--------|
| Checkpoint | Plan marks `[DEBATE: topic]` | Launch debate at that step |
| Agent-initiated | Agent signals `NEEDS_DEBATE: topic` | Pause, launch debate |
| User-initiated | User says "debate this" or "辩论" | Launch debate on current decision |

**Cost:** 3N+1 agent calls in 4 rounds (min 7 calls for 2-agent debate).

## Usage

Invoke any skill with `/<skill-name>` in Claude Code.
