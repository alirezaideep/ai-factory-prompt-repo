# Prompt Factory — Feature Brief

## Module Identity

| Field | Value |
|-------|-------|
| Module ID | `prompt_factory` |
| Layer | 4 (Repository Management) |
| Owner | Backend Team Lead |
| Status | Planned |
| Priority | Critical (core differentiator) |

## Purpose

The Prompt Factory is the **heart** of the AI Software Factory. It manages the Prompt Repository — the structured collection of Markdown files that serve as the "source of truth" for any AI agent generating code. It handles:
- Initial repository creation from product definition
- Incremental updates (add/modify/remove features)
- Version control with semantic versioning
- Dependency tracking between files
- Context assembly for execution agents
- Baseline/Delta management

## Core Capabilities

### 1. Repository Initialization
- Accept product definition (PRD, feature list, tech stack)
- Generate complete repository skeleton:
  - `shared/` — backend, frontend, design, app, personas, versioning
  - `features/` — one directory per module with brief, back, front, ui, workflows
  - `context_packs/` — backend, frontend, operations, command_room, memory, product_management
  - `changes/` — manifest, versioning guide, initial entry
- Validate completeness (all required files present)
- Set initial version (v0.1.0)

### 2. Incremental Updates (Baseline/Delta)
- **Baseline:** The current state of the repository (latest version)
- **Delta:** A set of changes to apply to specific files
- Never rewrite entire repository for small changes
- Delta format:
```json
{
  "version": "v1.3.0",
  "type": "minor",
  "description": "Add urgent order support",
  "changes": [
    {
      "file": "features/order_management/back/api_endpoints.md",
      "action": "append_section",
      "section_title": "### POST /api/v1/orders/:id/urgent",
      "content": "..."
    },
    {
      "file": "features/order_management/back/data_model.md",
      "action": "modify_section",
      "section_title": "### Table: orders",
      "find": "status VARCHAR(20)",
      "replace": "status VARCHAR(20),\n  priority VARCHAR(10) DEFAULT 'normal'"
    },
    {
      "file": "features/order_management/workflows/order_lifecycle.md",
      "action": "append_section",
      "section_title": "## Urgent Order Flow",
      "content": "..."
    }
  ]
}
```

### 3. Dependency Tracking
- Track which files reference which other files
- When a file changes, identify potentially affected files
- Dependency graph:
```
shared/backend.md ← features/*/back/api_endpoints.md
shared/frontend.md ← features/*/front/pages.md
shared/design.md ← features/*/ui/wireframes.md
features/X/back/data_model.md ← features/X/back/api_endpoints.md
features/X/back/api_endpoints.md ← features/X/front/pages.md
features/X/feature_brief.md ← features/X/back/* + features/X/front/*
```

### 4. Context Assembly
- Given a task (from Plan Schema), assemble the minimal set of files needed
- Respect token budget (max context window size)
- Priority order:
  1. Directly referenced files (from plan step)
  2. Dependency parents (files that the referenced files depend on)
  3. Shared conventions (memory/conventions.md)
  4. Historical context (similar past changes)
- Output: ordered list of file contents, ready for LLM prompt

### 5. Version Control
- Semantic versioning (MAJOR.MINOR.PATCH)
- Every change creates a new version entry
- Changelog auto-generated from delta
- Rollback support (revert to any previous version)
- Branch support (experimental changes without affecting main)

### 6. Validation
- Schema validation (required sections present in each file type)
- Cross-reference validation (no broken links between files)
- Consistency validation (API endpoints match data model fields)
- Completeness validation (every feature has all required subdirectories)

## API Endpoints

### POST /api/v1/prompt-factory/initialize
Create new repository from product definition.
```json
// Request
{
  "project_id": "proj_abc",
  "product_definition": {
    "name": "Qazvin Glass Order System",
    "description": "B2B order management for glass manufacturing",
    "tech_stack": {"backend": "Node.js/Express", "frontend": "React/Next.js", "db": "PostgreSQL"},
    "modules": ["order_management", "production", "inventory", "customers", "payments"],
    "personas": ["factory_owner", "sales_rep", "production_manager", "warehouse_operator"]
  }
}

// Response 201
{
  "success": true,
  "data": {
    "repo_id": "repo_xyz",
    "version": "v0.1.0",
    "files_created": 85,
    "structure": { ... }
  }
}
```

