# Agent Runtime — UI Wireframes

## Agent Registry Page
```
┌─────────────────────────────────────────────────────────────┐
│ [Logo] AI Factory    Agents  Executions  Projects  Settings │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Agent Registry                    [+ Register Agent]       │
│  ─────────────────────────────────────────────────────      │
│                                                             │
│  Filter: [All Types ▼] [All Models ▼] [Active ▼]          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ 🤖 Code Gen  │  │ 🧪 Tester    │  │ 📝 Reviewer  │     │
│  │ claude-sonnet│  │ claude-haiku │  │ claude-sonnet│     │
│  │ ● Active     │  │ ● Active     │  │ ● Active     │     │
│  │              │  │              │  │              │     │
│  │ Success: 94% │  │ Success: 97% │  │ Success: 91% │     │
│  │ Tasks: 342   │  │ Tasks: 891   │  │ Tasks: 256   │     │
│  │ Avg: $0.63   │  │ Avg: $0.08   │  │ Avg: $0.45   │     │
│  │ ▁▃▅▇▅▆▇▅▆▇ │  │ ▁▃▅▇▅▆▇▅▆▇ │  │ ▁▃▅▇▅▆▇▅▆▇ │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ 🏗️ DevOps    │  │ 📚 Docs      │  │ 🔍 Analyzer  │     │
│  │ gpt-4o-mini  │  │ claude-haiku │  │ claude-sonnet│     │
│  │ ● Active     │  │ ○ Paused     │  │ ● Active     │     │
│  │ ...          │  │ ...          │  │ ...          │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Agent Detail Page
```
┌─────────────────────────────────────────────────────────────┐
│ ← Back to Agents    Code Generator v2         [● Active ▼] │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─ Config ──────────────────┐  ┌─ Metrics (30d) ────────┐│
│  │ Model: claude-sonnet-4    │  │                         ││
│  │ Tools: file_write, ...    │  │  Success ████████░░ 94% ││
│  │ Budget: 100K tokens/task  │  │  Self-fix ██████░░░ 72% ││
│  │ Timeout: 300s             │  │  Escalate █░░░░░░░░  6% ││
│  │ Retries: 3 (exponential)  │  │                         ││
│  │                           │  │  Cost/task: $0.63 avg   ││
│  │ [Edit Config]             │  │  Tokens: 42K avg        ││
│  └───────────────────────────┘  └─────────────────────────┘│
│                                                             │
│  ┌─ Recent Executions ──────────────────────────────────── ┐│
│  │ ID        │ Status  │ Duration │ Tokens │ Cost │ Date   ││
│  │ task-a1b2 │ ✅ Done │ 45s      │ 38K    │ $0.57│ 2h ago ││
│  │ task-c3d4 │ ✅ Done │ 62s      │ 51K    │ $0.76│ 3h ago ││
│  │ task-e5f6 │ ❌ Fail │ 120s     │ 89K    │ $1.34│ 5h ago ││
│  │ task-g7h8 │ ✅ Done │ 33s      │ 29K    │ $0.44│ 6h ago ││
│  └──────────────────────────────────────────────────────── ┘│
└─────────────────────────────────────────────────────────────┘
```
