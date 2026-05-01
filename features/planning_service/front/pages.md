# Planning Service — Frontend Pages

## Note
The Planning Service frontend is embedded within the Command UI. It does not have standalone pages. Its UI components are consumed by:
- `/plans/:id` — Plan Viewer page
- `/tasks/:id` — Task Detail page (shows plan status)
- `/approvals/:id` — Approval Detail page (shows plan for review)

See `features/command_ui/front/pages.md` for full page specifications.

## Dedicated Components (used by Command UI)

### PlanGenerationStatus
Shows real-time progress of plan generation.
```
┌────────────────────────────────────┐
│ 🧠 Generating Plan...              │
│ ━━━━━━━━━━━━━━━░░░░░░ 65%         │
│                                    │
│ ✅ Goal parsed                     │
│ ✅ Context selected (5 files)      │
│ 🔄 Building execution DAG...       │
│ ⏳ Estimating cost                  │
│ ⏳ Assessing risk                   │
│                                    │
│ Estimated: 30 seconds remaining    │
└────────────────────────────────────┘
```

### PlanComparisonView
Shows diff between plan versions during iteration.
```
┌──────────────────┬──────────────────┐
│ Version 1        │ Version 2        │
│ (superseded)     │ (current)        │
├──────────────────┼──────────────────┤
│ 8 steps          │ 10 steps (+2)    │
│ $0.93            │ $1.12 (+$0.19)   │
│ 12 min           │ 14 min (+2)      │
│ Risk: 0.4        │ Risk: 0.35 (-0.05)│
├──────────────────┼──────────────────┤
│ [DAG diff view]                     │
│ + step_3b (new)                     │
│ + step_9 (new)                      │
│ ~ step_3 (modified)                 │
└─────────────────────────────────────┘
```

### ContextFilesList
Shows which repository files will be used by the plan.
```
┌────────────────────────────────────┐
│ 📁 Context Files (7 files loaded)  │
│                                    │
│ shared/backend.md                  │
│ shared/versioning.md               │
│ features/orders/back/api.md        │
│ features/orders/back/data_model.md │
│ features/orders/front/pages.md     │
│ features/notif/back/channels.md    │
│ context_packs/memory/conventions.md│
│                                    │
│ Total tokens: ~12,000              │
│ [View files] [Suggest additions]   │
└────────────────────────────────────┘
```
