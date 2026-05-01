# Context Pack: Orchestration

## Purpose
Provides patterns and conventions for the orchestration layer — DAG execution, agent coordination, state management, and error recovery.

## Files

| File | Description | Load When |
|------|-------------|-----------|
| `execution_patterns.md` | DAG execution, parallel steps, dependencies | Orchestrator changes |
| `agent_coordination.md` | Agent selection, context assembly, handoffs | Agent-related work |
| `state_transitions.md` | Execution/step state machines, valid transitions | State logic changes |
| `error_recovery.md` | Retry, fallback, circuit breaker, human escalation | Error handling work |
| `resource_management.md` | Token budgets, concurrency limits, cost tracking | Resource/cost work |
