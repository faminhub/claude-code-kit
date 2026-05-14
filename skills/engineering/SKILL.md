---
name: engineering
description: Apply senior engineering standards to all implementation. Use when writing new features, refactoring, reviewing architecture, or planning work. Enforces clean architecture, vertical slicing, async patterns, scalability, and production readiness.
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
metadata:
  version: "2.0.0"
  domain: engineering
  role: specialist
  scope: implementation
  output-format: plan+code
  related-skills: qa-review, testing, security-reviewer
---

# Senior Engineer

Principal-level engineer enforcing production-grade standards. No over-engineering. No shortcuts that create debt. Every decision has a reason you can state in one sentence.

## When to Use

- Writing new features or services
- Refactoring or restructuring existing code
- Designing API contracts or data models
- Reviewing architecture decisions
- Planning implementation approach for non-trivial work

## Core Workflow

1. **Understand** — Read the task. **Checkpoint:** Can you state the requirement in one sentence and name the files you'll touch? If not, ask.
2. **Research** — Check if a pattern already exists in the codebase before inventing one. Prefer consistency over novelty.
3. **Plan** — Vertical slice: schema → types → service → controller → (UI if applicable). Never build all DB, then all API, then all UI.
4. **Implement** — Bottom-up. Each layer depends only on the layer below it.
5. **Verify** — Does it satisfy the requirement? Are error paths handled? Is the happy path the only tested path?
6. **Harden** — Logging, timeouts, graceful shutdown, env config. These are not optional.

> **Checkpoint rule:** Before writing code, answer: (a) what does this change, (b) what breaks if it fails, (c) what does the caller expect on error? If you can't answer all three, you don't understand the task yet.

---

## Architecture Standards

### Module Boundaries
- Single responsibility per module/class/function
- Dependency injection over hard imports
- No circular dependencies — dependencies flow one direction only
- Business logic in service layer only — controllers are thin dispatchers, repositories are thin data accessors

### Layering — NestJS / TypeScript
```
Controller  → validates input, calls service, returns response
Service     → business logic, orchestrates data access
Repository / Client → data access only, no business logic
```

### Layering — FastAPI / Python
```
Router      → validates input via Pydantic, calls service, returns response schema
Service     → business logic, orchestrates data access, raises HTTPException
Repository  → SQLAlchemy queries only, no business logic
Depends()   → auth, DB session, logger injection — never inline in route
```

### Vertical Slices (not horizontal phases)
Build one complete feature path at a time — DB + service + API together.

Dependency order (implement bottom-up):
```
Database schema / migration
    └── Repository / DAO
            └── Service (business logic)
                    └── Controller / Route
                            └── DTO / validation
```

---

## Module & File Structure

**TypeScript / NestJS:**
- One `constants.ts` per module — no inline magic values or ad-hoc enums in services
- App-wide shared constants → `src/common/constants.ts`
- Barrel `index.ts` at module root — expose only public API, never internals
- File naming: `<name>.service.ts`, `<name>.controller.ts`, `<name>.dto.ts`, `<name>.guard.ts`

**Python / FastAPI:**
- One `constants.py` per module — status enums, rank dicts, group maps
- App-wide shared constants → `app/common/constants.py`
- Status values always via enum — never raw strings
- File naming: `router.py`, `service.py`, `schemas.py`, `models.py`, `constants.py`, `dependencies.py`

```typescript
// TS BAD — inline magic
if (['completed', 'archived'].includes(status)) { ... }
// TS GOOD
import { STATUS_GROUPS } from './constants';
if (STATUS_GROUPS.done.includes(status)) { ... }
```

```python
# Python BAD — raw string
if task.status == "completed": ...
# Python GOOD
from .constants import TaskStatus
if task.status == TaskStatus.COMPLETED: ...
```

## Comments & Documentation

### Philosophy

Write comments a senior engineer would be grateful to find six months later. Never state what the code does — only why it does it, or what would break if it were changed.

**Tone:** declarative, precise, impersonal. One sentence when possible.

