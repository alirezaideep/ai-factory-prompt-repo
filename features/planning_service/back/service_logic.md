# Planning Service — Service Logic

## Core Algorithm: Goal → Plan

### Step 1: Goal Parsing
```python
async def parse_goal(task: Task) -> ParsedGoal:
    """
    Uses LLM to extract structured information from natural language goal.
    
    Input: task.goal + task.scope + task.description
    Output: ParsedGoal with sub_tasks, affected_entities, change_type
    """
    prompt = f"""
    You are a software planning agent. Analyze this goal and decompose it.
    
    Goal: {task.goal}
    Scope: {task.scope}
    Description: {task.description}
    
    Available modules: {list_all_modules()}
    
    Output JSON:
    {{
      "sub_tasks": [
        {{"description": "...", "type": "backend|frontend|test|docs|devops", "complexity": "low|medium|high"}}
      ],
      "affected_modules": ["module_name"],
      "change_type": "feature|bugfix|refactor|docs",
      "requires_migration": true/false,
      "requires_api_change": true/false
    }}
    """
    return await llm_call(prompt, response_format=ParsedGoalSchema)
```

### Step 2: Context Selection
```python
async def select_context(parsed_goal: ParsedGoal) -> list[str]:
    """
    Determines which repository files each sub-task needs.
    Uses Context Loading Table from shared/app.md as primary guide.
    """
    context_files = []
    
    for module in parsed_goal.affected_modules:
        # Always load feature brief
        context_files.append(f"features/{module}/feature_brief.md")
        
        # Load based on change type
        if parsed_goal.change_type in ["feature", "bugfix"]:
            context_files.append(f"features/{module}/back/api_endpoints.md")
            context_files.append(f"features/{module}/back/data_model.md")
            context_files.append(f"features/{module}/front/pages.md")
        
        if parsed_goal.requires_migration:
            context_files.append(f"features/{module}/back/data_model.md")
            context_files.append("shared/versioning.md")
        
        if parsed_goal.requires_api_change:
            context_files.append(f"features/{module}/back/api_endpoints.md")
            context_files.append("shared/backend.md")
    
    # Always include shared context
    context_files.append("shared/backend.md")
    context_files.append("context_packs/memory/conventions.md")
    
    return deduplicate(context_files)
```

### Step 3: DAG Construction
```python
async def build_dag(parsed_goal: ParsedGoal, context: list[str]) -> DAG:
    """
    Constructs execution DAG with proper dependencies.
    
    Rules:
    1. Data model changes before API changes
    2. Backend before frontend (if dependent)
    3. Implementation before tests
    4. Tests before review
    5. Review before docs/merge
    6. Independent modules can run in parallel
    """
    prompt = f"""
    Build an execution DAG for these sub-tasks.
    
    Sub-tasks: {parsed_goal.sub_tasks}
    Affected modules: {parsed_goal.affected_modules}
    
    Rules:
    - Assign each step to an agent: code_agent, test_agent, review_agent, docs_agent
    - Respect dependency order: schema → migration → API → frontend → tests → review → docs
    - Maximize parallelism where safe
    - Each step must list required context_files from: {context}
    
    Output DAG JSON with nodes and edges.
    """
    dag = await llm_call(prompt, response_format=DAGSchema)
    
    # Validate: no cycles
    if has_cycle(dag):
        raise PlanValidationError("DAG contains cycle")
    
    # Validate: all nodes reachable from root
    if not all_reachable(dag):
        raise PlanValidationError("Unreachable nodes detected")
    
    return dag
```

