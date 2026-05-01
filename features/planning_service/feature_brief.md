# Planning Service — Feature Brief

## Module Identity

| Field | Value |
|-------|-------|
| Module ID | `planning_service` |
| Layer | 2 (AI Planning) |
| Owner | Backend Team Lead |
| Status | Planned |
| Priority | Critical (core intelligence) |

## Purpose

The Planning Service is the **brain** of the AI Software Factory. It receives a human goal (task) and produces a structured Plan Schema — a JSON document that defines exactly what needs to be done, by which agents, in what order, with what dependencies, estimated cost, and risk assessment.

## Core Capabilities

### 1. Goal Decomposition
- Parse natural language goal into structured sub-tasks
- Identify affected modules from goal description
- Determine task dependencies (what must happen before what)
- Estimate complexity per sub-task

### 2. Agent Assignment
- Match sub-tasks to appropriate agents (code, test, review, docs, devops)
- Consider agent specialization and current load
- Assign parallel vs sequential execution based on dependencies

### 3. DAG Construction
- Build Directed Acyclic Graph of execution steps
- Identify critical path (longest dependency chain)
- Optimize for parallelism where possible
- Validate DAG (no cycles, all nodes reachable)

### 4. Cost Estimation
- Estimate token usage per step (based on historical data)
- Calculate USD cost (per-model pricing)
- Estimate compute time per step
- Total cost with confidence interval

### 5. Risk Assessment
- Score risk 0.0 to 1.0 based on factors:
  - Modifies shared/core modules → +0.2
  - No existing tests for affected area → +0.2
  - Breaking API change → +0.3
  - First-time pattern (no historical data) → +0.1
  - Large scope (>10 steps) → +0.1
  - Touches authentication/security → +0.2

