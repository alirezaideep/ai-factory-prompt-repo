# Command UI — UI Wireframes & Design Specifications

## Global Layout

### Desktop (1280px+)
```
┌──────────────────────────────────────────────────────────────┐
│ ┌────────┐ ┌──────────────────────────────────────────────┐  │
│ │        │ │ Header: Breadcrumb | Search | Notifications │  │
│ │ Side   │ ├──────────────────────────────────────────────┤  │
│ │ bar    │ │                                              │  │
│ │        │ │              Main Content Area               │  │
│ │ 240px  │ │                                              │  │
│ │        │ │              (page-specific)                 │  │
│ │        │ │                                              │  │
│ │        │ │                                              │  │
│ └────────┘ └──────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Sidebar Navigation Items
```
🏠 Dashboard
📋 Tasks
✅ Approvals (badge: pending count)
⚡ Executions (badge: running count)
📁 Repository
📊 Feedback
⚙️ Settings
───────────
👤 User Profile
🔔 Notifications
```

## Color Usage by Context

| Context | Background | Text | Accent |
|---------|-----------|------|--------|
| Default page | `--background` | `--foreground` | — |
| Active sidebar item | `--muted` | `--foreground` | Left border blue |
| Status: success | — | `#10B981` | Green dot |
| Status: running | — | `#3B82F6` | Blue pulse |
| Status: failed | — | `#EF4444` | Red dot |
| Status: pending | — | `#F59E0B` | Amber dot |
| Approval card | `--card` | — | Left border amber |
| Code block | `#1E1E2E` | `#F8FAFC` | — |

## Key Interaction Patterns

### Task Wizard Steps
```
Step 1          Step 2          Step 3          Step 4
[●]─────────────[○]─────────────[○]─────────────[○]
 Goal            Scope           Priority        Review

┌────────────────────────────────────────────┐
│ Step 1: Define Your Goal                    │
│                                            │
│ Title: [________________________]          │
│                                            │
│ Description:                               │
│ ┌────────────────────────────────────┐     │
│ │                                    │     │
│ │ (rich text editor)                 │     │
│ │                                    │     │
│ └────────────────────────────────────┘     │
│                                            │
│ Goal Statement:                            │
│ [________________________________________] │
│                                            │
│ Attachments: [Drop files here] or [Browse] │
│                                            │
│                    [Cancel]  [Next →]       │
└────────────────────────────────────────────┘
```

### DAG Node Design
```
┌─────────────────────────┐
│ 🤖 code_agent           │  ← Agent icon + name (colored by agent)
│ ─────────────────────── │
│ Implement order model   │  ← Task description
│                         │
│ ⏱ ~3min  💰 $0.45      │  ← Estimates
│ Status: ✅ Complete     │  ← Status with color
└─────────────────────────┘
```

### Approval Card
```
┌─────────────────────────────────────────────┐
│ ⚠️ Plan Awaiting Approval                    │
│                                             │
│ Task: "Add urgent order support"            │
│ Submitted by: Developer A                   │
│ Steps: 8 | Cost: $1.35 | Risk: Medium       │
│ Time: 15 minutes ago                        │
│                                             │
│ [View Plan]  [Quick Approve]  [Reject]      │
└─────────────────────────────────────────────┘
```

### Log Viewer
```
┌─────────────────────────────────────────────┐
│ 📋 Logs — Step 6: Code Review               │
│ ─────────────────────────────────────────── │
│ [10:35:01] Starting code review...          │
│ [10:35:02] Loading file: order_model.py     │
│ [10:35:03] Analyzing 45 lines of code...    │
│ [10:35:05] Found 2 style issues             │
│ [10:35:06] Suggesting fix for line 23...    │
│ [10:35:08] ✅ Review complete               │
│ ▊                                           │  ← cursor (streaming)
│                                             │
│ [Auto-scroll: ON] [Copy] [Download]         │
└─────────────────────────────────────────────┘
```

## Responsive Breakpoints

| Breakpoint | Sidebar | Layout | Notes |
|-----------|---------|--------|-------|
| < 768px | Hidden (hamburger) | Single column | Mobile |
| 768-1024px | Icons only (64px) | Adapted | Tablet |
| 1024-1280px | Full (240px) | Standard | Laptop |
| > 1280px | Full + detail panel | Split view | Desktop |

## Animation & Transitions

| Element | Animation | Duration | Trigger |
|---------|-----------|----------|---------|
| Sidebar collapse | Width slide | 200ms | Toggle button |
| Modal open | Fade + scale up | 150ms | Action click |
| Toast notification | Slide in from right | 200ms | Event |
| Step completion | Check mark + green flash | 300ms | WebSocket event |
| Log entry | Fade in | 100ms | New log line |
| DAG node hover | Scale 1.02 + shadow | 150ms | Mouse hover |
