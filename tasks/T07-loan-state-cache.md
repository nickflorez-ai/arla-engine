# T07: Loan State Cache

## Overview
Implement loan state caching with Redis, using msgpack for efficient serialization and supporting partial updates.

## Dependencies
- T06: Redis Client
- T05: Loan State Loader

## Deliverables
- `src/cache/loan-state.ts` - Loan state cache operations

---

## Acceptance Criteria

- [ ] Get state from cache or load from Aurora
- [ ] Serialize with msgpack (not JSON)
- [ ] TTL of 1 hour
- [ ] Update cache atomically
- [ ] Invalidate cache on demand
- [ ] Track cache hit/miss metrics

---

## Test Specifications

### TS-07-01: Cache Miss Loads from Aurora
```typescript
// Test: Cache miss triggers Aurora load
import { getLoanState, invalidateLoanStateCache } from '../src/cache/loan-state';
import { getRedis } from '../src/cache/redis';

// Ensure cache is empty
await invalidateLoanStateCache('TEST-001');

const state = await getLoanState('TEST-001');
expect(state.proposalPid).toBe('TEST-001');
```

### TS-07-02: Cache Hit Returns Cached
```typescript
// Test: Second call returns cached state
import { getLoanState, invalidateLoanStateCache } from '../src/cache/loan-state';
import { getMetrics, resetMetrics } from '../src/utils/metrics';

resetMetrics();
await invalidateLoanStateCache('TEST-001');

// First call - cache miss
await getLoanState('TEST-001');

// Second call - cache hit
const state2 = await getLoanState('TEST-001');
expect(state2.proposalPid).toBe('TEST-001');

// Verify only one Aurora load
const metrics = getMetrics('aurora_load_loan_state');
expect(metrics.count).toBe(1);
```

### TS-07-03: Update Cache
```typescript
// Test: Cache update persists changes
import { getLoanState, updateLoanStateCache } from '../src/cache/loan-state';

const original = await getLoanState('TEST-001');
const originalVersion = original.version;

await updateLoanStateCache('TEST-001', { newField: 'newValue' }, 'Q100');

const updated = await getLoanState('TEST-001');
expect(updated.fields.newField).toBe('newValue');
expect(updated.answered.has('Q100')).toBe(true);
expect(updated.version).toBeGreaterThan(originalVersion);
```

### TS-07-04: Invalidation Works
```typescript
// Test: Invalidation clears cache
import { getLoanState, invalidateLoanStateCache } from '../src/cache/loan-state';
import { getRedis } from '../src/cache/redis';

await getLoanState('TEST-001');

const redis = getRedis();
const beforeInvalidate = await redis.exists('loan:TEST-001:fields');
expect(beforeInvalidate).toBe(1);

await invalidateLoanStateCache('TEST-001');

const afterInvalidate = await redis.exists('loan:TEST-001:fields');
expect(afterInvalidate).toBe(0);
```

### TS-07-05: TTL Set Correctly
```typescript
// Test: Cache entries have 1-hour TTL
import { getLoanState } from '../src/cache/loan-state';
import { getRedis } from '../src/cache/redis';

await getLoanState('TEST-001');

const redis = getRedis();
const ttl = await redis.ttl('loan:TEST-001:fields');

expect(ttl).toBeGreaterThan(3500); // ~1 hour
expect(ttl).toBeLessThanOrEqual(3600);
```

### TS-07-06: Serialization Size
```typescript
// Test: msgpack is smaller than JSON
import { encode } from '@msgpack/msgpack';

const testState = {
  fields: { field1: 'value1', field2: 123 },
  entities: { borrowers: [{ pid: 'b1', name: 'John', fields: {} }] },
};

const msgpackSize = encode(testState).length;
const jsonSize = JSON.stringify(testState).length;

expect(msgpackSize).toBeLessThan(jsonSize);
```

---

## Implementation

