# Prompt Factory — Repository Lifecycle Workflow

## Overview

The Prompt Factory manages the entire lifecycle of a Prompt Repository from initial creation through continuous evolution.

## Phase 1: Repository Creation (from Product Definition)

```
[User submits Product Definition Prompt]
    ↓
[Planning Service generates initial Plan Schema]
    ↓
[Plan includes: modules, features, tech stack, personas]
    ↓
[Prompt Factory receives approved Plan]
    ↓
[Generate Repository Skeleton]
    ├── README.md (from plan.title, plan.description)
    ├── shared/
    │   ├── backend.md (from plan.tech_stack.backend)
    │   ├── frontend.md (from plan.tech_stack.frontend)
    │   ├── design.md (from plan.design_system)
    │   ├── app.md (from plan.modules, plan.context_loading)
    │   ├── personas.md (from plan.personas)
    │   └── versioning.md (standard template)
    ├── features/ (one per plan.modules[])
    │   └── {module_name}/
    │       ├── feature_brief.md (from plan.modules[].description)
    │       ├── back/ (empty placeholders)
    │       ├── front/ (empty placeholders)
    │       ├── ui/ (empty placeholders)
    │       └── workflows/ (empty placeholders)
    ├── context_packs/ (standard set)
    └── changes/
        ├── manifest.yaml
        ├── VERSIONING_GUIDE.md
        └── entries/v0.1.0.md
    ↓
[Version: v0.1.0 — Skeleton]
    ↓
[Human Review + Approval]
    ↓
[Iterate if needed (back to Planning)]
```

## Phase 2: Feature Elaboration (Iterative)

```
For each module in priority order:
    ↓
[Planning Service generates detailed feature plan]
    ↓
[Plan includes: data model, API endpoints, UI pages, workflows]
    ↓
[Prompt Factory fills module files]
    ├── features/{module}/back/data_model.md
    ├── features/{module}/back/api_endpoints.md
    ├── features/{module}/back/service_logic.md
    ├── features/{module}/front/pages.md
    ├── features/{module}/front/components.md
    ├── features/{module}/ui/wireframes.md
    └── features/{module}/workflows/{workflow_name}.md
    ↓
[Version bump: v0.1.0 → v0.2.0 (first module)]
    ↓
[Human Review (Team Lead)]
    ├── Approved → Merge to main
    ├── Needs changes → Back to Planning
    └── Rejected → Discard branch
    ↓
[Repeat for next module]
    ↓
[All modules elaborated → v1.0.0]
```

## Phase 3: Code Generation (Execution)

```
[Repository at v1.0.0+ (ready for code generation)]
    ↓
[Orchestrator creates execution plan (DAG)]
    ↓
[For each step in DAG:]
    ↓
[Prompt Factory assembles context]
    = Loading Formula files for agent_type + module
    + Dependency outputs from previous steps
    + Relevant patterns from Knowledge Base
    ↓
[Agent executes and produces code]
    ↓
[Code reviewed (Review Agent or Human)]
    ↓
[If code changes require repo updates:]
    ├── New API discovered → update api_endpoints.md
    ├── Schema changed → update data_model.md
    ├── New component → update components.md
    └── Bug found → update known_issues.md
    ↓
[Prompt Factory commits changes → version bump]
    ↓
[Knowledge Base re-indexes changed files]
```

## Phase 4: Continuous Evolution

```
[Change Request (bug fix, new feature, refactor)]
    ↓
[Planning Service analyzes impact]
    ├── Which modules affected?
    ├── Which files need update?
    ├── What's the risk level?
    └── Estimated cost?
    ↓
[Create Branch in Prompt Repository]
    branch: "feature/{change_name}" or "fix/{bug_name}"
    ↓
[Update affected files on branch]
    ↓
[Merge Request → Team Lead reviews]
    ↓
[Merge → Version bump]
    ↓
[Orchestrator executes code changes]
    ↓
[Feedback Engine records outcome]
    ↓
[Knowledge Base learns from result]
    ↓
[Loop continues...]
```

## Delta Update Strategy (Baseline/Delta)

### Concept
Instead of regenerating entire files, the system tracks **deltas** (changes) relative to a baseline.