### 6. Context Selection
- Determine which prompt repository files are needed for each step
- Use Context Loading Table (shared/app.md) as guide
- Minimize context window usage (only load what's needed)
- Prioritize most relevant files per agent per step

## Plan Schema (Output Format)

```json
{
  "plan_id": "plan_xyz789",
  "task_id": "task_abc123",
  "version": 1,
  "created_at": "2026-01-15T10:30:00Z",
  "status": "generated",
  "summary": "Add urgent order support with priority processing",
  "affected_modules": ["order_management", "notifications"],
  "dag": {
    "nodes": [
      {
        "id": "step_1",
        "type": "execution",
        "agent": "code_agent",
        "description": "Add priority field to order data model",
        "context_files": [
          "features/order_management/back/data_model.md",
          "shared/backend.md"
        ],
        "estimated_tokens": 4000,
        "estimated_minutes": 2,
        "dependencies": []
      },
      {
        "id": "step_2",
        "type": "execution",
        "agent": "code_agent",
        "description": "Create database migration for priority field",
        "context_files": [
          "features/order_management/back/data_model.md",
          "shared/versioning.md"
        ],
        "estimated_tokens": 2000,
        "estimated_minutes": 1,
        "dependencies": ["step_1"]
      },
      {
        "id": "step_3",
        "type": "execution",
        "agent": "code_agent",
        "description": "Implement urgent order API endpoints",
        "context_files": [
          "features/order_management/back/api_endpoints.md",
          "shared/backend.md"
        ],
        "estimated_tokens": 6000,
        "estimated_minutes": 3,
        "dependencies": ["step_2"]
      },
      {
        "id": "step_4",
        "type": "execution",
        "agent": "code_agent",
        "description": "Implement notification trigger for urgent orders",
        "context_files": [
          "features/notifications/back/channels.md",
          "features/notifications/workflows/trigger_flow.md"
        ],
        "estimated_tokens": 4000,
        "estimated_minutes": 2,
        "dependencies": ["step_3"]
      },
      {
        "id": "step_5",
        "type": "execution",
        "agent": "test_agent",
        "description": "Write unit and integration tests",
        "context_files": [
          "context_packs/backend/testing_strategy.md",
          "features/order_management/back/api_endpoints.md"
        ],
        "estimated_tokens": 5000,
        "estimated_minutes": 3,
        "dependencies": ["step_3", "step_4"]
      },
      {
        "id": "step_6",
        "type": "execution",
        "agent": "review_agent",
        "description": "Code review all changes",
        "context_files": [
          "context_packs/memory/conventions.md",
          "shared/backend.md"
        ],
        "estimated_tokens": 3000,
        "estimated_minutes": 2,
        "dependencies": ["step_5"]
      },
      {
        "id": "step_7",
        "type": "execution",
        "agent": "code_agent",
        "description": "Implement frontend urgent order UI",
        "context_files": [
          "features/order_management/front/pages.md",
          "shared/frontend.md",
          "shared/design.md"
        ],
        "estimated_tokens": 5000,
        "estimated_minutes": 3,
        "dependencies": ["step_3"]
      },
      {
        "id": "step_8",
        "type": "execution",
        "agent": "docs_agent",
        "description": "Update documentation and changelog",
        "context_files": [
          "features/order_management/feature_brief.md",
          "shared/versioning.md",
          "changes/VERSIONING_GUIDE.md"
        ],
        "estimated_tokens": 2000,
        "estimated_minutes": 1,
        "dependencies": ["step_6", "step_7"]
      }
    ],
    "edges": [
      {"from": "step_1", "to": "step_2"},
      {"from": "step_2", "to": "step_3"},
      {"from": "step_3", "to": "step_4"},
      {"from": "step_3", "to": "step_5"},
      {"from": "step_4", "to": "step_5"},
      {"from": "step_5", "to": "step_6"},
      {"from": "step_3", "to": "step_7"},
      {"from": "step_6", "to": "step_8"},
      {"from": "step_7", "to": "step_8"}
    ]
  },
  "cost_estimate": {
    "total_tokens": 31000,
    "estimated_usd": 0.93,
    "confidence": 0.8,
    "breakdown_by_agent": {
      "code_agent": {"tokens": 19000, "usd": 0.57},
      "test_agent": {"tokens": 5000, "usd": 0.15},
      "review_agent": {"tokens": 3000, "usd": 0.09},
      "docs_agent": {"tokens": 2000, "usd": 0.06}
    }
  },
  "risk_assessment": {
    "score": 0.4,
    "level": "medium",
    "factors": [
      {"factor": "Modifies shared order module", "weight": 0.2},
      {"factor": "New workflow pattern (no history)", "weight": 0.1},
      {"factor": "Touches notification system", "weight": 0.1}
    ],
    "mitigations": [
      "Integration tests cover critical path",
      "Code review step before merge",
      "Rollback plan: revert migration"
    ]
  },
  "timeline": {
    "critical_path": ["step_1", "step_2", "step_3", "step_5", "step_6", "step_8"],
    "estimated_total_minutes": 12,
    "parallelizable_steps": [["step_4", "step_7"], ["step_5"]]
  }
}
```

## Technical Architecture

### Service Design
```
┌─────────────────────────────────────────────┐
│ Planning Service (FastAPI)                   │
├─────────────────────────────────────────────┤
│ POST /v1/plans/generate                      │
│ GET  /v1/plans/:id                           │
│ POST /v1/plans/:id/regenerate                │
│ GET  /v1/plans/:id/versions                  │
├─────────────────────────────────────────────┤
│ ┌─────────────┐  ┌─────────────────────┐   │
│ │ Goal Parser │  │ Context Selector    │   │
│ │ (LLM call)  │  │ (rule-based + LLM)  │   │
│ └─────────────┘  └─────────────────────┘   │
│ ┌─────────────┐  ┌─────────────────────┐   │
│ │ DAG Builder │  │ Cost Estimator      │   │
│ │ (algorithm) │  │ (historical + LLM)  │   │
│ └─────────────┘  └─────────────────────┘   │
│ ┌─────────────┐  ┌─────────────────────┐   │
│ │ Risk Scorer │  │ Plan Validator      │   │
│ │ (rule-based)│  │ (DAG validation)    │   │
│ └─────────────┘  └─────────────────────┘   │
├─────────────────────────────────────────────┤
│ Dependencies: PostgreSQL, Redis, Claude API  │
└─────────────────────────────────────────────┘
```

### LLM Prompting Strategy
1. **System prompt:** Role definition + output format (JSON Schema)
2. **Context injection:** Relevant repository files (from Context Loading Table)
3. **Task description:** User's goal + scope + constraints
4. **Historical examples:** Similar past plans (from Knowledge Base)
5. **Output validation:** JSON Schema validation + DAG cycle detection

## Dependencies

| Depends On | Reason |
|-----------|--------|
| Knowledge Base | Historical plans, similar tasks |
| Prompt Repository | Context files for planning |
| Auth Service | User permissions |
| Claude API | LLM for decomposition |

## Success Metrics

| Metric | Target |
|--------|--------|
| Plan generation time | < 60 seconds |
| Plan approval rate (first attempt) | > 70% |
| Cost estimation accuracy | ±20% |
| DAG validity | 100% (no cycles) |
| Context relevance score | > 0.8 |
