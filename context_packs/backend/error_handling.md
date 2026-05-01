# Error Handling

## Error Classification

| Category | Retryable | Alert | Example |
|----------|-----------|-------|---------|
| Validation | No | No | Invalid input, missing field |
| Auth | No | Yes (if repeated) | Invalid token, insufficient perms |
| Not Found | No | No | Resource doesn't exist |
| Conflict | No | No | Duplicate, version mismatch |
| External | Yes | Yes (if persistent) | LLM timeout, DB connection lost |
| Internal | Maybe | Yes | Unexpected null, logic bug |
| Resource | Yes | Yes | Out of memory, disk full |

## Error Hierarchy
```typescript
// Base error
export abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly code: string;
  readonly isOperational: boolean = true;
  readonly timestamp: Date = new Date();
  
  constructor(message: string, public readonly details?: Record<string, any>) {
    super(message);
  }
}

// Client errors (4xx)
export class ValidationError extends AppError { statusCode = 400; code = 'VALIDATION_ERROR'; }
export class UnauthorizedError extends AppError { statusCode = 401; code = 'UNAUTHORIZED'; }
export class ForbiddenError extends AppError { statusCode = 403; code = 'FORBIDDEN'; }
export class NotFoundError extends AppError { statusCode = 404; code = 'NOT_FOUND'; }
export class ConflictError extends AppError { statusCode = 409; code = 'CONFLICT'; }

// Server errors (5xx)
export class InternalError extends AppError { statusCode = 500; code = 'INTERNAL_ERROR'; isOperational = false; }
export class ExternalServiceError extends AppError { statusCode = 502; code = 'EXTERNAL_SERVICE_ERROR'; }
export class TimeoutError extends AppError { statusCode = 504; code = 'TIMEOUT'; }
```

## Global Error Handler
```typescript
function globalErrorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  // Structured logging
  const logPayload = {
    error: err.message,
    code: err instanceof AppError ? err.code : 'UNKNOWN',
    stack: err.stack,
    requestId: req.id,
    userId: req.user?.id,
    method: req.method,
    path: req.path,
  };

  if (err instanceof AppError && err.isOperational) {
    logger.warn(logPayload);
    return res.status(err.statusCode).json({
      error: { code: err.code, message: err.message, details: err.details, requestId: req.id }
    });
  }

  // Unexpected errors - log as critical, return generic message
  logger.error(logPayload);
  return res.status(500).json({
    error: { code: 'INTERNAL_ERROR', message: 'An unexpected error occurred', requestId: req.id }
  });
}
```

## Retry Strategy
```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries: number;
    backoff: number[];
    retryableErrors: string[];
    onRetry?: (attempt: number, error: Error) => void;
  }
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 0; attempt <= options.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      
      const isRetryable = options.retryableErrors.some(code => 
        error instanceof AppError && error.code === code
      );
      
      if (!isRetryable || attempt === options.maxRetries) throw error;
      
      const delay = options.backoff[attempt] ?? options.backoff[options.backoff.length - 1];
      options.onRetry?.(attempt + 1, error as Error);
      await sleep(delay * 1000);
    }
  }
  
  throw lastError!;
}
```

## Circuit Breaker (External Services)
```typescript
class CircuitBreaker {
  private failures = 0;
  private lastFailure?: Date;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private readonly threshold: number = 5,
    private readonly resetTimeout: number = 60000
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailure!.getTime() > this.resetTimeout) {
        this.state = 'half-open';
      } else {
        throw new ExternalServiceError('Circuit breaker is open');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failures = 0;
    this.state = 'closed';
  }

  private onFailure() {
    this.failures++;
    this.lastFailure = new Date();
    if (this.failures >= this.threshold) {
      this.state = 'open';
    }
  }
}
```

## Logging Standards
```typescript
// Structured JSON logging
const logger = createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: 'json',
  defaultMeta: { service: 'ai-factory', version: process.env.APP_VERSION }
});

// Log levels usage:
// error: Unrecoverable failures, data loss risk
// warn: Recoverable issues, degraded performance
// info: Business events, state transitions
// debug: Detailed flow, variable values (dev only)
```
