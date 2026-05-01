# Prompt Factory — Data Model

## Core Concept

The Prompt Factory manages **Prompt Repositories** — structured file collections that serve as the single source of truth for each project. It handles creation, versioning, merging, and context assembly.

## Table: prompt_repositories
```sql
CREATE TABLE prompt_repositories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) UNIQUE,
  
  name VARCHAR(100) NOT NULL,
  description TEXT,
  
  current_version VARCHAR(20) NOT NULL DEFAULT '0.1.0',
  total_files INTEGER NOT NULL DEFAULT 0,
  total_tokens INTEGER NOT NULL DEFAULT 0,
  
  structure JSONB NOT NULL DEFAULT '{}',
  -- Cached directory tree for quick access
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

## Table: repo_files
```sql
CREATE TABLE repo_files (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID NOT NULL REFERENCES prompt_repositories(id),
  
  path VARCHAR(500) NOT NULL,  -- e.g., "features/orders/back/api_endpoints.md"
  content TEXT NOT NULL,
  
  file_type VARCHAR(30) NOT NULL,
  -- Types: feature_brief | data_model | api_spec | workflow | wireframe |
  --        component | page | convention | persona | changelog
  
  module VARCHAR(50),  -- e.g., "order_management"
  layer VARCHAR(20),   -- e.g., "back", "front", "ui", "workflows"
  
  version VARCHAR(20) NOT NULL DEFAULT '0.1.0',
  checksum VARCHAR(64) NOT NULL,
  token_count INTEGER NOT NULL,
  
  last_modified_by UUID REFERENCES users(id),
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  
  UNIQUE(repo_id, path)
);

CREATE INDEX idx_repo_files_repo ON repo_files(repo_id);
CREATE INDEX idx_repo_files_module ON repo_files(module);
CREATE INDEX idx_repo_files_type ON repo_files(file_type);
CREATE INDEX idx_repo_files_path ON repo_files(path);
```

## Table: repo_commits
```sql
CREATE TABLE repo_commits (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID NOT NULL REFERENCES prompt_repositories(id),
  
  version VARCHAR(20) NOT NULL,
  parent_version VARCHAR(20),
  
  message TEXT NOT NULL,
  description TEXT,
  
  changes JSONB NOT NULL,
  -- [{path, operation: "created"|"modified"|"deleted", diff_summary}]
  
  files_changed INTEGER NOT NULL DEFAULT 0,
  
  committed_by UUID REFERENCES users(id),
  committed_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  
  UNIQUE(repo_id, version)
);

