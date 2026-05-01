# Command UI — Frontend Components

## Component Hierarchy

### Layout Components
| Component | Props | Description |
|-----------|-------|-------------|
| `DashboardLayout` | `children` | Sidebar + header + main content |
| `Sidebar` | `collapsed, items, activeItem` | Navigation sidebar |
| `Header` | `title, breadcrumbs, actions` | Page header with search |
| `CommandPalette` | `open, onClose` | Cmd+K quick navigation |

### Task Components
| Component | Props | Description |
|-----------|-------|-------------|
| `TaskWizard` | `onSubmit, draft?` | Multi-step task creation |
| `TaskCard` | `task, onClick` | Task summary card |
| `TaskTable` | `tasks, filters, sort` | Sortable/filterable task list |
| `TaskStatusBadge` | `status` | Colored status indicator |
| `TaskPriorityBadge` | `priority` | Priority level indicator |
| `TaskDetail` | `taskId` | Full task detail view |

### Plan Components
| Component | Props | Description |
|-----------|-------|-------------|
| `PlanDAG` | `nodes, edges, onNodeClick` | ReactFlow DAG visualization |
| `PlanTimeline` | `steps, currentStep` | Gantt-style timeline |
| `PlanCostPanel` | `estimate` | Cost breakdown display |
| `PlanRiskPanel` | `assessment` | Risk heat map |
| `PlanNodeDetail` | `node, onComment` | Selected node detail panel |
| `ApprovalActions` | `planId, onDecision` | Approve/Modify/Reject buttons |

### Execution Components
| Component | Props | Description |
|-----------|-------|-------------|
| `ExecutionTimeline` | `steps, currentStep` | Vertical step timeline |
| `LogViewer` | `executionId, stepId` | Real-time log streaming |
| `ProgressBar` | `completed, total` | Step progress indicator |
| `CostCounter` | `tokens, usd, time` | Live cost display |
| `AgentStatusCard` | `agent, status` | Agent activity indicator |
| `ExecutionActions` | `execId, status` | Pause/Retry/Abort buttons |

### Repository Components
| Component | Props | Description |
|-----------|-------|-------------|
| `FileTree` | `tree, selected, onSelect` | Expandable file navigator |
| `FileViewer` | `path, content, version` | Markdown renderer |
| `FileEditor` | `path, content, onSave` | CodeMirror editor |
| `VersionHistory` | `path, versions` | Version list with dates |
| `DiffViewer` | `path, fromVer, toVer` | Side-by-side diff |
| `CommitDialog` | `files, onCommit` | Commit message + version bump |

### Feedback Components
| Component | Props | Description |
|-----------|-------|-------------|
| `FeedbackForm` | `executionId, stepId` | Rating + comments form |
| `SuccessRateChart` | `data, period` | Line chart over time |
| `AgentComparisonChart` | `agents, metric` | Bar chart comparing agents |
| `EditRateChart` | `data` | How much AI output was edited |

### Shared/UI Components
| Component | Props | Description |
|-----------|-------|-------------|
| `StatCard` | `label, value, trend, icon` | Metric display card |
| `EmptyState` | `icon, title, description, action` | No-data placeholder |
| `LoadingSkeleton` | `variant` | Loading placeholder |
| `ConfirmDialog` | `title, message, onConfirm` | Destructive action confirmation |
| `NotificationBell` | `count, items` | Notification dropdown |
| `SearchInput` | `placeholder, onSearch` | Global search with suggestions |

## State Management

### Zustand Stores
```typescript
// stores/ui.ts
interface UIStore {
  sidebarCollapsed: boolean;
  theme: 'dark' | 'light';
  activeModal: string | null;
  toggleSidebar: () => void;
  setTheme: (theme: 'dark' | 'light') => void;
}

// stores/notifications.ts
interface NotificationStore {
  notifications: Notification[];
  unreadCount: number;
  markAsRead: (id: string) => void;
  markAllRead: () => void;
}

// stores/execution.ts
interface ExecutionStore {
  activeExecutions: Map<string, ExecutionState>;
  logs: Map<string, LogEntry[]>;
  appendLog: (execId: string, entry: LogEntry) => void;
  updateStep: (execId: string, stepId: string, status: StepStatus) => void;
}
```

### TanStack Query Keys
```typescript
const queryKeys = {
  tasks: {
    all: ['tasks'],
    list: (filters) => ['tasks', 'list', filters],
    detail: (id) => ['tasks', 'detail', id],
  },
  plans: {
    byTask: (taskId) => ['plans', 'task', taskId],
    detail: (planId) => ['plans', 'detail', planId],
  },
  executions: {
    all: ['executions'],
    detail: (id) => ['executions', 'detail', id],
    logs: (id, stepId) => ['executions', 'logs', id, stepId],
  },
  repo: {
    tree: ['repo', 'tree'],
    file: (path) => ['repo', 'file', path],
    history: (path) => ['repo', 'history', path],
  },
};
```
