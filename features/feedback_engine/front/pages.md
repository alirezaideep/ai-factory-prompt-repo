# Feedback Engine — Frontend Pages

## Page: Analytics Dashboard (`/analytics`)

### Layout
Full-width dashboard with KPI cards at top, charts below, and drill-down tables.

### Sections
1. **KPI Cards Row** — Overall success rate, avg quality, cost/task, human edit rate
2. **Trends Chart** — Line chart showing metrics over time (7d, 30d, 90d toggle)
3. **Agent Performance** — Bar chart comparing agent types
4. **Cost Breakdown** — Pie/donut chart by project and model
5. **Common Issues** — Table of recurring patterns with frequency

### Data Flow
```
trpc.feedback.analytics.agents.useQuery({ period })
trpc.feedback.analytics.cost.useQuery({ period, groupBy })
trpc.feedback.analytics.prompts.useQuery({ period })
```

---

## Page: Improvement Suggestions (`/analytics/suggestions`)

### Layout
Card list of AI-generated suggestions, sortable by confidence and impact.

### Components
- **SuggestionCard** — Shows description, evidence, estimated impact, confidence badge
- **ApplyDialog** — Confirmation dialog with diff preview before applying
- **EvidencePanel** — Expandable panel showing the signals that led to suggestion

### Actions
- Apply suggestion (triggers approval workflow)
- Dismiss suggestion (with reason)
- Save for later

---

## Page: Signal Explorer (`/analytics/signals`)

### Purpose
Raw signal browser for debugging and deep analysis.

### Features
- Filter by type, agent, project, date range
- View individual signal details
- Export to CSV for external analysis
- Compare signals across time periods
