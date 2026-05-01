# Command UI — Workflows

## Task Lifecycle (from UI perspective)

### States
```
draft → submitted → planning → plan_ready → approving → approved → executing → completed
                                    ↓                       ↓           ↓
                              plan_rejected            modified     failed
                                    ↓                       ↓           ↓
                              (back to planning)    (back to planning) (retry/abort)
```

### State Transitions (UI Actions)

| Current State | User Action | Next State | API Call |
|--------------|-------------|------------|----------|
| — | Click "New Task" | draft | — |
| draft | Fill form + Submit | submitted | POST /tasks |
| submitted | (automatic) | planning | — |
| planning | (wait for AI) | plan_ready | WebSocket event |
| plan_ready | Click "Review" | approving | — |
| approving | Click "Approve" | approved | POST /plans/:id/approve |
| approving | Click "Modify" | modified | POST /plans/:id/modify |
| approving | Click "Reject" | plan_rejected | POST /plans/:id/reject |
| modified | (automatic) | planning | — |
| plan_rejected | (automatic) | planning | — |
| approved | (automatic) | executing | — |
| executing | (wait for agents) | completed | WebSocket event |
| executing | Click "Abort" | failed | POST /executions/:id/abort |
| failed | Click "Retry" | executing | POST /executions/:id/retry |

## Approval Workflow (Iterative)

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐   │
│  │ Planning │────▶│ Review   │────▶│ Approved │   │
│  │ Service  │     │ (Human)  │     │          │   │
│  └──────────┘     └──────────┘     └──────────┘   │
│       ▲                │                            │
│       │                │ Modify/Reject              │
│       │                ▼                            │
│       │         ┌──────────┐                        │
│       └─────────│ Feedback │                        │
│                 │ (Human)  │                        │
│                 └──────────┘                        │
│                                                     │
│  This loop repeats until approved or cancelled      │
└─────────────────────────────────────────────────────┘
```

### Iteration Rules
- Maximum 5 iterations before escalation to Project Owner
- Each iteration preserves previous comments/feedback
- Plan diff shown between iterations (what changed)
- Cost re-estimated on each iteration

## Repository Merge Workflow (Team Lead)

```
1. Execution completes → Output ready for review
2. Team Lead receives notification
3. Team Lead opens output in Repository Browser
4. Team Lead reviews changes (diff view)
5. If acceptable:
   a. Team Lead clicks "Merge"
   b. System prompts for commit message
   c. System prompts for version bump type (major/minor/patch)
   d. System auto-generates changelog entry
   e. Files updated in repository
   f. Version bumped
6. If needs changes:
   a. Team Lead edits directly in editor
   b. Then proceeds with merge (step 5)
7. If rejected:
   a. Team Lead provides feedback
   b. Re-enters execution loop
```

## Notification Triggers

| Event | Recipients | Channel | Priority |
|-------|-----------|---------|----------|
| Plan ready for review | Team Lead | In-app + Slack | High |
| Plan approved | Task submitter | In-app | Medium |
| Plan rejected | Task submitter | In-app + Slack | High |
| Execution started | Task submitter | In-app | Low |
| Execution failed | Team Lead + submitter | In-app + Slack | Critical |
| Execution completed | Team Lead | In-app | High |
| Repository updated | All team members | In-app | Low |
| Cost threshold reached | Project Owner | In-app + Email | Critical |
