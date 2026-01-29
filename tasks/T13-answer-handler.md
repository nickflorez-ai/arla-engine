# T13: Answer Handler

## Overview
Process answer submissions: validate, update cache, queue async Aurora write, return new question queue.

## Dependencies
- T12: Queue Builder
- T07: Loan State Cache

## Deliverables
- `src/questions/answer-handler.ts`

---

## Acceptance Criteria

- [ ] Validates question exists
- [ ] Maps answer to access fields
- [ ] Updates Redis cache immediately
- [ ] Queues Aurora write via SQS
- [ ] Re-evaluates and returns new queue
- [ ] Tracks answer latency

---

## Test Specifications

### TS-13-01: Answer Updates Cache
```typescript
const originalState = await getLoanState('TEST-001');
expect(originalState.answered.has('Q100')).toBe(false);

await handleAnswer('TEST-001', { questionId: 'Q100', answer: 'US_CITIZEN' });

const updatedState = await getLoanState('TEST-001');
expect(updatedState.answered.has('Q100')).toBe(true);
expect(updatedState.fields.citizenship_type).toBe('US_CITIZEN');
```

### TS-13-02: Returns New Queue
```typescript
const queue = await handleAnswer('TEST-001', { questionId: 'Q100', answer: 'value' });
expect(queue.loanId).toBe('TEST-001');
expect(queue.queue).toBeDefined();
```

### TS-13-03: Invalid Question Throws
```typescript
await expect(
  handleAnswer('TEST-001', { questionId: 'INVALID', answer: 'value' })
).rejects.toThrow('Question not found');
```

### TS-13-04: Performance
```typescript
const start = performance.now();
await handleAnswer('TEST-001', { questionId: 'Q100', answer: 'value' });
expect(performance.now() - start).toBeLessThan(5);
```

---

## Implementation

See `EXECUTION_PLAN_FOR_REVIEW.md` Step 22 for full implementation.

Key features:
- Single-field questions map directly
- Multi-field questions map by label
- SQS queue for async durability
- Re-evaluation after each answer

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Answer processing | < 5ms |
| Cache update | < 1ms |
| SQS queue | < 1ms (async) |