```typescript
// BAD — narrates the obvious
// loop through projects and find matching task
for (const project of projects) { ... }

// BAD — vague
// handle edge case
if (!task) return null;

// GOOD — explains constraint that isn't obvious from code
// Visibility timeout is 30s; delay must stay below that to prevent duplicate SQS delivery
const RETRY_DELAY_MS = 25_000;

// GOOD — explains business rule enforced here
// Lower status rank cannot overwrite higher — prevents completed → pending regression
if (STATUS_RANK[newStatus] <= STATUS_RANK[current.status]) return current;

// GOOD — explains why a seemingly wrong thing is correct
// encodeURIComponent is a no-op for UUIDs but guards against path injection if ID format changes
this.http.get(`/v1/tasks/${encodeURIComponent(taskId)}`);
```

```python
# GOOD — explains compliance requirement, not the line itself
# Soft-delete only — hard deletes prohibited per audit trail policy
task.deleted_at = datetime.utcnow()
```

### API Documentation (mandatory)

Every public endpoint and every DTO/schema field must be documented. These power Swagger UI and client SDK generation — treat them as a public contract.

**NestJS:**
```typescript
// DTO fields — description states the field's purpose and valid range where relevant
@ApiProperty({ description: 'Task UUID', example: 'a1b2c3d4-5678-...' })
taskId: string;

@ApiProperty({ description: 'Priority level (1 = low, 5 = high)', example: 3 })
priority: number;

@ApiProperty({ description: 'Estimated duration in minutes', example: 60 })
estimatedMinutes: number;

// Endpoints — summary is one line; add ApiResponse for non-200 outcomes
@ApiOperation({ summary: 'Mark a task as completed' })
@ApiOkResponse({ description: 'Task status updated successfully' })
@ApiNotFoundResponse({ description: 'Task not found' })
@Post(':taskId/complete')
complete(...) {}
```

**FastAPI:**
```python
# Schema fields — description states purpose and constraints; example is realistic
class CreateTaskRequest(BaseModel):
    project_id: UUID = Field(..., description="Project UUID this task belongs to")
    title: str = Field(..., description="Task title", example="Implement auth middleware")
    priority: int = Field(default=3, description="Priority level (1=low, 5=high)", example=3)
    assignee_id: Optional[UUID] = Field(None, description="User UUID to assign the task to")

# Routes — summary is one line, tags group in Swagger
@router.post(
    "/",
    response_model=TaskResponse,
    summary="Create a new task",
    tags=["tasks"],
    status_code=201,
)
async def create_task(dto: CreateTaskRequest, db: AsyncSession = Depends(get_db)):
    ...
```

### Database Schema Comments

Document fields whose name alone doesn't convey the rule, unit, or constraint.

**Prisma:**
```prisma
model Task {
  id            String    @id @default(uuid())
  title         String
  status        String
  /// Numeric rank of current status. Service layer enforces rank can only increase.
  /// Transitions: pending(0) → assigned(1) → in_progress(2) → completed(3) / cancelled(4)
  statusRank    Int       @default(0)
  priority      Int       @default(3)
  /// Soft-delete timestamp. NULL = active. Hard deletes prohibited (audit requirement).
  deletedAt     DateTime?
}
```

**Alembic migration:**
```python
# Enum values documented inline — readers should not need to check application code
# task_status:
#   pending     — created, awaiting assignment
#   assigned    — assigned to a user, not yet started
#   in_progress — actively being worked on
#   completed   — finished successfully
#   cancelled   — abandoned before completion
op.execute("CREATE TYPE task_status AS ENUM ('pending','assigned','in_progress','completed','cancelled')")
```

### JSDoc / Docstrings

**Skip** on: simple getters, thin wrappers, standard CRUD, anything where name + types tell the full story.

**Write** on: non-obvious failure contracts, retry/fallback behavior, side effects, business constraints.

```typescript
// SKIP — signature is self-documenting
async findOne(taskId: string): Promise<Task>

// WRITE — failure contract and side effect are non-obvious from the signature
/**
 * Submits completion request to the workflow engine with up to 3 retries on transient failure.
 * Does NOT update local task status if all retries fail — caller must handle WorkflowException.
 */
async completeTask(taskId: string): Promise<CompletionResult>
```

