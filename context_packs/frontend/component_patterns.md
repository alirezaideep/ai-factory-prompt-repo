# Component Patterns

## Technology Stack
- **Framework:** React 19 + TypeScript 5.4
- **Styling:** Tailwind CSS 4 + shadcn/ui
- **State:** TanStack Query (server state) + Zustand (client state)
- **Routing:** React Router v7 (or TanStack Router)
- **Build:** Vite 6

## Component Architecture

### File Structure
```
src/
  components/
    ui/              ← shadcn/ui primitives (Button, Dialog, etc.)
    shared/          ← App-wide reusable (Header, Sidebar, StatusBadge)
    features/        ← Feature-specific composites
      execution/
        ExecutionDAG.tsx
        ExecutionStepCard.tsx
        ExecutionProgress.tsx
      planning/
        PlanEditor.tsx
        PlanStepList.tsx
  pages/
    Dashboard.tsx
    Executions.tsx
    ExecutionDetail.tsx
  hooks/
    useExecution.ts
    useWebSocket.ts
  lib/
    api.ts           ← API client (tRPC or fetch wrapper)
    utils.ts         ← Pure utility functions
```

### Component Template
```typescript
// Feature component template
import { useState, useMemo } from 'react';
import { Card, CardHeader, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';

interface ExecutionStepCardProps {
  step: ExecutionStep;
  isActive: boolean;
  onRetry?: (stepId: string) => void;
  onSkip?: (stepId: string) => void;
}

export function ExecutionStepCard({ step, isActive, onRetry, onSkip }: ExecutionStepCardProps) {
  // 1. Hooks at top
  const [expanded, setExpanded] = useState(false);
  const statusColor = useMemo(() => getStatusColor(step.status), [step.status]);

  // 2. Early returns for edge cases
  if (!step) return null;

  // 3. Render
  return (
    <Card className={cn('transition-all', isActive && 'ring-2 ring-primary')}>
      <CardHeader className="flex flex-row items-center justify-between">
        <span className="font-medium">{step.name}</span>
        <Badge variant={statusColor}>{step.status}</Badge>
      </CardHeader>
      {expanded && (
        <CardContent>
          {/* Step details */}
        </CardContent>
      )}
    </Card>
  );
}
```

## Composition Patterns

### Container/Presenter
```typescript
// Container: handles data fetching and state
function ExecutionPageContainer() {
  const { id } = useParams();
  const { data, isLoading, error } = useExecution(id);
  
  if (isLoading) return <ExecutionSkeleton />;
  if (error) return <ErrorDisplay error={error} />;
  if (!data) return <NotFound />;
  
  return <ExecutionPageView execution={data} />;
}

// Presenter: pure rendering, easily testable
function ExecutionPageView({ execution }: { execution: Execution }) {
  return (
    <div className="space-y-6">
      <ExecutionHeader execution={execution} />
      <ExecutionDAG steps={execution.steps} />
      <ExecutionLogs executionId={execution.id} />
    </div>
  );
}
```

### Compound Components
```typescript
// For complex UI with shared state
function StepList({ children }: { children: React.ReactNode }) {
  const [expandedId, setExpandedId] = useState<string | null>(null);
  
  return (
    <StepListContext.Provider value={{ expandedId, setExpandedId }}>
      <div className="space-y-2">{children}</div>
    </StepListContext.Provider>
  );
}

StepList.Item = function StepListItem({ step }: { step: Step }) {
  const { expandedId, setExpandedId } = useStepListContext();
  const isExpanded = expandedId === step.id;
  // ...
};

// Usage
<StepList>
  {steps.map(step => <StepList.Item key={step.id} step={step} />)}
</StepList>
```

## Performance Patterns

### Memoization
```typescript
// Memoize expensive computations
const dagLayout = useMemo(() => calculateDAGLayout(steps, edges), [steps, edges]);

// Memoize callbacks passed to children
const handleRetry = useCallback((stepId: string) => {
  retryMutation.mutate({ executionId, stepId });
}, [executionId]);

// Memoize components for list rendering
const MemoizedStepCard = React.memo(ExecutionStepCard);
```

### Lazy Loading
```typescript
// Route-level code splitting
const ExecutionDetail = lazy(() => import('./pages/ExecutionDetail'));
const PlanEditor = lazy(() => import('./pages/PlanEditor'));

// Component-level lazy loading
const DAGVisualization = lazy(() => import('./components/features/execution/ExecutionDAG'));
```

### Virtual Lists
```typescript
// For long lists (logs, history)
import { useVirtualizer } from '@tanstack/react-virtual';

function LogViewer({ logs }: { logs: LogEntry[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: logs.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40,
    overscan: 20,
  });
  // ...
}
```

## Naming Conventions
| Type | Convention | Example |
|------|-----------|---------|
| Component | PascalCase | `ExecutionStepCard` |
| Hook | camelCase with `use` prefix | `useExecution` |
| Utility | camelCase | `formatDuration` |
| Constant | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Type/Interface | PascalCase | `ExecutionStep` |
| File (component) | PascalCase.tsx | `ExecutionStepCard.tsx` |
| File (hook) | camelCase.ts | `useExecution.ts` |
| File (util) | camelCase.ts | `formatters.ts` |
| CSS class | kebab-case (Tailwind) | `text-primary` |
