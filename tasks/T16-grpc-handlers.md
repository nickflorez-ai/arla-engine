# T16: gRPC Handlers

## Overview
Request handlers for GetQuestions, SubmitAnswer, GetLoanState with proper error handling.

## Dependencies
- T15: gRPC Server
- T12: Queue Builder
- T13: Answer Handler

## Deliverables
- `src/grpc/handlers.ts`

---

## Acceptance Criteria

- [ ] GetQuestions returns queue for proposal
- [ ] SubmitAnswer processes and returns new queue
- [ ] GetLoanState returns cached state
- [ ] Validates required fields
- [ ] Returns proper gRPC status codes
- [ ] Logs request latency

---

## Test Specifications

### TS-16-01: GetQuestions Success
```typescript
const response = await client.getQuestions({ proposalPid: 'TEST-001' });
expect(response.loanId).toBe('TEST-001');
expect(Array.isArray(response.queue)).toBe(true);
```

### TS-16-02: GetQuestions Missing PID
```typescript
await expect(
  client.getQuestions({ proposalPid: '' })
).rejects.toMatchObject({ code: grpc.status.INVALID_ARGUMENT });
```

### TS-16-03: SubmitAnswer Success
```typescript
const response = await client.submitAnswer({
  proposalPid: 'TEST-001',
  questionId: 'Q100',
  answer: '"US_CITIZEN"',
});
expect(response.loanId).toBe('TEST-001');
```

### TS-16-04: GetLoanState Success
```typescript
const response = await client.getLoanState({ proposalPid: 'TEST-001' });
expect(response.proposalPid).toBe('TEST-001');
expect(response.fieldsJson).toBeDefined();
```

### TS-16-05: Latency Tracking
```typescript
// After request, check metrics
const metrics = getMetrics('grpc_get_questions');
expect(metrics.count).toBeGreaterThan(0);
```

---

## Implementation

See `EXECUTION_PLAN_FOR_REVIEW.md` Step 25 for full implementation.

Key features:
- Validation before processing
- JSON parsing for answer field
- Proper gRPC status codes
- Latency measurement wrapper

---

## Error Handling

| Error | gRPC Status |
|-------|-------------|
| Missing required field | INVALID_ARGUMENT |
| Proposal not found | NOT_FOUND |
| Internal error | INTERNAL |