### Implementation
```python
class DeltaManager:
    """
    Manages baseline/delta updates to minimize token usage
    and maintain consistency.
    """
    
    async def apply_delta(
        self,
        repo_id: str,
        file_path: str,
        delta: Delta
    ) -> str:
        """
        Apply a delta to a file without rewriting the entire content.
        
        Delta types:
        - section_add: Add a new section (## heading + content)
        - section_replace: Replace content under a specific heading
        - section_delete: Remove a section
        - line_insert: Insert lines at specific position
        - line_replace: Replace specific lines
        - append: Add content at end of file
        """
        current_content = await db.get_repo_file_content(repo_id, file_path)
        
        if delta.type == "section_add":
            new_content = self._add_section(current_content, delta.after_section, delta.content)
        elif delta.type == "section_replace":
            new_content = self._replace_section(current_content, delta.section_heading, delta.content)
        elif delta.type == "section_delete":
            new_content = self._delete_section(current_content, delta.section_heading)
        elif delta.type == "append":
            new_content = current_content + "\n\n" + delta.content
        else:
            raise ValueError(f"Unknown delta type: {delta.type}")
        
        return new_content
    
    async def generate_delta_prompt(
        self,
        repo_id: str,
        file_path: str,
        change_description: str
    ) -> str:
        """
        Generate a prompt for AI to produce a delta (not full file rewrite).
        This is the key to efficiency — AI only needs to describe the change,
        not regenerate the entire file.
        """
        current_content = await db.get_repo_file_content(repo_id, file_path)
        
        prompt = f"""
You are updating an existing document. DO NOT rewrite the entire file.
Instead, provide ONLY the delta (change) in the following format:

CURRENT FILE: {file_path}
---
{current_content}
---

REQUESTED CHANGE: {change_description}

OUTPUT FORMAT:
```delta
type: section_replace | section_add | section_delete | append
target_section: "## Section Heading" (if applicable)
after_section: "## Previous Section" (for section_add)
content: |
  (new content here)
```

Only output the delta, not the full file.
"""
        return prompt
```

## Conflict Resolution

```python
class ConflictResolver:
    """
    Handles merge conflicts when multiple branches modify the same file.
    """
    
    async def detect_conflicts(
        self,
        repo_id: str,
        branch_a: str,
        branch_b: str
    ) -> list[Conflict]:
        """
        Detect conflicts between two branches.
        Conflict = same file modified in both branches at overlapping sections.
        """
        changes_a = await db.get_branch_changes(repo_id, branch_a)
        changes_b = await db.get_branch_changes(repo_id, branch_b)
        
        conflicts = []
        for file_a in changes_a:
            for file_b in changes_b:
                if file_a.path == file_b.path:
                    # Check if changes overlap
                    if self._sections_overlap(file_a.sections_changed, file_b.sections_changed):
                        conflicts.append(Conflict(
                            path=file_a.path,
                            branch_a_change=file_a,
                            branch_b_change=file_b,
                            type="section_overlap"
                        ))
        
        return conflicts
    
    async def auto_resolve(self, conflict: Conflict) -> Resolution:
        """
        Attempt automatic resolution:
        - If changes are in different sections → merge both
        - If one is additive and other is modification → merge both
        - Otherwise → escalate to human
        """
        if conflict.type == "different_sections":
            return Resolution(strategy="merge_both", auto=True)
        
        if conflict.branch_a_change.is_additive and not conflict.branch_b_change.is_additive:
            return Resolution(strategy="merge_both", auto=True)
        
        return Resolution(strategy="human_review", auto=False)
```

## File Template System

When creating new modules, the Prompt Factory uses templates:

```python
TEMPLATES = {
    "feature_brief": """# {module_title} — Feature Brief

## Module Identity

| Field | Value |
|-------|-------|
| Module ID | `{module_id}` |
| Owner | {owner} |
| Status | Planned |
| Priority | {priority} |

## Purpose

{purpose_description}

## Core Capabilities

### 1. {capability_1}
{capability_1_description}

## API Endpoints

(To be defined during elaboration phase)

## Data Model

(To be defined during elaboration phase)

## Success Metrics

| Metric | Target |
|--------|--------|
| (TBD) | (TBD) |
""",
    
    "data_model": """# {module_title} — Data Model

## Tables

### Table: {primary_table}
```sql
CREATE TABLE {primary_table} (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  -- Fields to be defined
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

## Relationships

(To be defined)

## Indexes

(To be defined based on query patterns)
""",
    
    "api_endpoints": """# {module_title} — API Endpoints

## Base Path: /api/v1/{module_id}

## Endpoints

### GET /api/v1/{module_id}
List all {entities}.

### POST /api/v1/{module_id}
Create new {entity}.

### GET /api/v1/{module_id}/:id
Get single {entity} by ID.

### PUT /api/v1/{module_id}/:id
Update {entity}.

### DELETE /api/v1/{module_id}/:id
Delete {entity}.
"""
}
```
