# Coding Style

## Immutability (CRITICAL)

ALWAYS create new objects, NEVER mutate existing ones:

```
// Pseudocode
WRONG:  modify(original, field, value) → changes original in-place
CORRECT: update(original, field, value) → returns new copy with change
```

Rationale: Immutable data prevents hidden side effects, makes debugging easier, and enables safe concurrency.

## Core Principles

### KISS (Keep It Simple)

- Prefer the simplest solution that actually works
- Avoid premature optimization
- Optimize for clarity over cleverness

### DRY (Don't Repeat Yourself)

- Extract repeated logic into shared functions or utilities
- Avoid copy-paste implementation drift
- Introduce abstractions when repetition is real, not speculative

### YAGNI (You Aren't Gonna Need It)

- Do not build features or abstractions before they are needed
- Avoid speculative generality
- Start simple, then refactor when the pressure is real

## File Organization

MANY SMALL FILES > FEW LARGE FILES:
- High cohesion, low coupling
- 200-400 lines typical, 800 max
- Extract utilities from large modules
- Organize by feature/domain, not by type

## Module Structure — NestJS / TypeScript

Every feature module must have consistent file layout:

```
src/
  payments/
    payments.controller.ts   ← thin dispatcher only
    payments.service.ts      ← business logic
    payments.module.ts       ← DI wiring
    dto/
      payments.dto.ts        ← all DTOs for this module
    constants.ts             ← enums, status maps, rank tables for this module
  common/
    constants.ts             ← app-wide constants (shared across modules)
    types.ts                 ← shared TypeScript types/interfaces
```

Rules:
- One `constants.ts` per module — never inline magic values in service/controller files
- App-wide shared constants live in `src/common/constants.ts`
- All DTOs for a module in one `dto/<module>.dto.ts` file (split only if >150 lines)
- No ad-hoc inline enums (`type Status = 'a' | 'b'`) in service files — put in `constants.ts`

### File Naming — TypeScript / NestJS

| File type | Convention | Example |
|-----------|------------|---------|
| Service | `<name>.service.ts` | `payments.service.ts` |
| Controller | `<name>.controller.ts` | `payments.controller.ts` |
| Module | `<name>.module.ts` | `payments.module.ts` |
| DTO | `<name>.dto.ts` | `payments.dto.ts` |
| Guard | `<name>.guard.ts` | `api-key.guard.ts` |
| Interceptor | `<name>.interceptor.ts` | `logging.interceptor.ts` |
| Filter | `<name>.filter.ts` | `http-exception.filter.ts` |
| Constants | `constants.ts` | per module or `common/constants.ts` |
| Types | `types.ts` | per module or `common/types.ts` |
| Barrel | `index.ts` | module root only, exports public API |

### Barrel Exports (`index.ts`)

- Each module exposes public API via `index.ts` — importers never reach into internals
- `index.ts` re-exports only what other modules need — not everything

```typescript
// payments/index.ts
export { PaymentsModule } from './payments.module';
export { PaymentsService } from './payments.service';
// DTOs, guards, internals: NOT exported unless needed cross-module
```

### Constants File — TypeScript

```typescript
// payments/constants.ts
export const STATUS_RANK: Record<string, number> = {
  pending: 0,
  authorized: 1,
  captured: 2,
  refunded: 3,
  voided: 4,
};

export const STATUS_GROUPS = {
  active: ['pending', 'authorized'],
  completed: ['captured'],
  failed: ['voided', 'expired'],
} as const;

export const DEFAULT_PAGE_SIZE = 20;
export const MAX_PAGE_SIZE = 100;
```

---

## Module Structure — FastAPI / Python

Every feature module must have consistent file layout:

```
app/
  payments/
    router.py        ← thin dispatcher only (@router decorators)
    service.py       ← business logic
    schemas.py       ← Pydantic request/response models
    models.py        ← SQLAlchemy ORM models (if DB-backed)
    constants.py     ← Enum classes, status maps, rank dicts
    dependencies.py  ← FastAPI Depends() factories for this domain
  common/
    constants.py     ← app-wide constants
    dependencies.py  ← shared Depends() (auth, DB session, logger)
    exceptions.py    ← custom HTTPException subclasses
    schemas.py       ← shared Pydantic base models
  main.py            ← FastAPI app factory, router registration
```