```python
# SKIP
async def get_task(task_id: UUID) -> Task: ...

# WRITE — retry contract and partial-failure behavior need explicit documentation
async def complete_task(task_id: UUID) -> CompletionResult:
    """Submit completion to workflow engine with exponential backoff (max 3 retries).

    Does not update local task record on failure.
    Raises WorkflowEngineError if all retries are exhausted.
    """
```

**Rule:** JSDoc — no `@param` / `@returns` type blocks. TypeScript types already carry that. Plain prose only.

---

## Clean Code Rules

- Functions under 30 lines; extract named helpers for anything longer
- No magic numbers — use named constants
- Early returns over nested conditionals — flatten happy path to the end
- No commented-out code in commits
- No dead code: no-op variables, backwards-compat shims, `// removed` comments
- Three similar lines before abstracting — don't generalize until third use case
- Names must be self-documenting: `taskId` not `id`, `isEligibleForCompletion` not `flag`

---

## Error Handling Standards

### Rules
- Never swallow errors silently
- Never throw raw DB errors to clients — wrap in domain exceptions
- Never return generic "something went wrong" — name the problem
- Distinguish: validation error (400), auth error (401/403), not-found (404), conflict (409), upstream failure (502)

### Pattern (NestJS)
```typescript
// BAD — leaks DB internals
throw err; // Prisma error with SQL context

// GOOD — domain exception
throw new NotFoundException(`Task ${taskId} not found`);

// GOOD — upstream error wrapped
throw new BadGatewayException('Workflow engine unavailable');
```

### Exception Filter (NestJS)
For services that proxy upstream APIs, install a global exception filter that:
1. Catches `HttpException` from upstream clients
2. Logs the full error context internally
3. Returns a clean, typed error response to the caller

### Pattern (FastAPI)
```python
# BAD — leaks SQLAlchemy internals
except Exception as e:
    raise HTTPException(status_code=500, detail=str(e))

# GOOD — domain exception, clean message
raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

# GOOD — upstream failure wrapped
raise HTTPException(status_code=502, detail="Workflow engine unavailable")
```

### Global Exception Handler (FastAPI)
Register in `main.py` — catches unhandled exceptions before they reach the client:

```python
# common/exceptions.py
from fastapi import Request
from fastapi.responses import JSONResponse
import structlog

logger = structlog.get_logger()

async def unhandled_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.error(
        "unhandled exception",
        path=request.url.path,
        method=request.method,
        error=str(exc),
        exc_info=True,
    )
    return JSONResponse(status_code=500, content={"detail": "Internal server error"})

# main.py
app.add_exception_handler(Exception, unhandled_exception_handler)
```

---

## Idempotency Patterns

Apply whenever writing records that can be triggered multiple times (queue consumers, webhook handlers, retried API calls).

| Pattern | When to Use |
|---------|-------------|
| Unique constraint on business key | Prevent duplicate inserts on retry |
| Optimistic concurrency (`version` field) | Prevent stale reads overwriting newer state |
| Status rank check before update | Prevent state regression (e.g., `completed` → `pending`) |
| Idempotency key on write endpoints | Client can safely retry without double-applying effects |

```typescript
// Status rank guard — don't let lower status overwrite higher
const STATUS_RANK = { pending: 0, assigned: 1, in_progress: 2, completed: 3 };
if (STATUS_RANK[newStatus] <= STATUS_RANK[current.status]) {
  logger.warn('Ignoring status regression', { current: current.status, attempted: newStatus });
  return current; // idempotent — return existing record
}
```

---

## Observability Standards

### Structured Logging Standard (mandatory for all services)

Every API request emits **one** structured JSON log at request completion. All fields below are required:

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | string | ISO 8601 UTC |
| `level` | string | `INFO` / `WARN` / `ERROR` / `DEBUG` |
| `service` | string | Service name in `kebab-case` |
| `component` | string | Module/class name |
| `message` | string | Human-readable summary |
| `requestId` | string | UUID — from `x-request-id` header or generated |
| `api` | string | Full request URL/path |
| `method` | string | HTTP method |
| `statusCode` | number | HTTP status code |
| `duration` | number | Execution time in ms |
| `request_body` | object | Sanitized body — sensitive fields replaced with `[REDACTED]` |
| `metadata` | array | Step-by-step flow breadcrumbs (see below) |
| `error` | object | `{ message, stackTrace }` — only on failure |

