# Orchestrator — Execution Flow Workflow

## LangGraph Graph Definition

```python
from langgraph.graph import StateGraph, END

graph = StateGraph(ExecutionState)

# Nodes
graph.add_node("initialize", initialize_execution)
graph.add_node("find_ready_steps", find_ready_steps)
graph.add_node("dispatch_agents", dispatch_agents)
graph.add_node("wait_for_completion", wait_for_completion)
graph.add_node("process_results", process_results)
graph.add_node("check_completion", check_completion)
graph.add_node("handle_failure", handle_failure)
graph.add_node("finalize", finalize_execution)

# Edges
graph.set_entry_point("initialize")
graph.add_edge("initialize", "find_ready_steps")
graph.add_edge("find_ready_steps", "dispatch_agents")
graph.add_edge("dispatch_agents", "wait_for_completion")
graph.add_edge("wait_for_completion", "process_results")

graph.add_conditional_edges("process_results", route_after_results, {
    "continue": "check_completion",
    "failure": "handle_failure"
})

graph.add_conditional_edges("check_completion", route_after_check, {
    "more_steps": "find_ready_steps",
    "all_done": "finalize",
    "paused": END,
    "cancelled": "finalize"
})

graph.add_conditional_edges("handle_failure", route_after_failure, {
    "retry": "dispatch_agents",
    "skip": "check_completion",
    "escalate": END,
    "abort": "finalize"
})

graph.add_edge("finalize", END)
```

## Node Implementations

### initialize
```python
async def initialize_execution(state: ExecutionState) -> ExecutionState:
    """
    1. Parse DAG into nodes and edges
    2. Set all steps to 'pending'
    3. Validate DAG integrity
    4. Acquire execution lock
    5. Record start time
    """
    state["pending_steps"] = [n["step_id"] for n in state["dag_nodes"]]
    state["status"] = "running"
    state["started_at"] = now_iso()
    return state
```

### find_ready_steps
```python
async def find_ready_steps(state: ExecutionState) -> ExecutionState:
    """
    Find steps whose dependencies are ALL in completed_steps.
    Respect max_parallel limit.
    """
    ready = []
    for node in state["dag_nodes"]:
        step_id = node["step_id"]
        if step_id in state["pending_steps"]:
            deps = node.get("depends_on", [])
            if all(d in state["completed_steps"] for d in deps):
                ready.append(step_id)
    
    # Limit to max_parallel
    max_parallel = state.get("config", {}).get("max_parallel", 3)
    available_slots = max_parallel - len(state["running_steps"])
    ready = ready[:available_slots]
    
    # Move from pending to running
    for step_id in ready:
        state["pending_steps"].remove(step_id)
        state["running_steps"].append(step_id)
    
    return state
```

### dispatch_agents
```python
async def dispatch_agents(state: ExecutionState) -> ExecutionState:
    """
    For each running step:
    1. Determine agent type
    2. Assemble context from Prompt Factory
    3. Include outputs from dependency steps
    4. Call Agent Runtime
    """
    for step_id in state["running_steps"]:
        node = get_node(state, step_id)
        
        # Get context from Prompt Factory
        context = await prompt_factory.assemble_context(
            step_id=step_id,
            plan_id=state["plan_id"],
            max_tokens=50000
        )
        
        # Include outputs from dependencies
        dep_outputs = {
            dep: state["step_outputs"].get(dep, {})
            for dep in node.get("depends_on", [])
        }
        
        # Dispatch to Agent Runtime (async)
        await agent_runtime.execute(
            execution_id=state["execution_id"],
            step_id=step_id,
            agent_type=node["agent_type"],
            context=context,
            dependency_outputs=dep_outputs,
            config=node.get("config", {})
        )
    
    return state
```

### wait_for_completion
```python
async def wait_for_completion(state: ExecutionState) -> ExecutionState:
    """
    Wait for all running steps to complete (or timeout).
    Uses async polling with configurable interval.
    Checks for pause/cancel requests.
    """
    while state["running_steps"]:
        if state["pause_requested"]:
            state["status"] = "paused"
            return state
        
        if state["cancel_requested"]:
            state["status"] = "cancelled"
            return state
        
        for step_id in list(state["running_steps"]):
            result = await agent_runtime.check_status(state["execution_id"], step_id)
            
            if result.status == "completed":
                state["running_steps"].remove(step_id)
                state["completed_steps"].append(step_id)
                state["step_outputs"][step_id] = result.output
                state["tokens_used"] += result.tokens_used
                state["cost_usd"] += result.cost_usd
            
            elif result.status == "failed":
                state["running_steps"].remove(step_id)
                state["failed_steps"].append(step_id)
                state["errors"].append({
                    "step_id": step_id,
                    "error_type": result.error_type,
                    "message": result.error_message,
                    "retry_count": result.retry_count
                })
        
        # Checkpoint state
        await save_checkpoint(state)
        await asyncio.sleep(2)  # Poll interval
    
    return state
```

### check_completion
```python
async def check_completion(state: ExecutionState) -> str:
    """Route based on execution state."""
    if state["cancel_requested"]:
        return "cancelled"
    
    if state["pause_requested"]:
        return "paused"
    
    if not state["pending_steps"] and not state["running_steps"]:
        return "all_done"
    
    if state["pending_steps"]:
        return "more_steps"
    
    return "all_done"
```

### handle_failure
```python
async def handle_failure(state: ExecutionState) -> str:
    """Determine recovery strategy for failed steps."""
    for error in state["errors"]:
        step_id = error["step_id"]
        retry_count = error["retry_count"]
        
        node = get_node(state, step_id)
        max_retries = node.get("max_retries", 3)
        
        if retry_count < max_retries:
            # Move back to running for retry
            state["failed_steps"].remove(step_id)
            state["running_steps"].append(step_id)
            return "retry"
        
        if node.get("optional", False):
            # Skip optional steps
            state["failed_steps"].remove(step_id)
            state["skipped_steps"].append(step_id)
            return "skip"
        
        # Check budget for escalation
        if state["cost_usd"] > state["token_budget"] * 0.9:
            return "abort"
        
        # Escalate to human
        await notify_team_lead(state["execution_id"], step_id, error)
        return "escalate"
    
    return "abort"
```

## Parallel Execution Example

```
Plan DAG:
  step_1 (schema migration) → step_2 (backend API) → step_4 (integration test)
                             → step_3 (frontend UI) → step_4
  step_5 (docs) [independent]

Execution Timeline:
  T0: dispatch step_1, step_5 (parallel, no deps)
  T1: step_1 completes → dispatch step_2, step_3 (parallel)
      step_5 completes
  T2: step_2 completes, step_3 completes → dispatch step_4
  T3: step_4 completes → DONE
```

## Budget Guard

```python
async def check_budget(state: ExecutionState) -> bool:
    """Check if execution is within budget."""
    budget = state.get("token_budget", float("inf"))
    
    if state["tokens_used"] > budget * 0.8:
        emit_event("budget_warning", {"used": state["tokens_used"], "budget": budget})
    
    if state["tokens_used"] > budget:
        state["status"] = "paused"
        emit_event("budget_exceeded", {"used": state["tokens_used"], "budget": budget})
        return False
    
    return True
```
