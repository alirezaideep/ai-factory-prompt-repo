# Orchestrator — Frontend Pages

## Page: Execution Monitor (`/executions/:id`)

### Purpose
Real-time visualization of execution progress. Shows DAG as a graph with step statuses, live logs, and control buttons.

### Layout
```
┌─────────────────────────────────────────────────────────┐
│ [← Back] Execution: {plan_title}     [Pause] [Cancel]  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────────────────────────┐  ┌──────────┐ │
│  │         DAG Visualization           │  │ Summary  │ │
│  │                                     │  │          │ │
│  │  [step1]──→[step2]──→[step4]       │  │ 5/12 ✓  │ │
│  │              ↗                      │  │ 2 running│ │
│  │  [step5]  [step3]──→              │  │ 5 pending│ │
│  │                                     │  │          │ │
│  │  Legend: ● done ◐ running ○ pending │  │ Cost: $2 │ │
│  │          ✕ failed ⊘ skipped        │  │ ETA: 8m  │ │
│  └─────────────────────────────────────┘  └──────────┘ │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step Details (click a node above)                      │
│  ┌─────────────────────────────────────────────────────┐│
│  │ Step: step_2 (code_agent)  Status: Running          ││
│  │ Started: 2 min ago  Tokens: 3,200  Agent: Claude    ││
│  │                                                     ││
│  │ Live Output:                                        ││
│  │ > Reading data_model.md...                          ││
│  │ > Generating migration file...                      ││
│  │ > Writing src/migrations/004_add_field.sql          ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### Data Sources
- `GET /api/v1/orchestrator/executions/:id` — Initial load
- `WebSocket /ws/orchestrator/executions/:id` — Real-time updates
- `GET /api/v1/orchestrator/executions/:id/logs` — Step logs

### Interactions
- Click node → show step details panel
- Pause button → `POST /api/v1/orchestrator/executions/:id/pause`
- Cancel button → `POST /api/v1/orchestrator/executions/:id/cancel`
- Retry failed step → `POST /api/v1/orchestrator/executions/:id/retry-step`

---

## Page: Execution History (`/executions`)

### Purpose
List all past and current executions with filtering and search.

### Layout
```
┌─────────────────────────────────────────────────────────┐
│ Executions                    [Filter ▾] [Search...]    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Status │ Plan Title        │ Steps │ Cost  │ Duration  │
│  ───────┼───────────────────┼───────┼───────┼────────── │
│  ✓ Done │ Add priority field│ 8/8   │ $1.20 │ 18 min   │
│  ◐ Run  │ Payment module    │ 3/12  │ $0.80 │ 5 min    │
│  ✕ Fail │ Auth refactor     │ 5/7   │ $2.10 │ 25 min   │
│  ⊘ Cncl │ Remove legacy API │ 2/6   │ $0.30 │ 3 min    │
│                                                         │
│  [1] [2] [3] ... [12]  Showing 1-20 of 234             │
└─────────────────────────────────────────────────────────┘
```

### Filters
- Status: All | Running | Completed | Failed | Cancelled | Paused
- Date range
- Project
- Agent type involved
- Cost range

---

## Page: Execution Comparison (`/executions/compare`)

### Purpose
Compare two executions side-by-side (useful for A/B testing prompts or retry analysis).

### Layout
```
┌──────────────────────┬──────────────────────┐
│ Execution A          │ Execution B          │
│ Plan: Add field v1   │ Plan: Add field v2   │
├──────────────────────┼──────────────────────┤
│ Status: Failed       │ Status: Success      │
│ Steps: 5/8           │ Steps: 8/8           │
│ Cost: $1.80          │ Cost: $1.20          │
│ Duration: 22 min     │ Duration: 18 min     │
│ Quality: 5.0         │ Quality: 8.0         │
├──────────────────────┼──────────────────────┤
│ Diff: step_4 failed  │ step_4 succeeded     │
│ Reason: timeout      │ (expanded context)   │
└──────────────────────┴──────────────────────┘
```

---

## Components Used

| Component | Source | Purpose |
|-----------|--------|---------|
| DAGVisualization | Custom (D3.js) | Interactive graph rendering |
| StepDetailPanel | Custom | Show step info, logs, retry button |
| ExecutionTable | Shared DataTable | List with sort/filter/paginate |
| ProgressBar | UI Library | Overall progress indicator |
| LiveLog | Custom (WebSocket) | Real-time log streaming |
| CostBadge | Shared | Show cost with color coding |
| StatusBadge | Shared | Colored status indicator |
