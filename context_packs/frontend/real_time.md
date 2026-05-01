# Real-Time Communication

## WebSocket Architecture

### Connection Management
```typescript
// lib/websocket.ts
class WebSocketManager {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private listeners = new Map<string, Set<(data: any) => void>>();

  connect(url: string, token: string) {
    this.ws = new WebSocket(`${url}?token=${token}`);
    
    this.ws.onopen = () => {
      this.reconnectAttempts = 0;
      this.resubscribeAll();
    };
    
    this.ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      this.dispatch(msg.type, msg.data);
    };
    
    this.ws.onclose = (event) => {
      if (!event.wasClean) this.reconnect(url, token);
    };
  }

  subscribe(channel: string, callback: (data: any) => void) {
    if (!this.listeners.has(channel)) {
      this.listeners.set(channel, new Set());
    }
    this.listeners.get(channel)!.add(callback);
    
    // Tell server to subscribe
    this.send({ type: 'subscribe', channel });
    
    return () => {
      this.listeners.get(channel)?.delete(callback);
      if (this.listeners.get(channel)?.size === 0) {
        this.send({ type: 'unsubscribe', channel });
      }
    };
  }

  private reconnect(url: string, token: string) {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) return;
    
    const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
    this.reconnectAttempts++;
    
    setTimeout(() => this.connect(url, token), delay);
  }
}

export const wsManager = new WebSocketManager();
```

### React Hook
```typescript
// hooks/useWebSocket.ts
export function useWebSocket(channel: string, onMessage: (data: any) => void) {
  const callbackRef = useRef(onMessage);
  callbackRef.current = onMessage;

  useEffect(() => {
    const unsubscribe = wsManager.subscribe(channel, (data) => {
      callbackRef.current(data);
    });
    return unsubscribe;
  }, [channel]);
}

// Usage: Live execution updates
function ExecutionLiveView({ executionId }: { executionId: string }) {
  const queryClient = useQueryClient();
  
  useWebSocket(`execution:${executionId}`, (msg) => {
    switch (msg.type) {
      case 'step_progress':
        // Update cache directly for smooth UI
        queryClient.setQueryData(['execution', executionId], (old: Execution) => ({
          ...old,
          steps: old.steps.map(s => 
            s.id === msg.stepId ? { ...s, progress: msg.percent } : s
          )
        }));
        break;
        
      case 'step_completed':
        queryClient.invalidateQueries({ queryKey: ['execution', executionId] });
        break;
        
      case 'approval_needed':
        toast.info('Approval required', { action: { label: 'Review', onClick: () => navigate(`/approvals/${msg.approvalId}`) } });
        break;
    }
  });
}
```

## Server-Sent Events (SSE)

### For Log Streaming
```typescript
// hooks/useLogStream.ts
export function useLogStream(executionId: string) {
  const [logs, setLogs] = useState<LogEntry[]>([]);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    const eventSource = new EventSource(`/api/v1/orchestrator/executions/${executionId}/logs`);
    
    eventSource.onopen = () => setConnected(true);
    
    eventSource.addEventListener('step_started', (e) => {
      setLogs(prev => [...prev, { type: 'step_started', ...JSON.parse(e.data) }]);
    });
    
    eventSource.addEventListener('tool_call', (e) => {
      setLogs(prev => [...prev, { type: 'tool_call', ...JSON.parse(e.data) }]);
    });
    
    eventSource.addEventListener('step_completed', (e) => {
      setLogs(prev => [...prev, { type: 'step_completed', ...JSON.parse(e.data) }]);
    });
    
    eventSource.onerror = () => {
      setConnected(false);
      eventSource.close();
    };
    
    return () => eventSource.close();
  }, [executionId]);

  return { logs, connected };
}
```

## Channels & Subscriptions

| Channel Pattern | Data | Subscribers |
|----------------|------|-------------|
| `execution:{id}` | Step progress, status changes | Execution detail page |
| `project:{id}` | New executions, approvals | Project dashboard |
| `user:{id}` | Notifications, mentions | Global notification bell |
| `system` | Maintenance alerts, announcements | All connected clients |

## Offline Handling
```typescript
// Detect connectivity and queue actions
function useOfflineQueue() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  const queue = useRef<QueuedAction[]>([]);

  useEffect(() => {
    const handleOnline = () => {
      setIsOnline(true);
      // Flush queued actions
      queue.current.forEach(action => action.execute());
      queue.current = [];
    };
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', () => setIsOnline(false));
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', () => setIsOnline(false));
    };
  }, []);

  return { isOnline, enqueue: (action: QueuedAction) => queue.current.push(action) };
}
```
