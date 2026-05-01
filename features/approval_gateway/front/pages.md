# Approval Gateway — Frontend Pages

## Page: Approval Queue (`/approvals`)

### Layout
- Full-width table/list view
- Filter bar at top: status (pending/approved/rejected/all), risk level, type, project
- Sort options: newest first, highest risk first, expiring soon

### Components
- `ApprovalCard` — Compact card showing: title, type badge, risk badge, cost estimate, time remaining
- `ApprovalFilters` — Filter/sort controls
- `QuickActions` — Inline approve/reject buttons for low-risk items
- `BatchActions` — Select multiple and batch approve (only for auto-approvable items)

### Data Flow
```typescript
const { data: approvals, isLoading } = trpc.approvals.list.useQuery({
  status: filter.status,
  riskLevel: filter.riskLevel,
  type: filter.type,
  projectId: filter.projectId,
  page: currentPage,
  limit: 20
});
```

---

## Page: Approval Detail (`/approvals/:id`)

### Layout
- Two-column layout on desktop
- Left (60%): Payload visualization + diff viewer
- Right (40%): Decision panel + comment thread

### Sections

#### Header
- Title, type badge, risk badge
- Requested by (system/user), assigned to
- Time remaining (countdown), created at

#### Payload Viewer (Left Column)
- **For plan_approval:** Step-by-step plan visualization, DAG preview, cost breakdown
- **For merge_request:** Side-by-side diff viewer (before/after for each file)
- **For execution_start:** Agent config, files to modify, sandbox settings
- **For budget_override:** Spending chart, remaining budget, justification

#### Decision Panel (Right Column)
```typescript
// Decision form
interface DecisionForm {
  action: 'approve' | 'reject' | 'modify';
  reason?: string;  // Required for reject
  modifications?: {
    field: string;
    original_value: any;
    new_value: any;
  }[];
  conditions?: string[];  // Optional conditions for approval
}
```

#### Comment Thread
- Chronological list of comments
- Support for: plain text, code blocks, @mentions
- Type indicators: comment, question, suggestion, condition

### Actions
```typescript
const decideMutation = trpc.approvals.decide.useMutation({
  onSuccess: () => {
    toast.success('Decision recorded');
    router.push('/approvals');
  }
});

const commentMutation = trpc.approvals.comment.useMutation({
  onSuccess: () => {
    utils.approvals.getById.invalidate(approvalId);
  }
});
```

---

## Page: Approval Policies (`/settings/approval-policies`)

### Layout
- List of active policies with toggle switches
- Create/edit policy modal

### Policy Editor
```typescript
interface PolicyForm {
  name: string;
  description: string;
  triggerConditions: {
    type: string[];
    riskLevel?: string[];
    estimatedCostGt?: number;
    modules?: string[];
  };
  requiredApprovers: number;
  autoApproveConditions?: {
    riskLevel?: string[];
    estimatedCostLt?: number;
    moduleNotIn?: string[];
  };
  escalationTimeoutMinutes: number;
  escalationTo?: string;  // user ID
}
```

---

## Page: Approval History (`/approvals/history`)

### Layout
- Searchable table with columns: title, type, decision, decided_by, duration, cost
- Date range filter
- Export to CSV

### Analytics Cards (top)
- Average approval time (by risk level)
- Approval rate (approved vs rejected)
- Most common modification reasons
- Auto-approve percentage

---

## Shared Components

### `RiskBadge`
```typescript
const RISK_COLORS = {
  low: 'bg-green-100 text-green-800',
  medium: 'bg-yellow-100 text-yellow-800',
  high: 'bg-orange-100 text-orange-800',
  critical: 'bg-red-100 text-red-800'
};
```

### `CountdownTimer`
- Shows time remaining before escalation
- Changes color as deadline approaches
- Pulses when < 5 minutes remaining

### `DiffViewer`
- Side-by-side or unified diff view
- Syntax highlighting for code/markdown
- Collapsible sections for large diffs

### `CostBreakdown`
- Pie chart: tokens by agent type
- Bar chart: estimated vs actual (for historical)
- Total cost with confidence interval
