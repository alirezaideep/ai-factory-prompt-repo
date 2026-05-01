# Frontend Architecture

## Overview

The AI Software Factory frontend is a Next.js 14 application providing the Command UI — the human interface for task submission, plan visualization, approval workflows, execution monitoring, and feedback review.

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | Next.js 14 (App Router) | React 18+ |
| Language | TypeScript 5.x | Strict mode |
| Styling | Tailwind CSS 4 + shadcn/ui | Utility-first |
| State | Zustand + React Query (TanStack) | Client + server state |
| Forms | React Hook Form + Zod | Validation |
| Charts | Recharts + D3.js | Data visualization |
| DAG Viz | ReactFlow | Plan/workflow visualization |
| Real-time | Socket.IO client | Live updates |
| Testing | Vitest + Playwright | Unit + E2E |
| Bundler | Turbopack (via Next.js) | Fast builds |

## Application Structure

```
client/
├── app/                          ← Next.js App Router
│   ├── (auth)/                   ← Auth layout group
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/              ← Main dashboard layout
│   │   ├── tasks/                ← Task management
│   │   ├── plans/                ← Plan visualization
│   │   ├── approvals/            ← Approval queue
│   │   ├── prompts/              ← Prompt repository browser
│   │   ├── executions/           ← Execution monitoring
│   │   ├── knowledge/            ← Knowledge base management
│   │   ├── feedback/             ← Feedback & analytics
│   │   └── settings/             ← Platform settings
│   ├── api/                      ← BFF API routes
│   └── layout.tsx                ← Root layout
├── components/
│   ├── ui/                       ← shadcn/ui primitives
│   ├── shared/                   ← Shared components
│   ├── tasks/                    ← Task-specific components
│   ├── plans/                    ← Plan visualization components
│   ├── approvals/                ← Approval components
│   ├── prompts/                  ← Prompt browser components
│   ├── executions/               ← Execution monitoring components
│   └── feedback/                 ← Feedback components
├── hooks/                        ← Custom React hooks
├── lib/                          ← Utilities, API client, constants
├── stores/                       ← Zustand stores
├── types/                        ← TypeScript type definitions
└── styles/                       ← Global styles, theme
```

## Design Principles

### Component Architecture
- **Atomic Design:** atoms → molecules → organisms → templates → pages
- **Server Components by default:** Only use `"use client"` when interactivity needed
- **Composition over inheritance:** Use render props and compound components
- **Colocation:** Keep related files together (component + hook + types + test)

### State Management Strategy
| State Type | Solution | Example |
|-----------|----------|---------|
| Server state | TanStack Query | API data, task lists |
| UI state | Zustand | Sidebar open, active tab |
| Form state | React Hook Form | Task submission form |
| URL state | Next.js searchParams | Filters, pagination |
| Real-time | Socket.IO + Zustand | Execution progress |

### Data Fetching Pattern
```typescript
// Server Component (default)
async function TaskList() {
  const tasks = await api.tasks.list();
  return <TaskTable data={tasks} />;
}

// Client Component (interactive)
"use client";
function TaskActions({ taskId }: { taskId: string }) {
  const { mutate } = useMutation(api.tasks.approve);
  return <Button onClick={() => mutate(taskId)}>Approve</Button>;
}
```

## Page Structure

| Route | Purpose | Key Components |
|-------|---------|---------------|
| `/tasks` | Task submission & management | TaskForm, TaskList, TaskDetail |
| `/tasks/new` | Create new task | TaskWizard (multi-step) |
| `/plans` | View generated plans | PlanGraph (ReactFlow), PlanTimeline |
| `/plans/[id]` | Plan detail + approval | PlanDAG, ApprovalPanel, CostEstimate |
| `/approvals` | Approval queue | ApprovalList, ApprovalDetail, DiffView |
| `/prompts` | Browse prompt repository | FileTree, PromptEditor, VersionHistory |
| `/executions` | Monitor agent execution | ExecutionTimeline, LogViewer, AgentStatus |
| `/executions/[id]` | Single execution detail | StepProgress, OutputPreview, ErrorPanel |
| `/knowledge` | Knowledge base management | DocumentUpload, EmbeddingStatus, SearchTest |
| `/feedback` | Analytics & feedback | MetricsDashboard, SuccessRateChart, EditHistory |
| `/settings` | Platform configuration | TeamManagement, APIKeys, LLMConfig |

## Real-time Updates

### WebSocket Events
```typescript
// Connection
const socket = io(WS_URL, { auth: { token } });

// Events consumed by frontend
socket.on("execution.step.started", (data) => { ... });
socket.on("execution.step.completed", (data) => { ... });
socket.on("execution.failed", (data) => { ... });
socket.on("approval.requested", (data) => { ... });
socket.on("plan.generated", (data) => { ... });
```

### Optimistic Updates
- Task status changes: immediate UI update, rollback on failure
- Approval actions: optimistic with confirmation toast
- File edits in prompt browser: debounced save with conflict detection

## Accessibility & Responsiveness

- WCAG 2.1 AA compliance
- Keyboard navigation for all interactive elements
- Screen reader support via ARIA labels
- Responsive breakpoints: mobile (640px), tablet (768px), desktop (1024px), wide (1280px)
- RTL support ready (for Persian interface option)

## Performance Targets

| Metric | Target |
|--------|--------|
| LCP | < 2.5s |
| FID | < 100ms |
| CLS | < 0.1 |
| Bundle size (initial) | < 200KB gzipped |
| Time to interactive | < 3s |

## Error Handling (Frontend)

- Global error boundary with recovery UI
- Per-route error boundaries (Next.js `error.tsx`)
- Toast notifications for non-blocking errors
- Full-page error for critical failures
- Retry buttons with exponential backoff
- Offline detection with queue for pending actions
