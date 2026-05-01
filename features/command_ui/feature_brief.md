# Command UI — Feature Brief

## Module Identity

| Field | Value |
|-------|-------|
| Module ID | `command_ui` |
| Layer | 1 (Human Interface) |
| Owner | Frontend Team Lead |
| Status | Planned |
| Priority | Critical (foundation) |

## Purpose

The Command UI is the **single human interface** for the entire AI Software Factory. It provides task submission, plan visualization, approval workflows, execution monitoring, prompt repository browsing, and feedback collection — all in one unified dashboard.

## Core Capabilities

### 1. Task Submission
- Multi-step wizard for creating new tasks (goal, scope, constraints, priority)
- PRD/feature request upload (Markdown, Word, PDF)
- Voice-to-text task submission (future)
- Template library for common task types
- Draft saving and resumption

### 2. Plan Visualization
- Interactive DAG view (ReactFlow) showing task decomposition
- Timeline view (Gantt-style) showing estimated duration
- Cost estimation panel (token budget, compute time)
- Risk assessment visualization (heat map)
- Dependency graph highlighting critical path

### 3. Approval Workflow
- Approval queue with priority sorting
- Side-by-side diff view (before/after)
- Inline commenting on plan steps
- Approve / Modify / Reject with mandatory reason
- Batch approval for low-risk items (if FF enabled)
- Notification system (in-app + Slack/email)

### 4. Execution Monitoring
- Real-time agent activity dashboard
- Step-by-step progress with live logs
- Token usage and cost tracking (live)
- Error detection with suggested recovery
- Manual intervention points (pause, retry, abort)

### 5. Prompt Repository Browser
- File tree navigation with search
- Inline Markdown editor with preview
- Version history per file (git log style)
- Diff viewer between versions
- Commit/merge interface for team leads

### 6. Feedback & Analytics
- Execution success rate over time
- Agent performance comparison
- Cost per task/feature/module
- Human edit rate (how much AI output was modified)
- Quality metrics dashboard

### 7. Settings & Administration
- Team management (invite, roles, permissions)
- LLM provider configuration
- API key management
- Project settings (templates, defaults)
- Audit log viewer

## User Flows

### Flow 1: Submit New Task
```
Dashboard → "New Task" button → Task Wizard
  Step 1: Goal (text + optional file upload)
  Step 2: Scope (module selection, constraints)
  Step 3: Priority & Timeline
  Step 4: Review & Submit
→ Task created → Planning Service triggered
→ User sees "Planning in progress..." status
```

### Flow 2: Review & Approve Plan
```
Notification received → Approvals page → Select plan
→ View DAG visualization
→ Review cost estimate
→ Read risk assessment
→ (Optional) Add comments on specific steps
→ Approve / Modify / Reject
→ If Approved: Orchestrator triggered
→ If Modified: Back to Planning Service (iterate)
→ If Rejected: Feedback to submitter
```

### Flow 3: Monitor Execution
```
Executions page → Select active execution
→ View step-by-step progress (real-time)
→ See agent logs streaming
→ Monitor token/cost usage
→ (If error) View error details + suggested fix
→ (If error) Retry / Skip / Abort
→ Execution complete → Review output
→ Provide feedback (accept/edit/reject)
```

### Flow 4: Manage Prompt Repository
```
Repository page → Navigate file tree
→ Select file → View content (Markdown rendered)
→ View version history
→ (Team Lead) Edit file → Preview changes
→ (Team Lead) Commit with message → Version bump
→ Changelog entry auto-generated
```

## Technical Requirements

| Requirement | Specification |
|-------------|--------------|
| Framework | Next.js 14 (App Router) |
| Real-time | Socket.IO for live updates |
| State | Zustand + TanStack Query |
| DAG Rendering | ReactFlow |
| Charts | Recharts |
| Editor | CodeMirror 6 (Markdown mode) |
| Auth | OAuth 2.0 (Keycloak) |
| Responsive | Desktop-first, tablet-friendly |
| Theme | Dark mode default, light available |
| Performance | LCP < 2.5s, FID < 100ms |

## Dependencies

| Depends On | Reason |
|-----------|--------|
| Planning Service API | Receives plan data |
| Approval Gateway API | Sends approval decisions |
| Orchestrator API | Receives execution status |
| Knowledge Base API | Searches repository |
| Feedback Engine API | Sends feedback data |
| Auth Service | User authentication |

## Success Metrics

| Metric | Target |
|--------|--------|
| Task submission time | < 2 minutes |
| Plan review time | < 5 minutes |
| Approval response time | < 1 hour (SLA) |
| Page load time | < 2 seconds |
| User satisfaction (NPS) | > 8/10 |
| Daily active users | 100% of team |
