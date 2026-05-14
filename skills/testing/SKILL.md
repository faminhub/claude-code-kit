---
name: testing
description: Drive development with tests. Use when implementing any logic, fixing bugs, or changing behavior. Enforces TDD red-green-refactor cycle, bug reproduction pattern, and comprehensive coverage.
---

# Testing Skill

## The TDD Cycle

```
    RED                GREEN              REFACTOR
 Write a test    Write minimal code    Clean up the
 that fails  ──→  to make it pass  ──→  implementation  ──→  (repeat)
```

### Step 1: RED — Write Failing Test First
Test must fail. A test that passes immediately proves nothing.

```typescript
// Write test BEFORE implementation
describe('TaskService', () => {
  it('creates a task with title and default status', async () => {
    const task = await taskService.createTask({ title: 'Buy groceries' });
    expect(task.id).toBeDefined();
    expect(task.status).toBe('pending');
  });
});
```

### Step 2: GREEN — Minimal Implementation
Write minimum code to make test pass. Do not over-engineer.

### Step 3: REFACTOR — Clean Up
With tests green: extract shared logic, improve naming, remove duplication.
Run tests after every refactor step.

## Bug Fix Pattern (Prove-It)

Do NOT fix a bug before reproducing it with a test.

```
Bug report arrives
    └── Write test that demonstrates the bug (must FAIL)
            └── Implement the fix
                    └── Test PASSES (fix proven)
                            └── Run full suite (no regressions)
```

## Unit Tests
- Test each function in isolation with mocked dependencies
- One assertion concept per test
- Cover: happy path, empty input, null/undefined, boundary values
- Test error throwing explicitly — assert error type and message

## Integration Tests
- Test module interactions with real dependencies where feasible
- DB tests use real DB (not mocks) — mocks hide migration failures
- API tests hit actual routes end-to-end
- Seed minimal test data, clean up after each test

## Failure State Tests
- What happens when external service is down?
- What happens on DB timeout?
- What happens with malformed input that passes validation?
- What happens on partial failure in a batch operation?

## Edge Case Coverage
- Empty arrays/strings/objects
- Max length inputs
- Special characters and Unicode
- Concurrent calls to same resource
- Idempotency: calling twice must not double-apply effects

## Test Structure

```typescript
describe('[module/function name]', () => {
  describe('[scenario]', () => {
    it('[expected behavior]', () => {
      // Arrange
      // Act
      // Assert
    })
  })
})
```

## Coverage Targets
- Statements: 80%+
- Branches: 80%+
- Critical paths (auth, payments, data mutations): 95%+
- Test observable behavior, not implementation details
- Tests are proof — "seems right" is not done
