# T18: Integration Tests

## Overview
End-to-end integration tests covering full request flows through the gRPC API.

## Dependencies
- T17: Main Entry Point

## Deliverables
- `tests/integration/grpc.test.ts`
- `tests/integration/fixtures/` (test data)
- `tests/integration/setup.ts` (test harness)

---

## Acceptance Criteria

- [ ] Tests against real gRPC server
- [ ] Uses test database/cache
- [ ] Seeds test data before each test
- [ ] Cleans up after tests
- [ ] Tests full request/response cycle

---

## Test Specifications

### TS-18-01: Full GetQuestions Flow
```typescript
describe('GetQuestions', () => {
  beforeAll(async () => {
    await seedTestData();
    await startTestServer();
  });

  afterAll(async () => {
    await stopTestServer();
    await cleanupTestData();
  });

  it('returns questions for valid proposal', async () => {
    const response = await grpcClient.getQuestions({ proposalPid: 'TEST-001' });
    
    expect(response.loanId).toBe('TEST-001');
    expect(response.queue.length).toBeGreaterThan(0);
    expect(response.sections.length).toBeGreaterThan(0);
  });

  it('returns 404 for missing proposal', async () => {
    await expect(
      grpcClient.getQuestions({ proposalPid: 'NONEXISTENT' })
    ).rejects.toMatchObject({ code: 5 }); // NOT_FOUND
  });
});
```

### TS-18-02: Full Answer Flow
```typescript
describe('SubmitAnswer', () => {
  it('updates state and returns new queue', async () => {
    const before = await grpcClient.getQuestions({ proposalPid: 'TEST-001' });
    const questionId = before.queue[0].id;

    const response = await grpcClient.submitAnswer({
      proposalPid: 'TEST-001',
      questionId,
      answer: '"test_value"',
    });

    expect(response.queue.find(q => q.id === questionId)).toBeUndefined();
  });
});
```

### TS-18-03: Cache Behavior
```typescript
describe('Caching', () => {
  it('second request uses cache', async () => {
    resetMetrics();
    
    await grpcClient.getQuestions({ proposalPid: 'TEST-001' });
    await grpcClient.getQuestions({ proposalPid: 'TEST-001' });
    
    const metrics = getMetrics('cache_hit');
    expect(metrics.count).toBe(1);
  });
});
```

### TS-18-04: Latency Requirements
```typescript
describe('Performance', () => {
  it('hot path completes in < 10ms', async () => {
    // Warm up
    await grpcClient.getQuestions({ proposalPid: 'TEST-001' });
    
    const start = performance.now();
    await grpcClient.getQuestions({ proposalPid: 'TEST-001' });
    const elapsed = performance.now() - start;
    
    expect(elapsed).toBeLessThan(10);
  });
});
```

---

## Test Data Requirements

### Test Proposal (TEST-001)
- 2 borrowers
- 2 jobs
- 1 asset
- 1 liability
- 5 answered questions
- Standard question configs

### Test Configs
- 3 sections
- 10+ questions across levels
- Mix of criteria types

---

## Setup/Teardown

```typescript
// tests/integration/setup.ts
export async function seedTestData() {
  // Insert test proposal, borrowers, etc.
}

export async function cleanupTestData() {
  // Delete test data
}

export async function startTestServer() {
  // Start server on test port
}

export async function stopTestServer() {
  // Stop server
}
```
