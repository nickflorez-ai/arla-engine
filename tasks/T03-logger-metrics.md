# T03: Logger & Metrics

## Overview
Implement structured logging and latency metrics collection. Critical for debugging and performance monitoring.

## Dependencies
- T01: Project Setup

## Deliverables
- `src/utils/logger.ts` - Pino-based structured logger
- `src/utils/metrics.ts` - Latency tracking with percentiles

---

## Acceptance Criteria

- [ ] Logger outputs JSON in production, pretty-printed in development
- [ ] Log levels configurable via environment
- [ ] Metrics track P50, P95, P99 latencies
- [ ] Metrics track count per operation
- [ ] Metrics auto-prune old samples (keep last 1000)
- [ ] No performance impact (< 0.1ms per log call)

---

## Test Specifications

### TS-03-01: Logger Output Format
```typescript
// Test: Logger outputs structured JSON in production
process.env.NODE_ENV = 'production';
const { logger } = require('../src/utils/logger');

// Capture stdout
let output = '';
const originalWrite = process.stdout.write;
process.stdout.write = (chunk) => { output += chunk; return true; };

logger.info('test message', { key: 'value' });

process.stdout.write = originalWrite;

const parsed = JSON.parse(output);
expect(parsed.msg).toBe('test message');
expect(parsed.key).toBe('value');
expect(parsed.level).toBe(30); // info level
```

### TS-03-02: Metrics Percentile Accuracy
```typescript
// Test: Metrics calculate correct percentiles
import { recordLatency, getP50, getP95, getP99 } from '../src/utils/metrics';

// Record 100 samples: 1-100ms
for (let i = 1; i <= 100; i++) {
  recordLatency('test_op', i);
}

expect(getP50('test_op')).toBe(50);
expect(getP95('test_op')).toBe(95);
expect(getP99('test_op')).toBe(99);
```

### TS-03-03: Metrics Auto-Prune
```typescript
// Test: Metrics keep only last 1000 samples
import { recordLatency, getMetrics } from '../src/utils/metrics';

for (let i = 0; i < 1500; i++) {
  recordLatency('prune_test', i);
}

const metrics = getMetrics('prune_test');
expect(metrics.count).toBe(1000);
```

### TS-03-04: Logger Performance
```typescript
// Test: Logging adds < 0.1ms overhead
import { logger } from '../src/utils/logger';

const iterations = 10000;
const start = performance.now();

for (let i = 0; i < iterations; i++) {
  logger.debug('performance test', { i });
}

const elapsed = performance.now() - start;
const perCall = elapsed / iterations;

expect(perCall).toBeLessThan(0.1);
```

---

## Implementation

### src/utils/logger.ts
```typescript
import pino from 'pino';

const isDev = process.env.NODE_ENV !== 'production';
const level = process.env.LOG_LEVEL || (isDev ? 'debug' : 'info');

export const logger = pino({
  level,
  transport: isDev
    ? {
        target: 'pino-pretty',
        options: { colorize: true },
      }
    : undefined,
  base: {
    service: 'arla-engine',
    version: process.env.npm_package_version || '0.0.0',
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});

/**
 * Create a child logger with additional context.
 */
export function createLogger(context: Record<string, unknown>) {
  return logger.child(context);
}
```

### src/utils/metrics.ts
```typescript
const MAX_SAMPLES = 1000;

interface MetricsBucket {
  samples: number[];
}

const buckets = new Map<string, MetricsBucket>();

/**
 * Record a latency sample for an operation.
 */
export function recordLatency(operation: string, durationMs: number): void {
  let bucket = buckets.get(operation);
  if (!bucket) {
    bucket = { samples: [] };
    buckets.set(operation, bucket);
  }
  
  bucket.samples.push(durationMs);
  
  // Prune to keep only last MAX_SAMPLES
  if (bucket.samples.length > MAX_SAMPLES) {
    bucket.samples.shift();
  }
}

/**
 * Get percentile value for an operation.
 */
function getPercentile(operation: string, percentile: number): number {
  const bucket = buckets.get(operation);
  if (!bucket || bucket.samples.length === 0) return 0;
  
  const sorted = [...bucket.samples].sort((a, b) => a - b);
  const index = Math.floor(sorted.length * (percentile / 100));
  return sorted[Math.min(index, sorted.length - 1)];
}

export function getP50(operation: string): number {
  return getPercentile(operation, 50);
}

export function getP95(operation: string): number {
  return getPercentile(operation, 95);
}

export function getP99(operation: string): number {
  return getPercentile(operation, 99);
}

export function getMin(operation: string): number {
  const bucket = buckets.get(operation);
  if (!bucket || bucket.samples.length === 0) return 0;
  return Math.min(...bucket.samples);
}

export function getMax(operation: string): number {
  const bucket = buckets.get(operation);
  if (!bucket || bucket.samples.length === 0) return 0;
  return Math.max(...bucket.samples);
}

export function getCount(operation: string): number {
  const bucket = buckets.get(operation);
  return bucket?.samples.length || 0;
}

export interface OperationMetrics {
  count: number;
  p50: number;
  p95: number;
  p99: number;
  min: number;
  max: number;
}

export function getMetrics(operation: string): OperationMetrics {
  return {
    count: getCount(operation),
    p50: getP50(operation),
    p95: getP95(operation),
    p99: getP99(operation),
    min: getMin(operation),
    max: getMax(operation),
  };
}

export function getAllMetrics(): Record<string, OperationMetrics> {
  const result: Record<string, OperationMetrics> = {};
  for (const [op] of buckets) {
    result[op] = getMetrics(op);
  }
  return result;
}

/**
 * Reset all metrics. Useful for testing.
 */
export function resetMetrics(): void {
  buckets.clear();
}

/**
 * Timer utility for measuring operations.
 */
export function startTimer(): () => number {
  const start = performance.now();
  return () => performance.now() - start;
}

/**
 * Measure an async operation and record its latency.
 */
export async function measure<T>(
  operation: string,
  fn: () => Promise<T>
): Promise<T> {
  const timer = startTimer();
  try {
    return await fn();
  } finally {
    recordLatency(operation, timer());
  }
}
```

---

## Security Considerations

- [ ] Never log sensitive data (SSN, passwords)
- [ ] PII fields should be redacted in logs
- [ ] Log levels prevent verbose output in production

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Log call overhead | < 0.1ms |
| Metrics record overhead | < 0.01ms |
| Memory per 1000 samples | < 10KB |

---

## Error Handling

- Logger never throws (pino handles gracefully)
- Metrics operations never throw (return 0 on missing data)

---

## Notes

- Added `measure()` helper for convenient async timing
- Added `startTimer()` for manual timing control
- Added `resetMetrics()` for test isolation
- Added `createLogger()` for request-scoped context
