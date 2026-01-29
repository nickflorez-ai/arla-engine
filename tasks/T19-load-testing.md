# T19: Load Testing

## Overview
Load testing to verify performance at scale: concurrent connections, sustained throughput, latency under load.

## Dependencies
- T18: Integration Tests

## Deliverables
- `tests/load/runner.ts`
- `tests/load/scenarios/`
- Load test results documentation

---

## Acceptance Criteria

- [ ] Tests 500 requests/second sustained
- [ ] Measures P50, P95, P99 latency
- [ ] Tests concurrent connections
- [ ] Identifies breaking point
- [ ] Reports memory usage

---

## Test Scenarios

### LS-19-01: Baseline Hot Path
```typescript
// 100 requests, cached state
// Target: P95 < 10ms
```

### LS-19-02: Sustained Load
```typescript
// 500 req/sec for 60 seconds
// Target: P95 < 20ms, 0% errors
```

### LS-19-03: Cold Start Storm
```typescript
// 100 concurrent requests, uncached
// Target: All complete < 100ms
```

### LS-19-04: Mixed Workload
```typescript
// 70% GetQuestions, 30% SubmitAnswer
// 300 req/sec for 120 seconds
// Target: P95 < 30ms
```

### LS-19-05: Connection Ramp
```typescript
// 10 → 100 → 500 → 1000 connections
// Find breaking point
```

---

## Implementation

### tests/load/runner.ts
```typescript
import { performance } from 'perf_hooks';

interface LoadTestResult {
  totalRequests: number;
  successfulRequests: number;
  failedRequests: number;
  p50Ms: number;
  p95Ms: number;
  p99Ms: number;
  maxMs: number;
  requestsPerSecond: number;
  durationSeconds: number;
}

async function runLoadTest(
  scenario: () => Promise<void>,
  options: {
    concurrency: number;
    durationSeconds: number;
    targetRps?: number;
  }
): Promise<LoadTestResult> {
  const latencies: number[] = [];
  let failures = 0;
  const start = performance.now();
  const endTime = start + options.durationSeconds * 1000;

  // Worker loop
  async function worker() {
    while (performance.now() < endTime) {
      const reqStart = performance.now();
      try {
        await scenario();
        latencies.push(performance.now() - reqStart);
      } catch {
        failures++;
      }
      
      // Rate limiting if targetRps set
      if (options.targetRps) {
        const expectedTime = 1000 / options.targetRps;
        const actualTime = performance.now() - reqStart;
        if (actualTime < expectedTime) {
          await sleep(expectedTime - actualTime);
        }
      }
    }
  }

  // Run concurrent workers
  await Promise.all(
    Array(options.concurrency).fill(null).map(() => worker())
  );

  // Calculate results
  latencies.sort((a, b) => a - b);
  const duration = (performance.now() - start) / 1000;

  return {
    totalRequests: latencies.length + failures,
    successfulRequests: latencies.length,
    failedRequests: failures,
    p50Ms: latencies[Math.floor(latencies.length * 0.5)] || 0,
    p95Ms: latencies[Math.floor(latencies.length * 0.95)] || 0,
    p99Ms: latencies[Math.floor(latencies.length * 0.99)] || 0,
    maxMs: latencies[latencies.length - 1] || 0,
    requestsPerSecond: latencies.length / duration,
    durationSeconds: duration,
  };
}
```

---

## Success Criteria

| Scenario | Metric | Target |
|----------|--------|--------|
| Hot path | P95 | < 10ms |
| Sustained 500 RPS | P95 | < 20ms |
| Sustained 500 RPS | Error rate | 0% |
| Cold start | P99 | < 100ms |

---

## Reporting

Generate report with:
- Latency distribution histogram
- Throughput over time
- Error breakdown
- Resource usage (CPU, memory, connections)