**Level rules:** 5xx → `ERROR`, 4xx → `WARN`, else → `INFO`

**Metadata format** — each step has `step`, `message`, optional `vars`:
```json
{
  "metadata": [
    { "step": "validate-request", "message": "Checking credentials", "vars": { "username": "admin" } },
    { "step": "auth-success", "message": "User authenticated", "vars": { "userId": "user-123" } }
  ]
}
```

**Redaction rule:** Never log `password`, `token`, `api_key`, `secret`. Replace value with `[REDACTED]`.

**Example success log:**
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "INFO",
  "service": "task-service",
  "component": "TasksController",
  "message": "POST /v1/tasks/123/complete",
  "requestId": "9f6a45b3-1234",
  "api": "/v1/tasks/123/complete",
  "method": "POST",
  "statusCode": 200,
  "duration": 87,
  "request_body": { "title": "Implement auth middleware", "token": "[REDACTED]" },
  "metadata": [
    { "step": "findOne", "message": "fetching task", "vars": { "taskId": "123" } },
    { "step": "findOne", "message": "task fetched", "vars": { "taskStatus": "in_progress" } },
    { "step": "complete", "message": "task marked completed", "vars": { "taskId": "123" } }
  ]
}
```

### Log Levels
| Level | When |
|-------|------|
| DEBUG | Dev-only detail (not in prod) |
| INFO  | Business events: created, updated, completed |
| WARN  | Recoverable: 4xx, retry, state regression blocked |
| ERROR | 5xx, upstream down, data inconsistency — needs human action |

### NestJS Implementation (canonical pattern)

Use `AsyncLocalStorage` + `RequestContextService` + `LoggingInterceptor`:

- `RequestContextService` — holds `requestId` + `metadata[]` per async request scope
- `LoggingInterceptor` — wraps every HTTP request, emits single log on completion
- Services call `ctx.addMetadata({ step, message, vars })` to append breadcrumbs
- `redact()` utility strips sensitive keys before logging request body

```typescript
// In any service — add step breadcrumb
this.ctx.addMetadata({ step: 'create', message: 'creating task', vars: { projectId } });

// BAD
console.log('task created');
```

### FastAPI Implementation (canonical pattern)

Single log per request via middleware. Services append to `request.state.metadata`.

```python
# common/middleware.py
import uuid, time, traceback, json
from fastapi import Request
from .redact import redact

SENSITIVE_KEYS = {"password", "token", "secret", "authorization", "api_key",
                  "apikey", "access_token", "refresh_token"}

async def logging_middleware(request: Request, call_next):
    request_id = request.headers.get("x-request-id", str(uuid.uuid4()))
    start = time.monotonic()

    body = {}
    try:
        body = await request.json()
    except Exception:
        pass

    request.state.log_data = {
        "requestId": request_id,
        "method": request.method,
        "api": str(request.url),
        "request_body": redact(body),
        "metadata": [],
    }

    try:
        response = await call_next(request)
        duration = int((time.monotonic() - start) * 1000)
        status = response.status_code
        level = "ERROR" if status >= 500 else "WARN" if status >= 400 else "INFO"
        log = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level": level,
            "service": settings.SERVICE_NAME,   # kebab-case e.g. "task-service"
            "component": "middleware",
            "message": "Request completed successfully" if status < 400 else "Request failed",
            "statusCode": status,
            "duration": duration,
            **request.state.log_data,
        }
        print(json.dumps(log), flush=True)
        response.headers["x-request-id"] = request_id
        return response
    except Exception as exc:
        duration = int((time.monotonic() - start) * 1000)
        log = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level": "ERROR",
            "service": settings.SERVICE_NAME,
            "component": "middleware",
            "message": "Request failed",
            "statusCode": 500,
            "duration": duration,
            "error": {
                "message": str(exc),
                "stackTrace": traceback.format_exc(limit=5).split("\n"),
            },
            **request.state.log_data,
        }
        print(json.dumps(log), flush=True)
        raise
```

```python
# common/redact.py
SENSITIVE_KEYS = {"password", "token", "secret", "authorization", "api_key",
                  "apikey", "access_token", "refresh_token", "private_key"}

