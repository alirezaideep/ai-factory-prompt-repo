# Planning Service — UI Wireframes

## Plan Builder Page
```
┌─────────────────────────────────────────────────────────────┐
│ [Logo] AI Factory    New Task                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  What do you want to build?                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Add JWT-based authentication with login, register,  │   │
│  │ and password reset. Include email verification and  │   │
│  │ rate limiting on auth endpoints.                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Project: [E-commerce App ▼]  Priority: [Normal ▼]         │
│                                                             │
│  [Generate Plan →]                                          │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Generated Plan (v2 — after feedback)         [Regenerate]  │
│  ─────────────────────────────────────────────────────      │
│                                                             │
│  Steps: 12  │  Est. Duration: 45 min  │  Est. Cost: $3.20  │
│  Risk: Medium  │  Agents: 3 types                          │
│                                                             │
│  ┌─ Step List ──────────────────────────────────────────┐  │
│  │ 1. [DB] Create auth tables schema         ~$0.15     │  │
│  │ 2. [Code] Generate TypeScript models      ~$0.25     │  │
│  │ 3. [Code] Implement auth service          ~$0.65     │  │
│  │ 4. [Code] Create API endpoints            ~$0.45     │  │
│  │ 5. [Code] Add middleware                  ~$0.35     │  │
│  │ 6. [Code] Email verification service      ~$0.40     │  │
│  │ 7. [Code] Rate limiting middleware        ~$0.25     │  │
│  │ 8. [Test] Unit tests                      ~$0.30     │  │
│  │ 9. [Test] Integration tests               ~$0.20     │  │
│  │ 10. [Docs] API documentation              ~$0.10     │  │
│  │ 11. [Review] Code review                  ~$0.05     │  │
│  │ 12. [DevOps] Update deployment config     ~$0.05     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Feedback: [                                             ]  │
│  [Send Feedback]     [✅ Approve & Execute]                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Plan Iteration History
```
┌─────────────────────────────────────────────────────────────┐
│ Plan Iterations                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  v1 (initial) ──→ v2 (after feedback) ──→ v3 (approved) ✅ │
│                                                             │
│  ┌─ v1 → v2 Changes ───────────────────────────────────┐  │
│  │ Feedback: "Split auth service into smaller steps"    │  │
│  │ Changes:                                             │  │
│  │  - Step 3 split into 3a (JWT) + 3b (sessions)       │  │
│  │  - Added step 6 (email verification)                 │  │
│  │  - Budget increased from $2.80 to $3.20              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