Rules:
- One `constants.py` per module — never inline magic strings/dicts in service/router files
- App-wide shared constants → `app/common/constants.py`
- All Pydantic models for a module in `schemas.py` (split only if >150 lines)
- Status strings always reference `constants.py` — never raw string literals
- `Depends()` factories in `dependencies.py` — never inline in route signatures

### File Naming — Python / FastAPI

| File type | Convention | Example |
|-----------|------------|---------|
| Router | `router.py` | `payments/router.py` |
| Service | `service.py` | `payments/service.py` |
| Pydantic schemas | `schemas.py` | `payments/schemas.py` |
| ORM models | `models.py` | `payments/models.py` |
| Constants / Enums | `constants.py` | `payments/constants.py` |
| Dependencies | `dependencies.py` | `payments/dependencies.py` |
| Exceptions | `exceptions.py` | `common/exceptions.py` |
| App factory | `main.py` | `app/main.py` |

All Python files: `snake_case`. Classes: `PascalCase`. Constants: `UPPER_SNAKE_CASE`.

### Constants File — Python

```python
# payments/constants.py
from enum import Enum

class PaymentStatus(str, Enum):
    PENDING = "pending"
    AUTHORIZED = "authorized"
    CAPTURED = "captured"
    REFUNDED = "refunded"
    VOIDED = "voided"

STATUS_RANK: dict[str, int] = {
    PaymentStatus.PENDING: 0,
    PaymentStatus.AUTHORIZED: 1,
    PaymentStatus.CAPTURED: 2,
    PaymentStatus.REFUNDED: 3,
    PaymentStatus.VOIDED: 4,
}

STATUS_GROUPS: dict[str, list[str]] = {
    "active": [PaymentStatus.PENDING, PaymentStatus.AUTHORIZED],
    "completed": [PaymentStatus.CAPTURED],
    "failed": [PaymentStatus.VOIDED],
}

DEFAULT_PAGE_SIZE = 20
MAX_PAGE_SIZE = 100
```

Never compare raw strings to status values — always use the `PaymentStatus` enum.

## Error Handling

ALWAYS handle errors comprehensively:
- Handle errors explicitly at every level
- Provide user-friendly error messages in UI-facing code
- Log detailed error context on the server side
- Never silently swallow errors

## Input Validation

ALWAYS validate at system boundaries:
- Validate all user input before processing
- Use schema-based validation where available
- Fail fast with clear error messages
- Never trust external data (API responses, user input, file content)

## Naming Conventions

### TypeScript / NestJS
- Variables and functions: `camelCase`
- Booleans: `is`, `has`, `should`, `can` prefixes
- Interfaces, types, classes, components: `PascalCase`
- Constants: `UPPER_SNAKE_CASE`
- Custom hooks: `camelCase` with `use` prefix
- Files: `kebab-case` with role suffix (`payments.service.ts`)

### Python / FastAPI
- Variables, functions, files, modules: `snake_case`
- Classes (Pydantic models, ORM models, Enums): `PascalCase`
- Constants: `UPPER_SNAKE_CASE`
- Booleans: `is_`, `has_`, `should_`, `can_` prefixes
- Files: `snake_case` with role name (`payment_service.py` or `service.py` inside feature folder)

## Code Smells to Avoid

### Deep Nesting

Prefer early returns over nested conditionals once the logic starts stacking.

### Magic Numbers

Use named constants for meaningful thresholds, delays, and limits.

### Long Functions

Split large functions into focused pieces with clear responsibilities.

## Code Quality Checklist

Before marking work complete:
- [ ] Code is readable and well-named
- [ ] Functions are small (<50 lines)
- [ ] Files are focused (<800 lines)
- [ ] No deep nesting (>4 levels)
- [ ] Proper error handling
- [ ] No hardcoded values (use constants or config)
- [ ] No mutation (immutable patterns used)
