# Authentication & Security

## Authentication Methods

### JWT Tokens (Primary)
```typescript
// Access token: short-lived (15 min)
interface AccessTokenPayload {
  sub: string;        // user ID
  role: UserRole;     // admin | team_lead | developer | viewer
  orgId: string;      // organization ID
  permissions: string[];  // fine-grained permissions
  iat: number;
  exp: number;
}

// Refresh token: long-lived (7 days), stored in httpOnly cookie
interface RefreshTokenPayload {
  sub: string;
  tokenFamily: string;  // For rotation detection
  iat: number;
  exp: number;
}
```

### API Keys (Service-to-Service)
```typescript
// Format: factory_sk_live_{random_32_chars}
// Stored hashed in DB, never in plaintext
interface ApiKey {
  id: string;
  prefix: string;       // First 8 chars for identification
  hash: string;         // bcrypt hash of full key
  name: string;         // Human-readable name
  scopes: string[];     // Allowed operations
  rateLimit: number;    // Requests per minute
  expiresAt?: Date;
  lastUsedAt?: Date;
}
```

## Authorization (RBAC + ABAC)

### Roles
| Role | Description | Scope |
|------|-------------|-------|
| `admin` | Full system access | Global |
| `team_lead` | Manage team projects, approve changes | Team |
| `developer` | Create/edit within assigned projects | Project |
| `viewer` | Read-only access | Project |

### Permission Check Pattern
```typescript
// Middleware
function requirePermission(...permissions: Permission[]) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const user = req.user;
    
    // Check role-based permissions
    const hasRole = permissions.some(p => user.permissions.includes(p));
    
    // Check resource-based access (ABAC)
    if (!hasRole) {
      const resource = await getResource(req);
      const hasAccess = await checkResourceAccess(user, resource, permissions);
      if (!hasAccess) {
        throw new ForbiddenError();
      }
    }
    
    next();
  };
}

// Usage
router.post('/executions', 
  requirePermission('execution.create'),
  executionController.create
);
```

### Resource-Level Access
```typescript
// Projects have team members with specific roles
interface ProjectMember {
  userId: string;
  projectId: string;
  role: 'owner' | 'maintainer' | 'contributor' | 'viewer';
  permissions: string[];  // Override defaults
}
```

## Security Headers
```typescript
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],  // For SSR
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "wss:"],
    }
  },
  hsts: { maxAge: 31536000, includeSubDomains: true },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
}));
```

## Input Sanitization
```typescript
// All user inputs sanitized before processing
function sanitize(input: string): string {
  return input
    .trim()
    .replace(/[<>]/g, '')  // Strip HTML tags
    .slice(0, MAX_INPUT_LENGTH);
}

// File upload validation
const ALLOWED_MIME_TYPES = ['text/markdown', 'application/json', 'text/yaml'];
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
```

## Secrets Management
```typescript
// All secrets from environment variables
// Never in code, never in config files
const config = {
  jwtSecret: process.env.JWT_SECRET!,
  dbUrl: process.env.DATABASE_URL!,
  redisUrl: process.env.REDIS_URL!,
  llmApiKey: process.env.LLM_API_KEY!,
};

// Validate all required secrets at startup
function validateSecrets() {
  const required = ['JWT_SECRET', 'DATABASE_URL', 'REDIS_URL', 'LLM_API_KEY'];
  const missing = required.filter(k => !process.env[k]);
  if (missing.length > 0) {
    throw new Error(`Missing required secrets: ${missing.join(', ')}`);
  }
}
```

## Audit Logging
```typescript
// Every state-changing operation is logged
async function auditLog(event: AuditEvent) {
  await db.insert(auditLogs).values({
    action: event.action,
    resourceType: event.resourceType,
    resourceId: event.resourceId,
    actorId: event.actor.id,
    actorRole: event.actor.role,
    oldValues: event.oldValues,
    newValues: event.newValues,
    ipAddress: event.ipAddress,
    userAgent: event.userAgent,
    timestamp: new Date()
  });
}
```

## Rate Limiting
```typescript
// Tiered rate limiting
const rateLimits = {
  admin: { windowMs: 60000, max: 500 },
  team_lead: { windowMs: 60000, max: 200 },
  developer: { windowMs: 60000, max: 100 },
  viewer: { windowMs: 60000, max: 50 },
  api_key: { windowMs: 60000, max: 1000 },
};
```