### src/cache/loan-state.ts
```typescript
import { encode, decode } from '@msgpack/msgpack';
import { getRedis } from './redis';
import { LoanState, LoanStateSerialized } from '../questions/types';
import { loadLoanStateFromAurora } from '../db/loader';
import { logger } from '../utils/logger';
import { recordLatency, measure } from '../utils/metrics';

const CACHE_TTL = 3600; // 1 hour

// Key structure for split caching
const KEYS = {
  fields: (pid: string) => `loan:${pid}:fields`,
  entities: (pid: string) => `loan:${pid}:entities`,
  answered: (pid: string) => `loan:${pid}:answered`,
  meta: (pid: string) => `loan:${pid}:meta`,
};

/**
 * Get loan state from cache or load from Aurora.
 */
export async function getLoanState(proposalPid: string): Promise<LoanState> {
  return measure('cache_get_loan_state', async () => {
    const redis = getRedis();
    const fieldsKey = KEYS.fields(proposalPid);

    // Check cache
    const cachedFields = await redis.getBuffer(fieldsKey);

    if (cachedFields) {
      logger.debug('Loan state cache HIT', { proposalPid });
      recordLatency('cache_hit', 1);
      return await loadFromCache(proposalPid);
    }

    // Cache miss - load from Aurora
    logger.debug('Loan state cache MISS', { proposalPid });
    recordLatency('cache_miss', 1);

    const state = await loadLoanStateFromAurora(proposalPid);
    await saveToCache(state);

    return state;
  });
}

/**
 * Update loan state in cache after answer submission.
 * Does NOT write to Aurora - that's async via SQS.
 */
export async function updateLoanStateCache(
  proposalPid: string,
  fieldUpdates: Record<string, unknown>,
  answeredQuestionId: string
): Promise<LoanState> {
  return measure('cache_update_loan_state', async () => {
    // Get current state
    const state = await getLoanState(proposalPid);

    // Apply updates
    state.fields = { ...state.fields, ...fieldUpdates };
    state.answered.add(answeredQuestionId);
    state.version = Date.now();

    // Save back to cache
    await saveToCache(state);

    logger.debug('Loan state cache updated', {
      proposalPid,
      answeredQuestionId,
      fieldsUpdated: Object.keys(fieldUpdates).length,
    });

    return state;
  });
}

/**
 * Invalidate cache for a loan.
 */
export async function invalidateLoanStateCache(proposalPid: string): Promise<void> {
  const redis = getRedis();
  
  await redis.del(
    KEYS.fields(proposalPid),
    KEYS.entities(proposalPid),
    KEYS.answered(proposalPid),
    KEYS.meta(proposalPid)
  );
  
  logger.debug('Loan state cache invalidated', { proposalPid });
}

/**
 * Load full state from cache (all keys).
 */
async function loadFromCache(proposalPid: string): Promise<LoanState> {
  const redis = getRedis();

  const [fieldsBuffer, entitiesBuffer, answeredData, metaBuffer] = await Promise.all([
    redis.getBuffer(KEYS.fields(proposalPid)),
    redis.getBuffer(KEYS.entities(proposalPid)),
    redis.smembers(KEYS.answered(proposalPid)),
    redis.getBuffer(KEYS.meta(proposalPid)),
  ]);

  if (!fieldsBuffer || !entitiesBuffer || !metaBuffer) {
    throw new Error(`Incomplete cache state for ${proposalPid}`);
  }

  const fields = decode(fieldsBuffer) as Record<string, unknown>;
  const entities = decode(entitiesBuffer) as LoanState['entities'];
  const meta = decode(metaBuffer) as { version: number; loadedAt: string };

  return {
    proposalPid,
    version: meta.version,
    loadedAt: new Date(meta.loadedAt),
    fields,
    entities,
    answered: new Set(answeredData),
  };
}

/**
 * Save state to cache (split keys).
 */
async function saveToCache(state: LoanState): Promise<void> {
  const redis = getRedis();
  const pid = state.proposalPid;

  const fieldsBuffer = Buffer.from(encode(state.fields));
  const entitiesBuffer = Buffer.from(encode(state.entities));
  const metaBuffer = Buffer.from(encode({
    version: state.version,
    loadedAt: state.loadedAt.toISOString(),
  }));
  const answeredArray = Array.from(state.answered);

  // Use pipeline for atomicity
  const pipeline = redis.pipeline();
  
  pipeline.setex(KEYS.fields(pid), CACHE_TTL, fieldsBuffer);
  pipeline.setex(KEYS.entities(pid), CACHE_TTL, entitiesBuffer);
  pipeline.setex(KEYS.meta(pid), CACHE_TTL, metaBuffer);
  
  // Use set for answered questions
  pipeline.del(KEYS.answered(pid));
  if (answeredArray.length > 0) {
    pipeline.sadd(KEYS.answered(pid), ...answeredArray);
    pipeline.expire(KEYS.answered(pid), CACHE_TTL);
  }

  await pipeline.exec();
}

/**
 * Check if a loan is cached.
 */
export async function isLoanCached(proposalPid: string): Promise<boolean> {
  const redis = getRedis();
  const exists = await redis.exists(KEYS.fields(proposalPid));
  return exists === 1;
}
```

---

## Security Considerations

- [ ] No sensitive data logged
- [ ] TTL ensures stale data expires
- [ ] No PII in cache keys

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Cache get (hit) | < 2ms |
| Cache update | < 3ms |
| msgpack encode | < 0.5ms |
| msgpack decode | < 0.5ms |

---

## Error Handling

| Error | Handling |
|-------|----------|
| Cache miss | Load from Aurora (graceful fallback) |
| Redis unavailable | Load directly from Aurora (log warning) |
| Partial cache | Invalidate and reload |
| Encoding error | Log and throw |

---

## Notes

- Uses split keys for efficient partial updates
- `answered` uses Redis Set for O(1) add/check
- Pipeline for atomic multi-key operations
- msgpack 2-5x smaller and faster than JSON
