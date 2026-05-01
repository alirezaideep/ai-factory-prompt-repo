# Orchestrator — UI Wireframes

## Execution Monitor Page
```
┌─────────────────────────────────────────────────────────────┐
│ [Logo] AI Factory    Tasks  Executions  Agents  Analytics   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Execution: Add Authentication     [● Running]  [⏸ Pause]  │
│  Project: E-commerce App  │  Started: 15 min ago           │
│  Budget: $1.20 / $3.20 (38%)  │  Steps: 4/12 complete     │
│                                                             │
│  ┌─ DAG Visualization ─────────────────────────────────┐   │
│  │                                                     │   │
│  │  [✅ Schema]──→[✅ Models]──→[● Auth Svc]──→[○ API] │   │
│  │                     │              ↓            ↓    │   │
│  │                     └──→[✅ Types]  [○ Middle]  [○ Test]│   │
│  │                                     ↓            ↓    │   │
│  │                              [○ Integration Tests]    │   │
│  │                                     ↓                 │   │
│  │                              [○ Documentation]        │   │
│  │                                                       │   │
│  │  Legend: ✅=Done  ●=Running  ○=Pending  ❌=Failed     │   │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─ Current Step: Auth Service ─────────────────────────┐  │
│  │ Agent: Code Generator v2  │  Running for: 45s        │  │
│  │ Tokens: 12,400 / 80,000  │  Status: Generating...    │  │
│  │                                                       │  │
│  │ Live Output:                                          │  │
│  │ ┌─────────────────────────────────────────────────┐  │  │
│  │ │ Creating src/services/auth.ts...                │  │  │
│  │ │ Implementing validateToken()...                 │  │  │
│  │ │ Adding password hashing with bcrypt...          │  │  │
│  │ │ █                                               │  │  │
│  │ └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Execution History Page
```
┌─────────────────────────────────────────────────────────────┐
│ Executions                          [Filter ▼] [Search 🔍]  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  │ Status │ Task              │ Steps │ Cost  │ Duration │  │
│  │────────│───────────────────│───────│───────│──────────│  │
│  │ ✅ Done│ Add auth          │ 12/12 │ $2.84 │ 42 min   │  │
│  │ ● Run  │ Payment gateway   │ 4/8   │ $1.20 │ 15 min   │  │
│  │ ✅ Done│ Update docs       │ 3/3   │ $0.45 │ 5 min    │  │
│  │ ❌ Fail│ Migrate database  │ 2/6   │ $0.92 │ 18 min   │  │
│  │ ⏸ Pause│ Refactor API      │ 5/10  │ $1.55 │ 28 min   │  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
