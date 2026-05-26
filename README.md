# Claude Code Skills

A collection of skills that enhance the Claude Code development workflow.

## Skills

| Skill | Purpose |
|-------|---------|
| `agent-debate` | Launch structured multi-agent debate when multiple valid design approaches exist |
| `design-debate-review` | Review a plan/prompt before execution to find hidden decision points, annotate with `[Debate topic]` markers |

## Skill Details

### agent-debate

Use when encountering design decisions with multiple valid approaches, architectural tradeoffs, or when two implementation strategies need objective comparison during development.

**Core principle:** Two independent analyses + structured debate produces better decisions than a single agent's judgment. Adaptive depth routes JUDGE scrutiny based on measured proposal divergence — lighter for clearly distinct proposals, deeper when proposals converge (potential shared blind spots).

**The Debate Flow (9 phases):**

```
1. IDENTIFY   — narrow to a specific, scoped question, write a Debate Brief
2. SIZE       — calculate required agent count from breadth and depth scores
3. CONFIRM    — present the plan to the caller before launching
4. PROPOSE    — dispatch N agents in parallel, each from a different angle
5. CRITIQUE   — each agent reads all other proposals, identifies weaknesses
6. REBUTTAL   — each agent responds to critiques of their own proposal
7. DIVERGENCE — lightweight classifier measures structural divergence, routes JUDGE depth
8. JUDGE      — fast/standard/deep path based on divergence; winner + implementation plan
9. OUTPUT     — save winning plan to docs/debate/yyyy-MM-dd-{topic}.md, return to caller
```

**Adaptive depth routing:**

| Divergence | Path | Calls | Rationale |
|------------|------|-------|-----------|
| High (clearly distinct) | Fast | 3N+2 | Adversarial rounds already stress-tested well |
| Medium | Standard | 3N+2 | Default — proven baseline behavior |
| Low (convergent — blind spot risk) | Deep | 3N+4 | 3-judge ensemble + blind scoring for independent scrutiny |
| Classifier uncertain | Fallback | 3N+2 | Degrade gracefully to standard path |

**Output:** The winning implementation plan is saved to `docs/debate/` with the naming format `yyyy-MM-dd-{topic-slug}.md`. The document path and plan are returned to the caller (user or downstream skill), which decides the next step — typically chaining into `writing-plans` or `executing-plans`.

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

**Cost:** 3N+2 to 3N+4 agent calls (fast/standard: 3N+2, deep: 3N+4). For 2-agent debate: 8–10 calls. Deep path only triggers when proposals are highly convergent (shared blind spot risk).

### design-debate-review

Before submitting a plan or prompt for execution, throw it here first. This skill scans the content, finds where decisions are implicit or ambiguous, and returns:

1. **Annotated original text** — the plan/prompt with `[Debate <topic>]` markers inline at each decision point
2. **Decision point list** — what needs debate, why, and dependencies

From there, the user decides which topics to debate. Optionally chains into `agent-debate` for each confirmed topic.

**Process (4 steps):**
1. REVIEW — scan the plan against 4 dimensions (design impact, future constraints, multiple options, hidden assumptions)
2. EXTRACT — annotate the original text with `[Debate <topic>]` markers, define each topic as a specific question
3. OUTPUT — return annotated plan + decision point table, ask user what to debate
4. EXECUTE — run debates in parallel via `agent-debate` (optional, user-driven)

## Usage

Invoke any skill with `/<skill-name>` in Claude Code.
