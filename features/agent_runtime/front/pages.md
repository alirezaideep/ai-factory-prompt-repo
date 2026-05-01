# Agent Runtime — Frontend Pages

## Page: Agent Registry (`/agents`)

### Layout
Dashboard-style page showing all registered agent types in a grid/table view.

### Components
- **AgentCard** — Shows agent name, type, model, status indicator, success rate sparkline
- **AgentTable** — Tabular view with sortable columns (name, type, success rate, avg cost, executions)
- **AgentFilters** — Filter by type, status, model
- **RegisterAgentDialog** — Modal form for registering new agent types

### Data Flow
```
trpc.agents.list.useQuery({ filters })
trpc.agents.register.useMutation()
trpc.agents.updateConfig.useMutation()
```

### States
- Loading: Skeleton cards
- Empty: "No agents registered. Create your first agent."
- Error: Toast with retry button

---

## Page: Agent Detail (`/agents/:agentId`)

### Layout
Split view — left panel shows config, right panel shows live metrics and history.

### Sections
1. **Header** — Agent name, type badge, status toggle (active/paused)
2. **Configuration** — Editable system prompt, tool selection, budget settings
3. **Metrics Dashboard** — Success rate chart, cost over time, avg duration
4. **Execution History** — Filterable table of past executions with status, duration, tokens
5. **Error Analysis** — Common failure patterns, self-fix success rate

### Real-time Updates
- WebSocket subscription to agent execution events
- Live counter for active tasks
- Streaming log output for running tasks

---

## Page: Agent Playground (`/agents/:agentId/playground`)

### Purpose
Test an agent with custom input before using in production executions.

### Layout
- Left: Context editor (select files from repo, write instruction)
- Right: Output panel (streaming response, tool calls, final result)
- Bottom: Cost estimate, token usage breakdown

### Features
- Save test cases for regression testing
- Compare outputs between different agent versions
- Export successful runs as execution templates
