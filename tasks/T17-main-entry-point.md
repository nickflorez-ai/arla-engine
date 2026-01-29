# T17: Main Entry Point

## Overview
Application entry point: warmup connections, load configs, start server, handle shutdown.

## Dependencies
- T16: gRPC Handlers
- T14: SQS Queue Client

## Deliverables
- `src/index.ts`

---

## Acceptance Criteria

- [ ] Warmup Redis and Aurora connections
- [ ] Load all configs before accepting traffic
- [ ] Start gRPC server
- [ ] Handle SIGINT/SIGTERM gracefully
- [ ] Log startup metrics periodically
- [ ] Exit with proper codes

---

## Test Specifications

### TS-17-01: Startup Sequence
```typescript
// Process should start and log "ready"
// Check logs for correct order:
// 1. Warming up Redis
// 2. Warming up Aurora
// 3. Loading configs
// 4. Starting gRPC server
// 5. Ready
```

### TS-17-02: Graceful Shutdown
```typescript
// Send SIGTERM, verify:
// 1. gRPC server stops accepting
// 2. Connections closed
// 3. Exit code 0
```

### TS-17-03: Startup Failure
```typescript
// With invalid config, should:
// 1. Log error
// 2. Exit code 1
```

---

## Implementation

See `EXECUTION_PLAN_FOR_REVIEW.md` Step 26 for full implementation.

Key features:
- Sequential warmup (Redis → Aurora → configs → server)
- Periodic metrics logging (every 60s)
- Signal handlers for clean shutdown
- Clear error messages on failure

---

## Startup Order

```
1. Load environment config
2. Warmup Redis (getRedis().ping())
3. Warmup Aurora (pool.execute('SELECT 1'))
4. Load configs (loadAllConfigs())
5. Start gRPC server
6. Log "ready"
7. Start metrics interval
```

## Shutdown Order

```
1. Stop gRPC server (stop accepting)
2. Close rules engine
3. Close Redis
4. Close Aurora pool
5. Log final metrics
6. Exit
```
