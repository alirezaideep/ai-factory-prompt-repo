# State Management

## State Categories

| Category | Tool | Examples |
|----------|------|----------|
| Server state | TanStack Query | Executions, plans, approvals |
| Client UI state | Zustand | Sidebar open, theme, filters |
| Form state | React Hook Form | Input values, validation |
| URL state | React Router | Current page, query params |
| Real-time state | WebSocket + Zustand | Live execution progress |

## Server State (TanStack Query)

### Query Pattern
```typescript
// hooks/useExecution.ts
export function useExecution(id: string) {
  return useQuery({
    queryKey: ['execution', id],
    queryFn: () => api.executions.getById(id),
    staleTime: 5000,  // Consider fresh for 5s
    refetchInterval: (data) => data?.status === 'running' ? 3000 : false,
  });
}

export function useExecutions(filters: ExecutionFilters) {
  return useQuery({
    queryKey: ['executions', filters],
    queryFn: () => api.executions.list(filters),
    placeholderData: keepPreviousData,  // Keep old data while fetching new
  });
}
```

### Mutation Pattern (Optimistic Updates)
```typescript
export function useRetryStep(executionId: string) {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (stepId: string) => api.executions.retryStep(executionId, stepId),
    
    // Optimistic update
    onMutate: async (stepId) => {
      await queryClient.cancelQueries({ queryKey: ['execution', executionId] });
      const previous = queryClient.getQueryData(['execution', executionId]);
      
      queryClient.setQueryData(['execution', executionId], (old: Execution) => ({
        ...old,
        steps: old.steps.map(s => 
          s.id === stepId ? { ...s, status: 'retrying' } : s
        )
      }));
      
      return { previous };
    },
    
    onError: (err, stepId, context) => {
      queryClient.setQueryData(['execution', executionId], context?.previous);
      toast.error('Retry failed');
    },
    
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['execution', executionId] });
    }
  });
}
```

## Client State (Zustand)

### Store Pattern
```typescript
// stores/ui.store.ts
interface UIStore {
  sidebarOpen: boolean;
  theme: 'light' | 'dark' | 'system';
  activeProjectId: string | null;
  
  // Actions
  toggleSidebar: () => void;
  setTheme: (theme: UIStore['theme']) => void;
  setActiveProject: (id: string | null) => void;
}

export const useUIStore = create<UIStore>()(
  persist(
    (set) => ({
      sidebarOpen: true,
      theme: 'system',
      activeProjectId: null,
      
      toggleSidebar: () => set(s => ({ sidebarOpen: !s.sidebarOpen })),
      setTheme: (theme) => set({ theme }),
      setActiveProject: (id) => set({ activeProjectId: id }),
    }),
    { name: 'ui-store' }  // localStorage persistence
  )
);
```

### Real-time Store
```typescript
// stores/execution-live.store.ts
interface ExecutionLiveStore {
  activeExecutions: Map<string, LiveExecutionState>;
  
  // WebSocket updates
  updateStepProgress: (executionId: string, stepId: string, progress: number) => void;
  updateStepStatus: (executionId: string, stepId: string, status: StepStatus) => void;
  addLogEntry: (executionId: string, entry: LogEntry) => void;
}

export const useExecutionLiveStore = create<ExecutionLiveStore>()((set) => ({
  activeExecutions: new Map(),
  
  updateStepProgress: (executionId, stepId, progress) => set(state => {
    const execution = state.activeExecutions.get(executionId);
    if (execution) {
      const step = execution.steps.find(s => s.id === stepId);
      if (step) step.progress = progress;
    }
    return { activeExecutions: new Map(state.activeExecutions) };
  }),
  
  // ... other actions
}));
```

## URL State
```typescript
// Use URL for shareable/bookmarkable state
function useExecutionFilters() {
  const [searchParams, setSearchParams] = useSearchParams();
  
  const filters: ExecutionFilters = {
    status: searchParams.get('status') as ExecutionStatus || undefined,
    projectId: searchParams.get('project') || undefined,
    page: parseInt(searchParams.get('page') || '1'),
  };
  
  const setFilter = (key: string, value: string | null) => {
    setSearchParams(prev => {
      if (value) prev.set(key, value);
      else prev.delete(key);
      prev.set('page', '1');  // Reset page on filter change
      return prev;
    });
  };
  
  return { filters, setFilter };
}
```

## State Synchronization Rules
1. Server state is the source of truth — never store server data in Zustand
2. Optimistic updates must always have rollback logic
3. WebSocket updates merge into TanStack Query cache (not separate store)
4. URL state for anything that should survive page refresh
5. localStorage only for user preferences (theme, sidebar, locale)
