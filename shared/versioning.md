# Versioning Strategy

## Overview

The AI Software Factory uses a multi-level versioning system to track changes across the platform, prompt repositories, APIs, database schemas, and generated artifacts. Every change is traceable, reversible, and auditable.

## Versioning Levels

| Level | Scope | Format | Example |
|-------|-------|--------|---------|
| Platform | Entire AI Factory platform | SemVer | `v2.3.1` |
| Repository | Each prompt repository | SemVer | `v1.5.0` |
| API | Each service API | URL prefix | `/v1/`, `/v2/` |
| Database | Schema migrations | Sequential | `001_initial.sql` |
| Prompt | Individual prompt templates | Hash-based | `prompt_abc123` |
| Artifact | Generated code outputs | Git SHA | `a1b2c3d` |

## Semantic Versioning Rules (SemVer)

Format: `MAJOR.MINOR.PATCH`

| Change Type | Version Bump | Example | Trigger |
|-------------|-------------|---------|---------|
| Breaking change | MAJOR | `1.0.0 → 2.0.0` | API contract change, schema redesign |
| New feature | MINOR | `1.0.0 → 1.1.0` | New module, new endpoint, new workflow |
| Bug fix | PATCH | `1.0.0 → 1.0.1` | Fix logic error, update docs |
| Hotfix | PATCH | `1.0.1 → 1.0.2` | Critical production fix |

## Repository Versioning (Prompt Repos)

### When to Bump

| Action | Bump | Reason |
|--------|------|--------|
| Add new feature module | MINOR | New capability |
| Add new API endpoint | MINOR | New interface |
| Change data model (additive) | MINOR | New fields/tables |
| Change data model (breaking) | MAJOR | Remove/rename fields |
| Fix service logic | PATCH | Bug correction |
| Update documentation only | PATCH | Clarity improvement |
| Change workflow states | MINOR | Behavior change |
| Remove a feature | MAJOR | Breaking removal |

### Merge Workflow (Team Lead)

```
1. AI generates output (code/docs/prompt changes)
2. Team Lead reviews output
3. If approved:
   a. Update affected files in repository
   b. Determine version bump type (major/minor/patch)
   c. Update changes/manifest.yaml
   d. Write changes/entries/vX.Y.Z.md
   e. Commit with message: "feat|fix|breaking: description [vX.Y.Z]"
4. If rejected:
   a. Provide feedback
   b. Re-enter iteration loop
```

### Changelog Entry Format

```markdown
# vX.Y.Z — YYYY-MM-DD

## Summary
One-line description of what changed.

## Changes
- [feature] Added X to module Y
- [fix] Corrected Z in service logic
- [breaking] Removed deprecated endpoint /v1/old

## Files Modified
- features/module_name/back/api_endpoints.md
- features/module_name/back/data_model.md

## Migration Required
- SQL: `ALTER TABLE x ADD COLUMN y ...`
- API: New endpoint `/v1/new-endpoint`

## Dependencies Updated
- features/other_module/front/pages.md (updated import)

## Approved By
- Team Lead: @name
- Date: YYYY-MM-DD
```

## API Versioning

### Strategy: URL Path Versioning
- Current: `/v1/tasks`, `/v1/plans`
- Breaking change: `/v2/tasks` (old remains active)
- Deprecation: 6-month sunset period with warning headers

### Backward Compatibility Rules
1. Adding new fields to response: OK (non-breaking)
2. Adding new optional query params: OK (non-breaking)
3. Removing response fields: BREAKING (new version)
4. Changing field types: BREAKING (new version)
5. Adding required request fields: BREAKING (new version)

## Database Versioning

### Migration Naming
```
{sequential_number}_{description}.sql
Example: 003_add_execution_metrics_table.sql
```

### Migration Rules
1. Forward-only (no down migrations in production)
2. Every migration must be idempotent (re-runnable)
3. Data migrations separate from schema migrations
4. Large table alterations must be online (no locks)
5. Every migration requires a rollback plan (documented, not automated)

### Schema Change Workflow
```
1. Update features/{module}/back/data_model.md
2. Generate migration SQL
3. Test migration on staging
4. Apply migration
5. Update API if needed
6. Bump version
7. Write changelog entry
```

## Prompt Versioning

### Template Versioning
Each prompt template has:
- Content hash (SHA-256 of template content)
- Version tag (linked to repo version)
- Performance metrics (success rate, token usage)

### A/B Testing
- New prompt versions can run alongside old ones
- Traffic split configurable (e.g., 90% old, 10% new)
- Automatic rollback if success rate drops below threshold

## Git Conventions

### Branch Strategy
```
main                    ← Production-ready
├── develop             ← Integration branch
│   ├── feature/xxx     ← New features
│   ├── fix/xxx         ← Bug fixes
│   └── hotfix/xxx      ← Critical fixes (branch from main)
```

### Commit Message Format
```
type(scope): description [version]

Types: feat, fix, breaking, docs, refactor, test, chore
Scope: module name or "platform"
Version: only on merge commits

Examples:
feat(planning): add risk scoring to plan generation [v1.3.0]
fix(orchestrator): handle timeout in parallel execution [v1.2.1]
breaking(api): remove deprecated /v1/legacy endpoint [v2.0.0]
```

## Dependency Tracking

### Inter-Module Dependencies
When changing a module, check and update dependents:

| If you change... | Also update... |
|-----------------|----------------|
| Data model | API endpoints, frontend pages, tests |
| API endpoint | Frontend components, integration tests |
| Workflow states | Feature brief, UI wireframes |
| Design system | All frontend pages using changed tokens |
| Shared/backend.md | All backend modules (review) |
| Shared/frontend.md | All frontend modules (review) |

### Dependency Graph (maintained in context_packs/command_room/module_dependencies.md)

## Golden Rules

1. **Never change a file without bumping version**
2. **Never change data model without migration SQL**
3. **Never change API without updating frontend pages**
4. **Never change workflow without updating feature brief**
5. **Every architecture decision must be logged in decisions_log.md**
6. **Every known bug must be logged in known_issues.md**
7. **AI only modifies loaded files — never guesses about other files**

## Anti-Patterns

| Bad Practice | Correct Approach |
|-------------|-----------------|
| Loading all 120 files | Load only 3-5 relevant files per task |
| Changing without changelog | Always write changelog entry |
| Changing data model without migration | Always write migration SQL |
| Direct editing without review | Always go through approval loop |
| Skipping version bump | Every change gets a version |
| Writing files in Persian | All content in English for AI quality |