### POST /api/v1/prompt-factory/repos/:repoId/apply-delta
Apply incremental change to repository.
```json
// Request
{
  "delta": {
    "description": "Add urgent order support",
    "version_bump": "minor",
    "changes": [ ... ]
  },
  "validated": true,
  "approved_by": "team_lead_id"
}

// Response 200
{
  "success": true,
  "data": {
    "new_version": "v1.3.0",
    "files_modified": 3,
    "files_added": 0,
    "files_removed": 0,
    "dependency_warnings": []
  }
}
```

### GET /api/v1/prompt-factory/repos/:repoId/assemble-context
Assemble context for an execution step.
```json
// Request (query params)
?step_id=step_3&plan_id=plan_xyz&max_tokens=50000

// Response 200
{
  "success": true,
  "data": {
    "files": [
      {"path": "shared/backend.md", "content": "...", "tokens": 3200},
      {"path": "features/orders/back/api.md", "content": "...", "tokens": 2800}
    ],
    "total_tokens": 12500,
    "within_budget": true
  }
}
```

### GET /api/v1/prompt-factory/repos/:repoId/validate
Validate repository integrity.
```json
// Response 200
{
  "success": true,
  "data": {
    "valid": true,
    "warnings": ["features/payments/ui/wireframes.md is empty"],
    "errors": [],
    "coverage": {
      "features_with_all_files": 7,
      "features_missing_files": 1,
      "broken_references": 0
    }
  }
}
```

### GET /api/v1/prompt-factory/repos/:repoId/dependencies
Get dependency graph.

### POST /api/v1/prompt-factory/repos/:repoId/rollback
Rollback to a previous version.

### GET /api/v1/prompt-factory/repos/:repoId/history
Get version history.

## Data Model

### Table: repositories
```sql
CREATE TABLE repositories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id),
  name VARCHAR(100) NOT NULL,
  current_version VARCHAR(20) NOT NULL DEFAULT 'v0.1.0',
  
  structure JSONB NOT NULL,  -- File tree structure
  metadata JSONB DEFAULT '{}',
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

### Table: repo_files
```sql
CREATE TABLE repo_files (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID NOT NULL REFERENCES repositories(id),
  path VARCHAR(500) NOT NULL,
  content TEXT NOT NULL,
  
  version VARCHAR(20) NOT NULL,
  checksum VARCHAR(64) NOT NULL,  -- SHA-256 of content
  token_count INTEGER NOT NULL,
  
  dependencies TEXT[] DEFAULT '{}',  -- Paths this file depends on
  dependents TEXT[] DEFAULT '{}',  -- Paths that depend on this file
  
  last_modified_by UUID REFERENCES users(id),
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  
  UNIQUE(repo_id, path, version)
);

CREATE INDEX idx_repo_files_repo_path ON repo_files(repo_id, path);
CREATE INDEX idx_repo_files_version ON repo_files(version);
```

### Table: repo_versions
```sql
CREATE TABLE repo_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID NOT NULL REFERENCES repositories(id),
  version VARCHAR(20) NOT NULL,
  
  type VARCHAR(10) NOT NULL,  -- major | minor | patch
  description TEXT NOT NULL,
  
  delta JSONB NOT NULL,  -- The changes applied
  files_modified TEXT[] NOT NULL,
  files_added TEXT[] DEFAULT '{}',
  files_removed TEXT[] DEFAULT '{}',
  
  applied_by UUID REFERENCES users(id),
  plan_id UUID REFERENCES plans(id),  -- Which plan triggered this
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  
  UNIQUE(repo_id, version)
);
```

## Workflow: Repository Update Cycle

```
Plan Approved
    ↓
[Orchestrator executes steps]
    ↓
[Agent produces output (code/docs)]
    ↓
[Team Lead reviews output]
    ↓
[Team Lead approves merge]
    ↓
[Prompt Factory receives delta]
    ↓
[Validate delta against current baseline]
    ↓
[Check dependency impacts]
    ↓
[Apply delta to affected files]
    ↓
[Bump version]
    ↓
[Generate changelog entry]
    ↓
[Update dependency graph]
    ↓
[Notify affected team members]
    ↓
[Repository ready for next cycle]
```

## Success Metrics

| Metric | Target |
|--------|--------|
| Repository initialization time | < 2 minutes |
| Delta application time | < 5 seconds |
| Context assembly time | < 2 seconds |
| Validation accuracy | 100% (no false positives) |
| Version history integrity | 100% (no gaps) |
| Dependency tracking accuracy | > 95% |
