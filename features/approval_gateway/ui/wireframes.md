# Approval Gateway — UI Wireframes

## Approval Queue Page
```
┌─────────────────────────────────────────────────────────────┐
│ [Logo] AI Factory    Tasks  Approvals(3)  Projects  Settings│
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Pending Approvals (3)              [Filter ▼] [Sort ▼]    │
│  ─────────────────────────────────────────────────────      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 🟡 Plan Review: Add user authentication             │   │
│  │ Project: E-commerce App  │  12 steps  │  ~$3.20     │   │
│  │ Risk: Medium  │  Requested: 2h ago  │  Expires: 22h │   │
│  │                                                     │   │
│  │ [View Details]  [✅ Approve]  [✏️ Modify]  [❌ Reject]│   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 🟡 Escalation: Step failed after 3 retries          │   │
│  │ Project: CRM System  │  Agent: Code Gen  │  Error   │   │
│  │ Risk: Low  │  Requested: 30m ago  │  Expires: 23.5h │   │
│  │                                                     │   │
│  │ [View Details]  [Fix & Continue]  [Skip]  [Abort]   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 🟢 Auto-approved: Update API documentation          │   │
│  │ Project: AI Factory  │  3 steps  │  ~$0.45          │   │
│  │ Risk: Low  │  Auto-approved by policy  │  1h ago    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Approval Detail Page
```
┌─────────────────────────────────────────────────────────────┐
│ ← Back    Plan Review: Add user authentication    [Pending] │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─ Plan Summary ────────────────────────────────────────┐ │
│  │ Goal: Implement JWT-based authentication with login,  │ │
│  │       registration, and password reset flows          │ │
│  │                                                       │ │
│  │ Steps: 12  │  Est. Duration: 45 min  │  Budget: $3.20│ │
│  │ Risk: Medium (touches auth, database, and API)        │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌─ Execution DAG ──────────────────────────────────────┐  │
│  │                                                       │  │
│  │  [Schema] → [Models] → [Auth Service] → [API] ──┐   │  │
│  │                              ↓                    │   │  │
│  │                         [Middleware]              │   │  │
│  │                              ↓                    ↓   │  │
│  │                         [Tests] ──────────→ [Integration]│
│  │                                                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─ Decision ────────────────────────────────────────────┐ │
│  │ Comment: [                                          ] │ │
│  │                                                       │ │
│  │ [✅ Approve]    [✏️ Modify Plan]    [❌ Reject]        │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```
