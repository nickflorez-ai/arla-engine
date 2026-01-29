# T06: Redis Client

## Overview
Implement Redis client with connection management, health checks, and msgpack serialization support.

## Dependencies
- T02: Type Definitions
- T03: Logger & Metrics

## Deliverables
- `src/cache/redis.ts` - Redis client wrapper

---

## Acceptance Criteria

- [ ] Lazy connection (connects on first use)
- [ ] Configurable via environment
- [ ] Health check function
- [ ] Warmup function for startup
- [ ] Graceful close
- [ ] Error logging

---

## Test Specifications

### TS-06-01: Connection
```typescript
// Test: Connects to Redis
import { getRedis, closeRedis } from '../src/cache/redis';

const redis = getRedis();
const pong = await redis.ping();
expect(pong).toBe('PONG');
await closeRedis();
```

### TS-06-02: Singleton Pattern
```typescript
// Test: Returns same instance
import { getRedis, closeRedis } from '../src/cache/redis';

const r1 = getRedis();
const r2 = getRedis();
expect(r1).toBe(r2);
await closeRedis();
```

### TS-06-03: Set and Get
```typescript
// Test: Basic operations work
import { getRedis, closeRedis } from '../src/cache/redis';

const redis = getRedis();
await redis.set('test:key', 'value');
const result = await redis.get('test:key');
expect(result).toBe('value');
await redis.del('test:key');
await closeRedis();
```

### TS-06-04: Binary Data (msgpack)
```typescript
// Test: Binary data round-trips correctly
import { getRedis, closeRedis } from '../src/cache/redis';
import { encode, decode } from '@msgpack/msgpack';

const redis = getRedis();
const data = { key: 'value', num: 42 };
const packed = Buffer.from(encode(data));

await redis.setex('test:binary', 60, packed);
const retrieved = await redis.getBuffer('test:binary');
const unpacked = decode(retrieved!);

expect(unpacked).toEqual(data);
await redis.del('test:binary');
await closeRedis();
```

### TS-06-05: Health Check
```typescript
// Test: Health check passes
import { isRedisHealthy, closeRedis } from '../src/cache/redis';

const healthy = await isRedisHealthy();
expect(healthy).toBe(true);
await closeRedis();
```

---

## Implementation

### src/cache/redis.ts
```typescript
import Redis from 'ioredis';
import { config } from '../config';
import { logger } from '../utils/logger';

let redis: Redis | null = null;

/**
 * Get or create Redis client.
 * Uses lazy initialization.
 */
export function getRedis(): Redis {
  if (!redis) {
    redis = new Redis({
      host: config.redis.host,
      port: config.redis.port,
      password: config.redis.password || undefined,
      maxRetriesPerRequest: 3,
      enableReadyCheck: true,
      lazyConnect: true,
      retryStrategy: (times) => {
        if (times > 3) {
          logger.error('Redis connection failed after 3 retries');
          return null; // Stop retrying
        }
        return Math.min(times * 100, 1000);
      },
    });

    redis.on('connect', () => {
      logger.info('Redis connected', { host: config.redis.host });
    });

    redis.on('error', (err) => {
      logger.error('Redis error', { error: err.message });
    });

    redis.on('close', () => {
      logger.debug('Redis connection closed');
    });
  }
  return redis;
}

/**
 * Warmup Redis connection.
 * Call at startup before accepting traffic.
 */
export async function warmupRedis(): Promise<void> {
  logger.info('Warming up Redis connection...');
  const r = getRedis();
  
  const start = performance.now();
  await r.ping();
  const elapsed = performance.now() - start;
  
  logger.info('Redis warmup complete', { durationMs: elapsed });
}

/**
 * Check if Redis is healthy.
 */
export async function isRedisHealthy(): Promise<boolean> {
  try {
    const r = getRedis();
    const pong = await r.ping();
    return pong === 'PONG';
  } catch (error) {
    logger.error('Redis health check failed', { error: (error as Error).message });
    return false;
  }
}

/**
 * Close Redis connection gracefully.
 */
export async function closeRedis(): Promise<void> {
  if (redis) {
    await redis.quit();
    redis = null;
    logger.info('Redis connection closed');
  }
}

/**
 * Get Redis info for monitoring.
 */
export async function getRedisInfo(): Promise<Record<string, string>> {
  const r = getRedis();
  const info = await r.info();
  
  const result: Record<string, string> = {};
  for (const line of info.split('\n')) {
    const [key, value] = line.split(':');
    if (key && value) {
      result[key.trim()] = value.trim();
    }
  }
  return result;
}
```

### src/config/index.ts (redis section)
```typescript
redis: {
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379', 10),
  password: process.env.REDIS_PASSWORD || undefined,
},
```

---

## Security Considerations

- [ ] Password from environment, never hardcoded
- [ ] TLS connection in production (add `tls` config)
- [ ] Connection string never logged

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Ping latency | < 1ms |
| Set/Get latency | < 1ms |
| Warmup time | < 50ms |

---

## Error Handling

| Error | Handling |
|-------|----------|
| Connection refused | Retry 3 times, then fail |
| Auth failure | Log error, fail immediately |
| Network error | Auto-reconnect (ioredis default) |
| Command timeout | Log and throw |

---

## Notes

- Uses `ioredis` for robust connection handling
- `lazyConnect: true` prevents blocking on import
- `getBuffer()` needed for msgpack binary data
- Retry strategy backs off exponentially