def redact(obj: object) -> object:
    if not obj or not isinstance(obj, dict):
        return obj
    return {
        k: "[REDACTED]" if k.lower() in SENSITIVE_KEYS else redact(v)
        for k, v in obj.items()
    }
```

```python
# tasks/service.py — append metadata breadcrumbs
from fastapi import Request

class TaskService:
    async def create_task(self, request: Request, dto: CreateTaskRequest) -> TaskResponse:
        request.state.log_data["metadata"].append(
            {"step": "create", "message": "creating task", "vars": {"project_id": str(dto.project_id)}}
        )
        result = await self.repository.create(dto)
        request.state.log_data["metadata"].append(
            {"step": "create", "message": "task created", "vars": {"task_id": str(result.id)}}
        )
        return result
```

### External Call Logging (FastAPI)
```python
import time, httpx
from fastapi import Request

async def notify_assignee(self, request: Request, dto: dict) -> dict:
    start = time.monotonic()
    try:
        response = await self.client.post("/v1/notifications", json=dto)
        response.raise_for_status()
        request.state.log_data["metadata"].append({
            "step": "notify_assignee",
            "message": "upstream POST /v1/notifications",
            "vars": {"httpStatus": response.status_code, "duration": int((time.monotonic() - start) * 1000)},
        })
        return response.json()
    except httpx.HTTPStatusError as e:
        request.state.log_data["metadata"].append({
            "step": "notify_assignee",
            "message": "upstream POST /v1/notifications failed",
            "vars": {"httpStatus": e.response.status_code, "duration": int((time.monotonic() - start) * 1000)},
        })
        raise HTTPException(status_code=502, detail="Notification service unavailable")
```

---

## External Dependency Rules

Every call to an external service (HTTP, DB, SQS, cache) must have:

| Requirement | Why |
|-------------|-----|
| Timeout | Prevent indefinite hangs |
| Error handling | Don't let upstream failures become 500s without context |
| Request ID propagation | Trace requests across service boundaries |
| Duration logging | Identify latency regressions |

**NestJS — axios:**
```typescript
// Timeout on axios
this.http = axios.create({ baseURL, timeout: 10_000 });

// Request ID propagation via interceptor
this.http.interceptors.request.use((config) => {
  const requestId = this.ctx.getRequestId();
  if (requestId) config.headers['x-request-id'] = requestId;
  return config;
});
```

**FastAPI — httpx:**
```python
# common/http_client.py
import httpx
from contextvars import ContextVar

request_id_var: ContextVar[str] = ContextVar("request_id", default="")

def make_client(base_url: str, token: str) -> httpx.AsyncClient:
    return httpx.AsyncClient(
        base_url=base_url,
        headers={"Authorization": f"Bearer {token}"},
        timeout=httpx.Timeout(10.0),  # 10s total; set connect/read separately if needed
    )

# Inject request_id on every outbound request via event hook
async def propagate_request_id(request: httpx.Request) -> None:
    rid = request_id_var.get("")
    if rid:
        request.headers["x-request-id"] = rid

client = make_client(base_url, token)
client.event_hooks["request"] = [propagate_request_id]
```

Retry only transient errors (5xx, network timeout). Never retry 4xx — they are permanent.

---

## Database Patterns

- Use transactions where multiple writes must be atomic
- Index foreign keys and columns used in `WHERE` clauses
- Soft delete over hard delete when audit trail matters
- Never `SELECT *` in application code — select only needed columns
- Paginate all list queries — no unbounded fetches

**NestJS — Prisma:**
```typescript
// BAD — unbounded
prisma.task.findMany({ where: { projectId } });

// GOOD — paginated
prisma.task.findMany({ where: { projectId }, skip: (page - 1) * limit, take: limit });
```

**FastAPI — SQLAlchemy AsyncSession:**

```python
# common/database.py
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

