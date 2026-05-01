# Execution Patterns

## DAG Construction

### From Plan Schema to DAG
```python
def build_dag(plan: PlanSchema) -> ExecutionDAG:
    """
    Convert a plan schema into an executable DAG.
    Rules:
    1. Steps with no dependencies start immediately (root nodes)
    2. Steps with dependencies wait for all deps to complete
    3. Independent steps run in parallel (up to max_parallel)
    4. Failed steps block their dependents (unless skip policy)
    """
    dag = ExecutionDAG()
    
    for step in plan.steps:
        node = DAGNode(
            id=step.id,
            name=step.name,
            agent_type=step.agent_type,
            input_files=step.input_files,
            output_files=step.output_files,
            estimated_tokens=step.estimated_tokens,
            timeout_seconds=step.timeout_seconds or 300,
        )
        dag.add_node(node)
    
    for dep in plan.dependencies:
        dag.add_edge(dep.from_step, dep.to_step)
    
    # Validate: no cycles, all referenced steps exist
    dag.validate()
    
    return dag
```

### Execution Loop
```python
async def execute_dag(dag: ExecutionDAG, config: ExecutionConfig):
    """
    Main execution loop. Runs until all nodes complete or unrecoverable failure.
    """
    scheduler = DAGScheduler(dag, max_parallel=config.max_parallel)
    
    while not scheduler.is_complete():
        # Get next batch of ready nodes
        ready_nodes = scheduler.get_ready_nodes()
        
        if not ready_nodes and scheduler.has_running():
            # Wait for running nodes to complete
            await scheduler.wait_for_any_completion()
            continue
        
        if not ready_nodes and not scheduler.has_running():
            # Deadlock or all blocked
            raise ExecutionDeadlockError(scheduler.get_blocked_nodes())
        
        # Launch ready nodes (respecting max_parallel)
        for node in ready_nodes[:config.max_parallel - scheduler.running_count()]:
            # Assemble context for this node
            context = await assemble_context(node, dag, config)
            
            # Check budget
            if not budget_check(node, config):
                await request_budget_approval(node, config)
                continue
            
            # Launch agent
            task = asyncio.create_task(execute_node(node, context, config))
            scheduler.mark_running(node.id, task)
        
        # Brief pause to prevent tight loop
        await asyncio.sleep(0.1)
    
    return scheduler.get_results()
```

## Context Assembly

### Loading Formula
```python
async def assemble_context(node: DAGNode, dag: ExecutionDAG, config: ExecutionConfig) -> AgentContext:
    """
    Build the context payload for an agent based on the loading formula.
    Priority order (highest to lowest):
    1. Direct input files (from plan)
    2. Output from dependency nodes
    3. Relevant repo files (from Knowledge Base)
    4. Global conventions (from context_packs)
    """
    context = AgentContext(max_tokens=config.context_budget_per_step)
    
    # 1. Direct inputs (always included)
    for file_path in node.input_files:
        content = await repo.read_file(file_path)
        context.add(content, priority=Priority.CRITICAL, source=file_path)
    
    # 2. Outputs from completed dependencies
    for dep_id in dag.get_dependencies(node.id):
        dep_output = dag.get_node_output(dep_id)
        if dep_output:
            context.add(dep_output.summary, priority=Priority.HIGH, source=f"output:{dep_id}")
    
    # 3. Relevant repo files (semantic search)
    relevant = await knowledge_base.search(
        query=node.name + " " + node.description,
        filters={"project_id": config.project_id, "layer": node.agent_type},
        max_results=5
    )
    for chunk in relevant:
        context.add(chunk.text, priority=Priority.MEDIUM, source=chunk.path)
    
    # 4. Global conventions (if budget allows)
    conventions = get_conventions_for_agent(node.agent_type)
    for conv in conventions:
        context.add(conv, priority=Priority.LOW, source="conventions")
    
    # Trim to budget
    context.trim_to_budget()
    
    return context
```

## Parallel Execution Patterns

### Fan-Out / Fan-In
```
[Step A] → [Step B1] → [Step D]
         → [Step B2] →
         → [Step B3] →
```
- B1, B2, B3 run in parallel after A completes
- D waits for ALL of B1, B2, B3

### Pipeline
```
[Step A] → [Step B] → [Step C] → [Step D]
```
- Strictly sequential, each depends on previous

### Diamond
```
[Step A] → [Step B] → [Step D]
         → [Step C] →
```
- B and C run in parallel, D waits for both

## Output Handling

### Step Output Format
```typescript
interface StepOutput {
  stepId: string;
  status: 'completed' | 'failed';
  
  // What was produced
  files: {
    path: string;
    action: 'created' | 'modified' | 'deleted';
    content?: string;  // For merge review
    diff?: string;     // For modified files
  }[];
  
  // Summary for dependent steps
  summary: string;
  
  // Metrics
  tokensUsed: number;
  durationMs: number;
  model: string;
  
  // For review
  confidence: number;  // 0-1, agent self-assessment
  needsReview: boolean;
}
```

### Output Validation
```python
async def validate_step_output(output: StepOutput, node: DAGNode) -> ValidationResult:
    """
    Validate that step output meets expectations before allowing dependents to proceed.
    """
    issues = []
    
    # Check all expected output files were produced
    for expected in node.output_files:
        if expected not in [f.path for f in output.files]:
            issues.append(f"Missing expected output: {expected}")
    
    # Check file syntax (parse without executing)
    for file in output.files:
        if file.path.endswith('.ts') or file.path.endswith('.tsx'):
            syntax_ok = await check_typescript_syntax(file.content)
            if not syntax_ok:
                issues.append(f"Syntax error in {file.path}")
    
    # Check confidence threshold
    if output.confidence < 0.7:
        issues.append(f"Low confidence ({output.confidence}), may need review")
    
    return ValidationResult(
        valid=len(issues) == 0,
        issues=issues,
        requires_approval=output.needsReview or output.confidence < 0.5
    )
```

## Checkpointing
```python
async def checkpoint_execution(execution_id: str, completed_steps: list[str]):
    """
    Save execution state so it can be resumed after failure/pause.
    """
    state = {
        "execution_id": execution_id,
        "completed_steps": completed_steps,
        "step_outputs": {s: get_output(s) for s in completed_steps},
        "timestamp": datetime.utcnow().isoformat(),
        "repo_version": await get_current_repo_version(),
    }
    
    await storage.save_checkpoint(execution_id, state)

async def resume_from_checkpoint(execution_id: str):
    """
    Resume execution from last checkpoint.
    """
    state = await storage.load_checkpoint(execution_id)
    
    # Rebuild DAG with completed steps marked
    dag = await rebuild_dag(state["execution_id"])
    for step_id in state["completed_steps"]:
        dag.mark_completed(step_id, state["step_outputs"][step_id])
    
    # Continue execution from where we left off
    return await execute_dag(dag, config)
```
