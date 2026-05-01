# Versioning Guide

## Overview

This repository uses a multi-level versioning system to ensure traceability, reproducibility, and safe evolution of the AI Software Factory platform.

## Versioning Levels

### Level 1: Repository Version (SemVer)
The overall prompt repository version. Tracked in `manifest.yaml`.

Format: `MAJOR.MINOR.PATCH`

| Component | When to Bump | Example |
|-----------|-------------|---------|
| MAJOR | Breaking changes to loading formula, incompatible restructuring | Rename shared/ to specs/ |
| MINOR | New module, new context pack, new convention | Add notifications module |
| PATCH | Fix typo, update example, clarify wording | Fix SQL syntax in data_model |

### Level 2: API Version
Each module's API endpoints are versioned independently.

Format: `/api/v{N}/...` where N is an integer.

Rules:
- New endpoints → same version (additive, non-breaking)
- Changed request/response shape → new version (v1 → v2)
- Old versions supported for 2 major releases minimum
- Deprecation notice 1 version before removal

### Level 3: Database Schema Version
Managed through sequential migration files.

Format: `{YYYYMMDD}_{sequence}_{description}.sql`

Rules:
- Every schema change requires a migration file
- Migrations are forward-only (no down migrations in production)
- Destructive changes (DROP, ALTER column type) require explicit approval
- Data migrations separate from schema migrations

### Level 4: File-Level Version
Each file in the repository has an implicit version via Git commit history.

Tracked via:
- Git blame (who changed what, when)
- Changelog entries (what changed in each version)
- File header comments (optional, for critical files)

## Change Workflow

### Step 1: Identify Change Type
```
Is it a new feature/module?     → MINOR bump
Is it a bug fix/typo?           → PATCH bump
Does it break existing usage?   → MAJOR bump
```

### Step 2: Make Changes
1. Create a branch: `feature/xxx` or `fix/xxx`
2. Edit the relevant files
3. Update any dependent files (if API changes, update front/ pages too)
4. Run validation (check cross-references, verify loading formula)

### Step 3: Create Changelog Entry
Create `changes/entries/vX.Y.Z.md` with:
- Summary of changes
- Files modified
- Migration notes (if any)
- Breaking changes (if any)

### Step 4: Update Manifest
Update `changes/manifest.yaml`:
- Bump `current_version`
- Add new entry to `versions` list
- Update `last_updated`

### Step 5: Review and Merge
- Team lead reviews the change
- Verify no broken cross-references
- Merge to develop → main

## Delta Update Protocol

### For AI Agents
When an AI agent needs to update the repository, it follows this protocol:

```
1. READ the specific file(s) to be changed
2. IDENTIFY the exact section to modify
3. PRODUCE a delta (not full file rewrite):
   - For additions: specify insertion point + new content
   - For modifications: specify old text + new text
   - For deletions: specify text to remove
4. VERSION: increment appropriate version
5. LOG: add changelog entry
```

### Delta Format
```yaml
delta:
  file: "features/orchestrator/back/api_endpoints.md"
  operation: "add"  # add | modify | delete
  anchor: "## Endpoints"  # Where to apply change
  position: "after"  # before | after | replace
  content: |
    ### POST /api/v1/orchestrator/executions/{id}/pause
    Pause a running execution.
    ...
```

### Why Deltas (Not Full Rewrites)
1. **Efficiency:** AI doesn't need to regenerate unchanged content
2. **Safety:** Less chance of accidentally removing existing content
3. **Reviewability:** Humans can see exactly what changed
4. **Conflict reduction:** Smaller changes = fewer merge conflicts
5. **Token savings:** Less input/output tokens per operation

## Cross-Reference Integrity

### Rules
When changing a file, check if other files reference it:

| If You Change... | Also Update... |
|-----------------|----------------|
| API endpoint signature | Frontend pages that call it |
| Database table/column | API endpoints + data_model |
| Workflow state | State transitions + UI wireframes |
| Feature brief scope | All sub-files (back/, front/, ui/, workflows/) |
| Shared convention | All features that follow it |
| Loading formula | app.md context table |

### Validation Script
```bash
# Check for broken cross-references
./scripts/validate-refs.sh

# Check for orphaned files (not referenced anywhere)
./scripts/find-orphans.sh

# Check version consistency
./scripts/check-versions.sh
```

## Rollback Procedure

If a version introduces problems:

1. **Identify:** Which version introduced the issue
2. **Revert:** `git revert` the merge commit (preserves history)
3. **Document:** Add to `known_issues.md` with the version reference
4. **Fix forward:** Create a new patch version with the fix
5. **Never force-push:** History must be preserved for audit

## Version Compatibility Matrix

| Repo Version | Required Claude Model | Min Context Window | API Version |
|-------------|----------------------|-------------------|-------------|
| 0.1.x | Sonnet 3.5+ | 100K tokens | v1 |
| 0.2.x | Sonnet 3.5+ | 100K tokens | v1 |
| 0.3.x | Sonnet 3.5+ | 150K tokens | v1 |
| 1.0.x | Opus or Sonnet 4 | 200K tokens | v1 |
