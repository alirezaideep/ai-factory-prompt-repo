# State Transitions

## Execution State Machine

```
                    ┌──────────────────────────────────────────┐
                    │                                          │
    ┌─────────┐    │  ┌──────────┐   ┌─────────┐   ┌──────┐ │
    │ created │────┼─▶│ planning │──▶│ planned │──▶│review│ │
    └─────────┘    │  └──────────┘   └─────────┘   └──────┘ │
                    │       │                           │      │
                    │       ▼                           ▼      │
                    │  ┌──────────┐              ┌──────────┐ │
                    │  │  failed  │◀─────────────│ rejected │ │
                    │  └──────────┘              └──────────┘ │
                    │       ▲                           │      │
                    │       │                           ▼      │
                    │       │         ┌──────────┐  ┌────────┐│
                    │       ├─────────│ running  │◀─│approved││
                    │       │         └──────────┘  └────────┘│
                    │       │              │                    │
                    │       │              ▼                    │
                    │       │         ┌──────────┐             │
                    │       └─────────│completing│             │
                    │                 └──────────┘             │
                    │                      │                    │
                    │                      ▼                    │
                    │                 ┌──────────┐             │
                    │                 │completed │             │
                    │                 └──────────┘             │
                    │                      │                    │
                    │                      ▼                    │
                    │                 ┌──────────┐             │
                    │                 │  merged  │             │
                    │                 └──────────┘             │
                    └──────────────────────────────────────────┘
```

## Valid Transitions

| From | To | Trigger | Guard Conditions |
|------|----|---------|-----------------|
| `created` | `planning` | User submits goal | Valid goal format |
| `planning` | `planned` | AI generates plan | Plan passes validation |
| `planning` | `failed` | AI fails to plan | Max retries exceeded |
| `planned` | `review` | Plan ready for review | Auto-transition |
| `review` | `approved` | Human approves | Has approval authority |
| `review` | `rejected` | Human rejects | Has approval authority |
| `rejected` | `planning` | User requests re-plan | Feedback provided |
| `approved` | `running` | Orchestrator starts | Resources available |
| `running` | `completing` | All steps done | All steps succeeded/skipped |
| `running` | `failed` | Unrecoverable error | Retry policy exhausted |
| `running` | `review` | Step needs approval | Step.needsReview = true |
| `completing` | `completed` | Validation passes | Output validated |
| `completing` | `failed` | Validation fails | Critical issues found |
| `completed` | `merged` | Changes merged to repo | Merge conflict resolved |

## Step State Machine

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌───────────┐
│ pending │──▶│ queued  │──▶│ running │──▶│ completed │
└─────────┘   └─────────┘   └─────────┘   └───────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │  failed  │ │ retrying │ │  paused  │
              └──────────┘ └──────────┘ └──────────┘
                    │             │             │
                    ▼             ▼             ▼
              ┌──────────┐ ┌─────────┐   ┌─────────┐
              │ skipped  │ │ running │   │ running │
              └──────────┘ └─────────┘   └─────────┘
```

### Step Transitions

| From | To | Trigger | Side Effects |
|------|----|---------|-------------|
| `pending` | `queued` | Dependencies met | Notify scheduler |
| `queued` | `running` | Agent assigned | Start timer, allocate budget |
| `running` | `completed` | Agent finishes | Store output, notify dependents |
| `running` | `failed` | Agent error | Log error, check retry policy |
| `running` | `paused` | Needs human input | Send notification |
| `failed` | `retrying` | Retry policy allows | Increment retry count |
| `failed` | `skipped` | Skip policy or manual | Unblock dependents (if allowed) |
| `retrying` | `running` | After backoff delay | Reset timer |
| `paused` | `running` | Human provides input | Resume with new context |

## State Persistence
```typescript
// Every state transition is persisted atomically
async function transition(
  entityId: string,
  entityType: 'execution' | 'step',
  fromState: string,
  toState: string,
  trigger: string,
  metadata?: Record<string, any>
): Promise<void> {
  await db.transaction(async (tx) => {
    // 1. Verify current state matches expected
    const current = await tx.query(
      `SELECT status FROM ${entityType}s WHERE id = ? FOR UPDATE`,
      [entityId]
    );
    
    if (current.status !== fromState) {
      throw new ConflictError(
        `Expected ${fromState}, found ${current.status}`
      );
    }
    
    // 2. Update state
    await tx.query(
      `UPDATE ${entityType}s SET status = ?, updated_at = NOW() WHERE id = ?`,
      [toState, entityId]
    );
    
    // 3. Record transition in history
    await tx.query(
      `INSERT INTO state_transitions (entity_id, entity_type, from_state, to_state, trigger, metadata, created_at)
       VALUES (?, ?, ?, ?, ?, ?, NOW())`,
      [entityId, entityType, fromState, toState, trigger, JSON.stringify(metadata)]
    );
  });
  
  // 4. Emit event (outside transaction)
  await eventBus.emit(`${entityType}.state_changed`, {
    entityId, fromState, toState, trigger, metadata
  });
}
```

## Concurrency Control
```typescript
// Optimistic locking for state transitions
interface Execution {
  id: string;
  status: ExecutionStatus;
  version: number;  // Incremented on every update
}

async function safeTransition(id: string, expectedVersion: number, newStatus: string) {
  const result = await db.query(
    `UPDATE executions SET status = ?, version = version + 1 
     WHERE id = ? AND version = ?`,
    [newStatus, id, expectedVersion]
  );
  
  if (result.affectedRows === 0) {
    throw new ConflictError('Concurrent modification detected');
  }
}
```