engine = create_async_engine(settings.DATABASE_URL, pool_pre_ping=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncSession:  # use as Depends(get_db)
    async with AsyncSessionLocal() as session:
        yield session
```

```python
# tasks/router.py — session injected via Depends
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from common.database import get_db

@router.get("/{task_id}", response_model=TaskResponse)
async def get_task(task_id: UUID, db: AsyncSession = Depends(get_db)):
    return await task_service.find_one(db, task_id)
```

```python
# tasks/service.py — transaction pattern
async def create_task(db: AsyncSession, dto: CreateTaskRequest) -> Task:
    async with db.begin():           # atomic — rolls back on exception
        task = Task(**dto.model_dump())
        db.add(task)
        await db.flush()             # get generated ID without committing
        # ... other writes in same transaction
    return task

# Paginated query — never unbounded
async def list_tasks(db: AsyncSession, page: int, limit: int) -> list[Task]:
    result = await db.execute(
        select(Task).offset((page - 1) * limit).limit(limit)
    )
    return result.scalars().all()
```

---

## Performance Rules

- No N+1 queries — batch with `findMany({ where: { id: { in: ids } } })`
- `Promise.all` for independent async operations — don't await sequentially
- Background jobs for anything over 200ms in a request path
- Measure before optimizing — no premature optimization

```typescript
// BAD — N+1
for (const project of projects) {
  const tasks = await prisma.task.findFirst({ where: { projectId: project.id } });
}

// GOOD — batch
const tasks = await prisma.task.findMany({
  where: { projectId: { in: projects.map(p => p.id) } },
});
```

---

## Production Readiness Checklist

Before any PR:

- [ ] Structured logging with request ID on all paths
- [ ] Timeouts on all external HTTP/DB calls
- [ ] Error responses don't leak stack traces or internal paths
- [ ] Environment config via env vars — no hardcoded values
- [ ] Secrets not in code, logs, or version control
- [ ] Auth/authorization checked at every protected endpoint
- [ ] Graceful shutdown: NestJS `enableShutdownHooks()` / FastAPI `lifespan` context manager
- [ ] List endpoints paginated
- [ ] Transactions used where multi-write atomicity is needed
- [ ] No `console.log` / `print()` in production paths — use structured logger

---

## Change Sizing

```
~100 lines changed   → Ideal. Reviewable in one sitting.
~300 lines changed   → Acceptable if single logical change.
~1000 lines changed  → Too large. Split it.
```

Separate refactoring from feature work — submit as separate PRs.

---

## MUST DO

- State the requirement in one sentence before writing code
- Implement bottom-up: schema → service → controller
- Handle error paths explicitly — not just the happy path
- Log every external call with status and duration
- Paginate every list endpoint
- Use transactions where multiple writes must be atomic

## Security Patterns

### Token / API Key Comparison — ALWAYS constant-time

**NestJS (TypeScript):**
```typescript
import { timingSafeEqual } from 'crypto';

const a = Buffer.from(incomingToken);
const b = Buffer.from(validToken);
if (a.length !== b.length || !timingSafeEqual(a, b)) {
  throw new UnauthorizedException('Invalid API key');
}
```

**FastAPI (Python):**
```python
import hmac

def verify_api_key(x_api_key: str = Header(...)):
    valid = settings.API_KEY
    if not hmac.compare_digest(x_api_key.encode(), valid.encode()):
        raise HTTPException(status_code=401, detail="Invalid API key")
```

**Why:** String equality (`==`, `!==`) short-circuits on first mismatch — timing difference leaks how many characters matched (timing attack). `timingSafeEqual` / `hmac.compare_digest` always take the same time regardless of match position.

### Auth Rules
- Token read from env via `ConfigService` / `settings` — never hardcoded
- Never log token value — redact with `[REDACTED]`
- 401 = unauthenticated (no valid token), 403 = authenticated but unauthorized (wrong role/scope)
- Guard/dependency applied globally — opt-out with `@Public()` decorator, not opt-in per route

---

## MUST NOT DO

- Business logic in controllers or repositories
- Swallow errors silently (`catch {}` or empty catch)
- Return raw DB/upstream errors to clients
- Hardcode URLs, secrets, or environment-specific values
- Create abstractions before the third use case
- Add feature flags, backwards-compat shims, or `// removed` comments

---

## Reference

| Concern | Standard |
|---------|----------|
| Function length | ≤ 30 lines |
| File length | ≤ 800 lines |
| External call timeout | 10 000 ms default |
| List endpoint default page size | 20 |
| Max page size | 100 |
| Log sensitive fields | Never (tokens, passwords, secrets) |
| Status updates | Rank-guarded — lower rank cannot overwrite higher |
