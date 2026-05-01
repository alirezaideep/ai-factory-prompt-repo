# Command UI — Frontend Pages

## Page Map

| Route | Page | Layout | Auth Required |
|-------|------|--------|--------------|
| `/` | Dashboard | DashboardLayout | Yes |
| `/tasks` | Task List | DashboardLayout | Yes |
| `/tasks/new` | Task Wizard | DashboardLayout | Yes |
| `/tasks/:id` | Task Detail | DashboardLayout | Yes |
| `/plans/:id` | Plan Viewer | DashboardLayout | Yes |
| `/approvals` | Approval Queue | DashboardLayout | Yes (Lead+) |
| `/approvals/:id` | Approval Detail | DashboardLayout | Yes (Lead+) |
| `/executions` | Execution List | DashboardLayout | Yes |
| `/executions/:id` | Execution Detail | DashboardLayout | Yes |
| `/repo` | Repository Browser | DashboardLayout | Yes |
| `/repo/:path` | File Viewer/Editor | DashboardLayout | Yes |
| `/feedback` | Feedback Dashboard | DashboardLayout | Yes |
| `/settings` | Settings | DashboardLayout | Yes (Admin) |
| `/login` | Login | AuthLayout | No |

---

## Page Specifications

### Dashboard (`/`)

**Purpose:** Overview of platform activity at a glance.

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ [Sidebar]  │  Header: "Dashboard" + Search + Profile │
│            ├─────────────────────────────────────────┤
│ Dashboard  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐      │
│ Tasks      │  │Tasks│ │Pend.│ │Run. │ │Done │      │
│ Approvals  │  │ 12  │ │  3  │ │  5  │ │  8  │      │
│ Executions │  └─────┘ └─────┘ └─────┘ └─────┘      │
│ Repository │  ┌────────────────┐ ┌────────────────┐  │
│ Feedback   │  │ Recent Tasks   │ │ Active Agents  │  │
│ Settings   │  │ (table)        │ │ (status cards) │  │
│            │  └────────────────┘ └────────────────┘  │
│            │  ┌────────────────────────────────────┐  │
│            │  │ Cost & Performance Chart (7 days)  │  │
│            │  └────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**Data Sources:**
- `GET /api/v1/dashboard/summary`
- WebSocket: `task.status_changed`, `execution.completed`

**Key Components:**
- StatCard (4x top row)
- RecentTasksTable
- ActiveAgentsGrid
- CostChart (Recharts area chart)

---

### Task Wizard (`/tasks/new`)

**Purpose:** Multi-step form for creating a new task.

**Steps:**
1. **Goal** — Title, description, goal statement, file upload
2. **Scope** — Module selection (checkboxes), constraints (text areas)
3. **Priority** — Priority level, deadline (optional), dependencies
4. **Review** — Summary of all inputs, submit button

**Validation:**
- Step 1: Title required (min 10 chars), goal required (min 20 chars)
- Step 2: At least one module selected
- Step 3: Priority required
- Step 4: Confirmation checkbox

**UX Notes:**
- Auto-save draft every 30 seconds
- Back/Next navigation with progress indicator
- File upload supports drag-and-drop
- Module selection shows dependency warnings

---

### Plan Viewer (`/plans/:id`)

**Purpose:** Interactive visualization of generated plan.

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ Header: Plan #xyz | Status: Awaiting Approval       │
├─────────────────────────────────────────────────────┤
│ [Tabs: DAG | Timeline | Cost | Risk]                │
├────────────────────────────────┬────────────────────┤
│                                │ Detail Panel       │
│   DAG Visualization            │ (selected node)    │
│   (ReactFlow canvas)           │ - Agent assigned   │
│                                │ - Estimated time   │
│                                │ - Dependencies     │
│                                │ - Description      │
│                                │ - [Comment] button │
├────────────────────────────────┴────────────────────┤
│ [Approve] [Modify] [Reject]    Cost: $1.35 | 15min │
└─────────────────────────────────────────────────────┘
```

**Interactions:**
- Click node → show detail panel
- Hover edge → show dependency type
- Zoom/pan on DAG canvas
- Tab switching shows different views of same plan
- Approve/Modify/Reject buttons (visible to Lead+ only)

---

### Execution Detail (`/executions/:id`)

**Purpose:** Real-time monitoring of agent execution.

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ Header: Execution #123 | Status: Running ●          │
├─────────────────────────────────────────────────────┤
│ Progress: ████████░░░░ 5/8 steps (62%)              │
├────────────────────────────────┬────────────────────┤
│ Step List (vertical timeline)  │ Live Log Viewer    │
│ ✅ Step 1: Parse requirements  │ (streaming text)   │
│ ✅ Step 2: Generate schema     │                    │
│ ✅ Step 3: Create migration    │ > Generating code  │
│ ✅ Step 4: Implement API       │ > for order model  │
│ ✅ Step 5: Write tests         │ > Adding priority  │
│ 🔄 Step 6: Code review        │ > field...         │
│ ⏳ Step 7: Integration test    │                    │
│ ⏳ Step 8: Documentation       │                    │
├────────────────────────────────┴────────────────────┤
│ Cost: $0.96 | Tokens: 32K | Time: 4m 23s           │
│ [Pause] [Retry Step] [Abort]                        │
└─────────────────────────────────────────────────────┘
```

**Real-time Updates:**
- WebSocket events update step status
- Log viewer auto-scrolls (with pause on scroll-up)
- Cost counter updates every 5 seconds
- Progress bar animates on step completion

---

### Repository Browser (`/repo`)

**Purpose:** Browse, view, and edit prompt repository files.

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ Header: Repository | Version: v1.5.0 | [Search]     │
├──────────────┬──────────────────────────────────────┤
│ File Tree    │ File Content (Markdown rendered)      │
│              │                                      │
│ ▼ shared/    │ # Backend Architecture               │
│   backend.md │                                      │
│   frontend.md│ ## Overview                          │
│   design.md  │ The AI Software Factory backend...   │
│ ▼ features/  │                                      │
│   ▼ orders/  │ ## Tech Stack                        │
│     brief.md │ | Layer | Technology |                │
│     ▼ back/  │ ...                                  │
│       api.md │                                      │
│              ├──────────────────────────────────────┤
│              │ [Edit] [History] [Diff] | v1.5.0     │
└──────────────┴──────────────────────────────────────┘
```

**Features:**
- File tree with expand/collapse
- Search across all files (full-text)
- Markdown rendering with syntax highlighting
- Edit mode (CodeMirror) for team leads
- Version history sidebar
- Diff viewer between any two versions
- Commit interface with message and version bump selector