### Step 4: Cost Estimation
```python
def estimate_cost(dag: DAG) -> CostEstimate:
    """
    Estimates token usage and USD cost based on:
    1. Historical data for similar tasks (from Knowledge Base)
    2. Step complexity (low=2K, medium=5K, high=10K tokens)
    3. Agent type multipliers (review=0.5x, docs=0.3x)
    """
    total_tokens = 0
    breakdown = {}
    
    for node in dag.nodes:
        base_tokens = COMPLEXITY_MAP[node.complexity]  # low=2000, med=5000, high=10000
        agent_multiplier = AGENT_MULTIPLIER[node.agent]  # code=1.0, test=0.8, review=0.5, docs=0.3
        estimated = int(base_tokens * agent_multiplier)
        
        total_tokens += estimated
        breakdown[node.agent] = breakdown.get(node.agent, 0) + estimated
    
    usd = total_tokens * PRICE_PER_TOKEN  # e.g., $0.00003 per token
    
    return CostEstimate(
        total_tokens=total_tokens,
        estimated_usd=round(usd, 2),
        confidence=calculate_confidence(dag),
        breakdown_by_agent=breakdown
    )
```

### Step 5: Risk Assessment
```python
def assess_risk(parsed_goal: ParsedGoal, dag: DAG) -> RiskAssessment:
    """
    Rule-based risk scoring with mitigation suggestions.
    """
    score = 0.0
    factors = []
    
    if any(m in SHARED_MODULES for m in parsed_goal.affected_modules):
        score += 0.2
        factors.append("Modifies shared module")
    
    if parsed_goal.requires_migration:
        score += 0.15
        factors.append("Requires database migration")
    
    if parsed_goal.requires_api_change:
        score += 0.15
        factors.append("Breaking API change possible")
    
    if len(dag.nodes) > 10:
        score += 0.1
        factors.append("Large scope (>10 steps)")
    
    if not has_test_step(dag):
        score += 0.2
        factors.append("No test step in plan")
    
    # Check historical success rate for similar tasks
    historical = knowledge_base.find_similar(parsed_goal)
    if historical.success_rate < 0.7:
        score += 0.1
        factors.append("Similar tasks had low success rate")
    
    mitigations = generate_mitigations(factors)
    
    return RiskAssessment(
        score=min(score, 1.0),
        level="low" if score < 0.3 else "medium" if score < 0.6 else "high",
        factors=factors,
        mitigations=mitigations
    )
```

## Regeneration Logic (Iterative Loop)

```python
async def regenerate_plan(plan_id: str, feedback: str, modifications: list) -> Plan:
    """
    Regenerates plan incorporating human feedback.
    Preserves approved steps, only modifies requested changes.
    """
    current_plan = await get_plan(plan_id)
    
    prompt = f"""
    Modify this execution plan based on feedback.
    
    Current plan: {current_plan.dag}
    Human feedback: {feedback}
    Specific modifications: {modifications}
    
    Rules:
    - Keep unchanged steps identical
    - Only modify/add/remove steps as requested
    - Maintain valid DAG (no cycles)
    - Recalculate affected dependencies
    - Update cost and timeline estimates
    """
    
    new_dag = await llm_call(prompt, response_format=DAGSchema)
    
    new_plan = Plan(
        task_id=current_plan.task_id,
        version=current_plan.version + 1,
        parent_version=current_plan.version,
        feedback=feedback,
        dag=new_dag,
        cost_estimate=estimate_cost(new_dag),
        risk_assessment=assess_risk(parsed_goal, new_dag),
        timeline=calculate_timeline(new_dag)
    )
    
    # Mark old version as superseded
    current_plan.status = "superseded"
    
    return new_plan
```

## Configuration

```yaml
planning:
  llm:
    model: "claude-3-opus"
    temperature: 0.2  # Low for structured output
    max_tokens: 4000
    timeout_seconds: 60
  
  limits:
    max_steps_per_plan: 20
    max_iterations: 5  # Max regeneration attempts
    max_parallel_generations: 10
  
  cost:
    price_per_token: 0.00003
    confidence_threshold: 0.6
  
  risk:
    auto_approve_threshold: 0.2  # Below this, can auto-approve (if FF enabled)
    escalation_threshold: 0.8  # Above this, escalate to Project Owner
```
