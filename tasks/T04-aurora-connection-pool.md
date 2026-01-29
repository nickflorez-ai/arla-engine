# T04: Aurora Connection Pool

## Overview
Implement MySQL connection pool for Aurora database access with connection warmup and health checks.

## Dependencies
- T02: Type Definitions
- T03: Logger & Metrics

## Deliverables
- `src/db/aurora.ts` - Connection pool with warmup

---

## Acceptance Criteria

- [ ] Connection pool creates on first query
- [ ] Pool size configurable via environment
- [ ] Keep-alive prevents connection timeout
- [ ] `warmup()` function tests connectivity
- [ ] `query()` function returns typed results
- [ ] Pool closes gracefully on shutdown

---

## Test Specifications

### TS-04-01: Connection Pool Creation
```typescript
// Test: Pool connects to database
import { getPool, closePool } from '../src/db/aurora';

const pool = await getPool();
const [rows] = await pool.execute('SELECT 1 as test');
expect(rows[0].test).toBe(1);
await closePool();
```

### TS-04-02: Query Helper
```typescript
// Test: query() returns typed results
import { query, closePool } from '../src/db/aurora';

interface TestRow { value: number }
const rows = await query<TestRow>('SELECT 1 as value');
expect(rows[0].value).toBe(1);
await closePool();
```

### TS-04-03: Warmup Function
```typescript
// Test: warmup() succeeds with healthy database
import { warmup, closePool } from '../src/db/aurora';

await expect(warmup()).resolves.not.toThrow();
await closePool();
```

### TS-04-04: Pool Reuse
```typescript
// Test: Multiple getPool() calls return same pool
import { getPool, closePool } from '../src/db/aurora';

const pool1 = await getPool();
const pool2 = await getPool();
expect(pool1).toBe(pool2);
await closePool();
```

### TS-04-05: Connection Timeout
```typescript
// Test: Query timeout works
import { query, closePool } from '../src/db/aurora';

await expect(
  query('SELECT SLEEP(5)', [], { timeout: 100 })
).rejects.toThrow();
await closePool();
```

---

## Implementation

### src/db/aurora.ts
```typescript
import mysql, { Pool, PoolOptions, QueryResult } from 'mysql2/promise';
import { config } from '../config';
import { logger } from '../utils/logger';
import { measure } from '../utils/metrics';

let pool: Pool | null = null;

const poolConfig: PoolOptions = {
  host: config.aurora.host,
  port: config.aurora.port,
  user: config.aurora.user,
  password: config.aurora.password,
  database: config.aurora.database,
  connectionLimit: config.aurora.poolSize,
  waitForConnections: true,
  queueLimit: 0,
  enableKeepAlive: true,
  keepAliveInitialDelay: 10000,
  connectTimeout: 5000,
};

/**
 * Get or create the connection pool.
 */
export async function getPool(): Promise<Pool> {
  if (!pool) {
    pool = mysql.createPool(poolConfig);
    logger.info('Aurora pool created', {
      host: config.aurora.host,
      database: config.aurora.database,
      poolSize: config.aurora.poolSize,
    });
  }
  return pool;
}

export interface QueryOptions {
  timeout?: number;
}

/**
 * Execute a query and return typed results.
 */
export async function query<T>(
  sql: string,
  params?: unknown[],
  options?: QueryOptions
): Promise<T[]> {
  return measure('aurora_query', async () => {
    const p = await getPool();
    
    const queryOptions = options?.timeout
      ? { timeout: options.timeout }
      : undefined;
    
    const [rows] = await p.execute(sql, params, queryOptions);
    return rows as T[];
  });
}

/**
 * Warmup the connection pool by executing a simple query.
 * Call this at startup before accepting traffic.
 */
export async function warmup(): Promise<void> {
  logger.info('Warming up Aurora connection pool...');
  
  const p = await getPool();
  const start = performance.now();
  
  // Get a connection and run a simple query
  const connection = await p.getConnection();
  await connection.execute('SELECT 1');
  connection.release();
  
  const elapsed = performance.now() - start;
  logger.info('Aurora warmup complete', { durationMs: elapsed });
}

/**
 * Close the connection pool gracefully.
 */
export async function closePool(): Promise<void> {
  if (pool) {
    await pool.end();
    pool = null;
    logger.info('Aurora pool closed');
  }
}

/**
 * Check if the pool is healthy.
 */
export async function isHealthy(): Promise<boolean> {
  try {
    const p = await getPool();
    const connection = await p.getConnection();
    await connection.execute('SELECT 1');
    connection.release();
    return true;
  } catch (error) {
    logger.error('Aurora health check failed', { error: (error as Error).message });
    return false;
  }
}
```

### src/config/index.ts (aurora section)
```typescript
aurora: {
  host: process.env.AURORA_HOST || 'localhost',
  port: parseInt(process.env.AURORA_PORT || '3306', 10),
  user: process.env.AURORA_USER || 'arla',
  password: process.env.AURORA_PASSWORD || '',
  database: process.env.AURORA_DATABASE || 'octane',
  poolSize: parseInt(process.env.AURORA_POOL_SIZE || '20', 10),
},
```

---

## Security Considerations

- [ ] Password from environment, never hardcoded
- [ ] SSL/TLS connection in production (add `ssl` config)
- [ ] Connection string never logged
- [ ] Prepared statements prevent SQL injection

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Connection acquisition | < 5ms |
| Query latency overhead | < 1ms |
| Pool warmup | < 100ms |

---

## Error Handling

| Error | Handling |
|-------|----------|
| Connection refused | Log error, throw (startup fails) |
| Query timeout | Log error, throw |
| Pool exhausted | Queue request (waitForConnections: true) |
| Connection lost | Auto-reconnect (mysql2 default) |

---

## Notes

- Uses `mysql2/promise` for async/await support
- Keep-alive prevents Aurora connection timeouts
- `measure()` wraps queries for latency tracking
- `warmup()` should be called before accepting traffic
