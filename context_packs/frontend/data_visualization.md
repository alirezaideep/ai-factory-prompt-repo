# Data Visualization

## Libraries
- **DAG/Graph:** React Flow (for execution DAGs, dependency graphs)
- **Charts:** Recharts (for metrics, analytics)
- **Timeline:** Custom (for execution history, Gantt-like views)

## DAG Visualization (Execution Steps)

### React Flow Setup
```typescript
import ReactFlow, { 
  Node, Edge, Background, Controls, MiniMap,
  useNodesState, useEdgesState 
} from 'reactflow';

interface ExecutionDAGProps {
  steps: ExecutionStep[];
  dependencies: { from: string; to: string }[];
  onStepClick: (stepId: string) => void;
}

export function ExecutionDAG({ steps, dependencies, onStepClick }: ExecutionDAGProps) {
  const [nodes, setNodes, onNodesChange] = useNodesState(
    steps.map(step => ({
      id: step.id,
      type: 'executionStep',
      position: calculatePosition(step, steps, dependencies),
      data: { step, onClick: onStepClick }
    }))
  );
  
  const edges: Edge[] = dependencies.map(dep => ({
    id: `${dep.from}-${dep.to}`,
    source: dep.from,
    target: dep.to,
    animated: isStepRunning(dep.from, steps),
    style: { stroke: getEdgeColor(dep, steps) }
  }));

  return (
    <div className="h-[500px] w-full border rounded-lg">
      <ReactFlow
        nodes={nodes}
        edges={edges}
        nodeTypes={{ executionStep: ExecutionStepNode }}
        onNodesChange={onNodesChange}
        fitView
      >
        <Background />
        <Controls />
        <MiniMap />
      </ReactFlow>
    </div>
  );
}
```

### Custom Node
```typescript
function ExecutionStepNode({ data }: { data: { step: ExecutionStep } }) {
  const { step } = data;
  
  return (
    <div className={cn(
      'px-4 py-3 rounded-lg border-2 min-w-[180px]',
      STATUS_STYLES[step.status]
    )}>
      <div className="flex items-center gap-2">
        <StatusIcon status={step.status} />
        <span className="font-medium text-sm">{step.name}</span>
      </div>
      {step.status === 'running' && (
        <Progress value={step.progress} className="mt-2 h-1.5" />
      )}
      <div className="flex justify-between mt-1 text-xs text-muted-foreground">
        <span>{step.agentType}</span>
        {step.duration && <span>{formatDuration(step.duration)}</span>}
      </div>
    </div>
  );
}

const STATUS_STYLES = {
  pending: 'border-gray-300 bg-gray-50',
  running: 'border-blue-400 bg-blue-50 animate-pulse',
  completed: 'border-green-400 bg-green-50',
  failed: 'border-red-400 bg-red-50',
  skipped: 'border-gray-200 bg-gray-100 opacity-60',
  retrying: 'border-yellow-400 bg-yellow-50',
};
```

## Metrics Charts

### Cost Over Time
```typescript
import { LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer } from 'recharts';

function CostChart({ data }: { data: DailyCost[] }) {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <LineChart data={data}>
        <XAxis dataKey="date" tickFormatter={formatDate} />
        <YAxis tickFormatter={(v) => `$${v}`} />
        <Tooltip 
          formatter={(value: number) => [`$${value.toFixed(2)}`, 'Cost']}
          labelFormatter={formatDate}
        />
        <Line 
          type="monotone" 
          dataKey="cost" 
          stroke="hsl(var(--primary))" 
          strokeWidth={2}
          dot={false}
        />
        <Line 
          type="monotone" 
          dataKey="budget" 
          stroke="hsl(var(--destructive))" 
          strokeDasharray="5 5"
          dot={false}
        />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

### Success Rate Gauge
```typescript
function SuccessGauge({ rate }: { rate: number }) {
  const color = rate >= 0.9 ? 'text-green-500' : rate >= 0.7 ? 'text-yellow-500' : 'text-red-500';
  
  return (
    <div className="relative w-32 h-32">
      <svg viewBox="0 0 100 100" className="transform -rotate-90">
        <circle cx="50" cy="50" r="40" fill="none" stroke="currentColor" 
          className="text-muted" strokeWidth="8" />
        <circle cx="50" cy="50" r="40" fill="none" stroke="currentColor"
          className={color} strokeWidth="8"
          strokeDasharray={`${rate * 251.2} 251.2`}
          strokeLinecap="round" />
      </svg>
      <span className="absolute inset-0 flex items-center justify-center text-2xl font-bold">
        {Math.round(rate * 100)}%
      </span>
    </div>
  );
}
```

## Execution Timeline
```typescript
function ExecutionTimeline({ steps }: { steps: CompletedStep[] }) {
  const totalDuration = steps[steps.length - 1].endTime - steps[0].startTime;
  
  return (
    <div className="relative h-16 bg-muted rounded-lg overflow-hidden">
      {steps.map(step => {
        const left = ((step.startTime - steps[0].startTime) / totalDuration) * 100;
        const width = ((step.endTime - step.startTime) / totalDuration) * 100;
        
        return (
          <Tooltip key={step.id} content={`${step.name}: ${formatDuration(step.duration)}`}>
            <div
              className={cn('absolute h-full', STATUS_BG[step.status])}
              style={{ left: `${left}%`, width: `${Math.max(width, 1)}%` }}
            />
          </Tooltip>
        );
      })}
    </div>
  );
}
```

## Color Conventions
| Metric | Color | Tailwind |
|--------|-------|----------|
| Success/Healthy | Green | `text-green-500` |
| Running/Active | Blue | `text-blue-500` |
| Warning/Degraded | Yellow | `text-yellow-500` |
| Error/Failed | Red | `text-red-500` |
| Neutral/Pending | Gray | `text-gray-400` |
| Cost/Budget | Purple | `text-purple-500` |