CREATE INDEX idx_repo_commits_repo ON repo_commits(repo_id);
CREATE INDEX idx_repo_commits_version ON repo_commits(version);
```

## Table: repo_branches (for parallel work)
```sql
CREATE TABLE repo_branches (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID NOT NULL REFERENCES prompt_repositories(id),
  
  name VARCHAR(100) NOT NULL,
  -- e.g., "feature/add-priority", "fix/payment-flow"
  
  base_version VARCHAR(20) NOT NULL,
  current_version VARCHAR(20) NOT NULL,
  
  status VARCHAR(20) NOT NULL DEFAULT 'active',
  -- Status: active | merged | abandoned
  
  created_by UUID REFERENCES users(id),
  merged_by UUID REFERENCES users(id),
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  merged_at TIMESTAMP WITH TIME ZONE,
  
  UNIQUE(repo_id, name)
);
```

## Table: merge_requests
```sql
CREATE TABLE merge_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID NOT NULL REFERENCES prompt_repositories(id),
  branch_id UUID NOT NULL REFERENCES repo_branches(id),
  
  title VARCHAR(200) NOT NULL,
  description TEXT,
  
  status VARCHAR(20) NOT NULL DEFAULT 'open',
  -- Status: open | approved | merged | rejected | conflict
  
  changes JSONB NOT NULL,
  -- [{path, operation, before_content, after_content}]
  
  conflicts JSONB DEFAULT '[]',
  -- [{path, conflict_type, resolution}]
  
  requested_by UUID REFERENCES users(id),
  reviewed_by UUID REFERENCES users(id),
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  reviewed_at TIMESTAMP WITH TIME ZONE,
  merged_at TIMESTAMP WITH TIME ZONE
);
```

## Table: context_assemblies (cached assembled contexts)
```sql
CREATE TABLE context_assemblies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID NOT NULL REFERENCES prompt_repositories(id),
  
  purpose VARCHAR(100) NOT NULL,
  -- e.g., "code_agent:orders:add_priority_field"
  
  files_included TEXT[] NOT NULL,
  -- Ordered list of file paths included
  
  assembled_content TEXT NOT NULL,
  total_tokens INTEGER NOT NULL,
  
  repo_version VARCHAR(20) NOT NULL,
  -- Version at time of assembly (for cache invalidation)
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_context_assemblies_repo ON context_assemblies(repo_id);
CREATE INDEX idx_context_assemblies_purpose ON context_assemblies(purpose);
```

## Context Assembly Logic

```python
class ContextAssembler:
    """
    Assembles the right set of files for a given task.
    Uses the Loading Formula from shared/app.md.
    """
    
    LOADING_FORMULAS = {
        "code_agent_backend": [
            "shared/backend.md",
            "features/{module}/feature_brief.md",
            "features/{module}/back/data_model.md",
            "features/{module}/back/api_endpoints.md",
            "shared/versioning.md"
        ],
        "code_agent_frontend": [
            "shared/frontend.md",
            "shared/design.md",
            "features/{module}/feature_brief.md",
            "features/{module}/front/pages.md",
            "features/{module}/front/components.md",
            "features/{module}/ui/wireframes.md"
        ],
        "test_agent": [
            "features/{module}/back/data_model.md",
            "features/{module}/back/api_endpoints.md",
            "context_packs/backend/testing_strategy.md"
        ],
        "review_agent": [
            "shared/backend.md",
            "context_packs/backend/auth_and_security.md",
            "features/{module}/feature_brief.md"
        ],
        "planning_service": [
            "shared/app.md",
            "shared/backend.md",
            "shared/frontend.md",
            "features/{module}/feature_brief.md"
        ]
    }
    
    async def assemble(
        self,
        repo_id: str,
        agent_type: str,
        module: str,
        additional_files: list[str] = None,
        max_tokens: int = 50000
    ) -> str:
        """
        Assemble context for an agent working on a specific module.
        """
        formula_key = agent_type
        formula = self.LOADING_FORMULAS.get(formula_key, [])
        
        # Resolve module placeholder
        file_paths = [p.replace("{module}", module) for p in formula]
        
        # Add additional files
        if additional_files:
            file_paths.extend(additional_files)
        
        # Load files respecting token budget
        assembled = []
        tokens_used = 0
        
        for path in file_paths:
            file = await db.get_repo_file(repo_id, path)
            if not file:
                continue
            
            if tokens_used + file.token_count > max_tokens:
                break
            
            assembled.append(f"--- FILE: {path} ---\n\n{file.content}\n\n")
            tokens_used += file.token_count
        
        return "\n".join(assembled)
```

## Versioning Logic

```python
class RepoVersionManager:
    """
    Manages semantic versioning for the prompt repository.
    """
    
    async def commit_changes(
        self,
        repo_id: str,
        changes: list[FileChange],
        message: str,
        committed_by: str,
        bump_type: str = "auto"  # auto | patch | minor | major
    ) -> str:
        """
        Commit changes to the repository with version bump.
        """
        repo = await db.get_repo(repo_id)
        current = parse_version(repo.current_version)
        
        # Auto-detect bump type
        if bump_type == "auto":
            bump_type = self._detect_bump_type(changes)
        
        # Calculate new version
        new_version = self._bump_version(current, bump_type)
        
        # Apply changes to repo_files
        for change in changes:
            if change.operation == "created":
                await db.create_repo_file(repo_id, change.path, change.content, new_version)
            elif change.operation == "modified":
                await db.update_repo_file(repo_id, change.path, change.content, new_version)
            elif change.operation == "deleted":
                await db.delete_repo_file(repo_id, change.path)
        
        # Create commit record
        await db.create_commit(repo_id, new_version, repo.current_version, message, changes)
        
        # Update repo version
        await db.update_repo_version(repo_id, new_version)
        
        # Trigger Knowledge Base re-indexing
        changed_paths = [c.path for c in changes if c.operation != "deleted"]
        await knowledge_base.on_repo_updated(repo_id, changed_paths)
        
        return new_version
    
    def _detect_bump_type(self, changes: list[FileChange]) -> str:
        """
        Auto-detect version bump type:
        - major: new module added, breaking API change
        - minor: new feature file, new workflow
        - patch: content update, fix, documentation
        """
        for change in changes:
            if change.operation == "created" and "/feature_brief.md" in change.path:
                return "minor"  # New feature module
            if "breaking" in change.message.lower():
                return "major"
        
        has_new_files = any(c.operation == "created" for c in changes)
        if has_new_files:
            return "minor"
        
        return "patch"
```
