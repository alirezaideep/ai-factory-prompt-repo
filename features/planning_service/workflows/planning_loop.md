# Planning Service — Workflows

## Planning Loop (Iterative)

```
┌─────────────────────────────────────────────────────────────┐
│                    PLANNING LOOP                              │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│  │  Receive │───▶│ Generate │───▶│ Validate │             │
│  │  Task    │    │  Plan    │    │  DAG     │             │
│  └──────────┘    └──────────┘    └──────────┘             │
│                       ▲               │                     │
│                       │               ▼                     │
│                  ┌──────────┐    ┌──────────┐             │
│                  │  Apply   │◀───│  Human   │             │
│                  │ Feedback │    │  Review  │             │
│                  └──────────┘    └──────────┘             │
│                                       │                     │
│                                       ▼                     │
│                                  ┌──────────┐             │
│                                  │ Approved │──▶ EXIT      │
│                                  └──────────┘             │
│                                                             │
│  Max iterations: 5                                          │
│  Escalation: After 5 iterations → Project Owner            │
└─────────────────────────────────────────────────────────────┘
```

## Detailed Flow

### Phase 1: Task Reception
```
Trigger: POST /api/v1/planning/generate
Input: task_id, goal, scope, constraints

Actions:
1. Validate task exists and is in 'submitted' status
2. Acquire generation lock (Redis, TTL 120s)
3. Update task status → 'planning'
4. Emit event: task.status_changed
5. Begin async generation
```

### Phase 2: Plan Generation
```
Actions:
1. Parse goal → structured sub-tasks (LLM call #1)
2. Select context files → list of repo paths
3. Load context files from repository
4. Build DAG → nodes + edges (LLM call #2)
5. Validate DAG (no cycles, all reachable)
6. Estimate cost (rule-based + historical)
7. Assess risk (rule-based)
8. Calculate timeline (critical path analysis)
9. Save plan to database (version 1)
10. Update task status → 'plan_ready'
11. Emit event: plan.generated
12. Release generation lock
```

### Phase 3: Human Review
```
Trigger: Human opens plan in Command UI
UI Shows: DAG visualization, cost, risk, timeline

Possible outcomes:
A) Approve → plan.status = 'approved', task.status = 'approved'
B) Modify → feedback captured, plan.status = 'superseded', goto Phase 4
C) Reject → plan.status = 'rejected', task.status = 'plan_rejected', goto Phase 4
```

### Phase 4: Feedback Integration
```
Trigger: Human provides feedback (modify or reject with suggestions)

Actions:
1. Parse feedback into structured modifications
2. Load current plan + feedback
3. Regenerate plan (LLM call with previous plan + feedback)
4. Increment version
5. Validate new DAG
6. Recalculate cost and risk
7. Save new version
8. Emit event: plan.regenerated
9. Return to Phase 3

Exit conditions:
- Approved → proceed to Orchestrator
- Max iterations reached → escalate to Project Owner
- Cancelled → task.status = 'cancelled'
```

## State Machine

```
States: generating | generated | awaiting_approval | approved | rejected | superseded | failed

Transitions:
  generating → generated (generation complete)
  generating → failed (LLM error, timeout)
  generated → awaiting_approval (sent to human)
  awaiting_approval → approved (human approves)
  awaiting_approval → rejected (human rejects)
  awaiting_approval → superseded (human requests modification)
  superseded → generating (regeneration started)
  rejected → generating (regeneration with feedback)
  failed → generating (retry)
```

## Error Handling

| Error | Recovery | Max Retries |
|-------|----------|-------------|
| LLM timeout | Retry with shorter context | 3 |
| LLM invalid JSON | Retry with stricter prompt | 2 |
| DAG cycle detected | Retry with explicit acyclic constraint | 2 |
| Context file not found | Skip file, add warning | — |
| Lock acquisition failed | Wait 5s, retry | 3 |
| Database write failed | Retry with backoff | 3 |

## Metrics Collected

| Metric | Type | Purpose |
|--------|------|---------|
| `planning.generation_time_ms` | Histogram | Track generation speed |
| `planning.iterations_per_task` | Counter | Track how many attempts needed |
| `planning.approval_rate` | Gauge | First-attempt approval rate |
| `planning.cost_accuracy` | Gauge | Estimated vs actual cost |
| `planning.llm_tokens_used` | Counter | Track LLM consumption |
| `planning.dag_size` | Histogram | Steps per plan distribution |
