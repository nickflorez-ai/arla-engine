# T14: SQS Queue Client

## Overview
AWS SQS client for async Aurora writes. Non-blocking, fire-and-forget with error logging.

## Dependencies
- T03: Logger & Metrics

## Deliverables
- `src/queue/sqs.ts`

---

## Acceptance Criteria

- [ ] Queue messages to SQS
- [ ] Non-blocking (doesn't fail voice response)
- [ ] Configurable via environment
- [ ] Graceful skip if SQS not configured
- [ ] Message includes all audit fields

---

## Test Specifications

### TS-14-01: Queue Message
```typescript
// With SQS configured
const result = await queueAuroraWrite({
  proposalPid: 'TEST-001',
  questionId: 'Q100',
  fieldUpdates: { field: 'value' },
  timestamp: new Date().toISOString(),
});
// Should not throw
```

### TS-14-02: Skip Without Config
```typescript
// Without SQS_QUEUE_URL
process.env.SQS_QUEUE_URL = '';
await expect(queueAuroraWrite(message)).resolves.not.toThrow();
// Should log warning, not error
```

### TS-14-03: Non-Blocking on Error
```typescript
// Force SQS error
await expect(queueAuroraWrite(message)).resolves.not.toThrow();
// Errors logged, not thrown
```

---

## Implementation

See `EXECUTION_PLAN_FOR_REVIEW.md` Step 21 for full implementation.

Key features:
- AWS SDK v3
- Message attributes for routing
- Error caught and logged, never thrown
- Audit fields in message body

---

## Security Considerations

- [ ] AWS credentials from environment/IAM role
- [ ] Message body encrypted in transit
- [ ] No PII in message attributes (use body)
