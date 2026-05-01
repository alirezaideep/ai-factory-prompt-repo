# Prompt Factory — API Endpoints

## Base Path: `/api/v1/repos`

### Repository Management

#### POST /api/v1/repos
Create a new prompt repository from a product definition.

**Request:**
```json
{
  "projectId": "proj-uuid",
  "productDefinition": "A comprehensive order management system for a glass manufacturing company...",
  "template": "standard",
  "options": {
    "generateShared": true,
    "generateFeatures": true,
    "generateContextPacks": true,
    "initVersion": "0.1.0"
  }
}
```

**Response:**
```json
{
  "repoId": "repo-uuid",
  "status": "generating",
  "estimatedFiles": 45,
  "gitUrl": "git@internal:repos/proj-uuid.git"
}
```

#### GET /api/v1/repos/{repoId}
Get repository metadata and current version.

#### GET /api/v1/repos/{repoId}/tree
Get file tree of the repository.

**Response:**
```json
{
  "version": "0.3.0",
  "files": [
    { "path": "shared/backend.md", "size": 4200, "lastModified": "2025-02-20" },
    { "path": "features/orders/feature_brief.md", "size": 2800, "lastModified": "2025-02-18" }
  ],
  "totalFiles": 62
}
```

### File Operations

#### GET /api/v1/repos/{repoId}/files/{path}
Read a specific file from the repository.

#### PUT /api/v1/repos/{repoId}/files/{path}
Update a file (creates new version).

**Request:**
```json
{
  "content": "# Updated content...",
  "commitMessage": "feat(orders): add priority field to order schema",
  "author": "planning-service"
}
```

#### POST /api/v1/repos/{repoId}/delta
Apply a delta update (partial change without full rewrite).

**Request:**
```json
{
  "deltas": [
    {
      "file": "features/orders/back/data_model.md",
      "operation": "add",
      "anchor": "## Columns",
      "position": "after",
      "content": "| priority | enum('low','normal','high','urgent') | NOT NULL | DEFAULT 'normal' |"
    },
    {
      "file": "features/orders/back/api_endpoints.md",
      "operation": "modify",
      "find": ""status": "string"",
      "replace": ""status": "string",\n    "priority": "string""
    }
  ],
  "commitMessage": "feat(orders): add priority field",
  "bumpVersion": "patch"
}
```

### Version Management

#### GET /api/v1/repos/{repoId}/versions
List all versions with changelog.

#### POST /api/v1/repos/{repoId}/versions
Create a new version (bump).

**Request:**
```json
{
  "type": "minor",
  "summary": "Added notification module with email and SMS channels",
  "breaking": false
}
```

#### GET /api/v1/repos/{repoId}/diff/{fromVersion}/{toVersion}
Get diff between two versions.

### Context Loading

#### POST /api/v1/repos/{repoId}/context
Get the optimal set of files for a specific task (implements loading formula).

**Request:**
```json
{
  "taskType": "add_api_endpoint",
  "module": "orders",
  "additionalContext": ["shared/backend.md"]
}
```

**Response:**
```json
{
  "files": [
    { "path": "shared/backend.md", "reason": "API conventions" },
    { "path": "features/orders/feature_brief.md", "reason": "Module overview" },
    { "path": "features/orders/back/api_endpoints.md", "reason": "Existing endpoints" },
    { "path": "features/orders/back/data_model.md", "reason": "Data schema" },
    { "path": "context_packs/backend/api_conventions.md", "reason": "Standards" }
  ],
  "totalTokens": 18500,
  "loadingFormula": "backend_api_change"
}
```
