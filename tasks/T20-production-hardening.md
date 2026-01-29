# T20: Production Hardening

## Overview
Final hardening for production: security, observability, resilience, and deployment configuration.

## Dependencies
- T19: Load Testing

## Deliverables
- Security audit fixes
- Prometheus metrics endpoint
- Health check endpoints
- Circuit breaker implementation
- Dockerfile and deployment config
- Runbook documentation

---

## Acceptance Criteria

- [ ] All security checklist items complete
- [ ] Prometheus-compatible metrics
- [ ] Kubernetes-ready health probes
- [ ] Circuit breakers on external calls
- [ ] TLS for all connections
- [ ] Runbook for operations

---

## Hardening Checklist

### Security
- [ ] No secrets in code/logs
- [ ] TLS for Redis connection
- [ ] TLS for Aurora connection
- [ ] TLS for gRPC server
- [ ] Input validation on all endpoints
- [ ] Rate limiting
- [ ] Auth/AuthZ (if applicable)

### Observability
- [ ] Prometheus `/metrics` endpoint
- [ ] Structured JSON logs
- [ ] Request ID tracing
- [ ] Latency histograms
- [ ] Error rate metrics
- [ ] Connection pool metrics

### Resilience
- [ ] Circuit breaker on Redis
- [ ] Circuit breaker on Aurora
- [ ] Timeout on all external calls
- [ ] Graceful degradation paths
- [ ] Retry with backoff

### Deployment
- [ ] Multi-stage Dockerfile
- [ ] Non-root container user
- [ ] Health check endpoints
- [ ] Resource limits
- [ ] Horizontal scaling tested

---

## Implementation

### Dockerfile
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
RUN adduser -D arla
USER arla
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/config ./config
EXPOSE 50051
CMD ["node", "dist/index.js"]
```

### Health Endpoints
```typescript
// Add to gRPC service
rpc HealthCheck(Empty) returns (HealthResponse);
rpc ReadinessCheck(Empty) returns (ReadinessResponse);

// HealthCheck: Is process alive?
// ReadinessCheck: Is ready to serve? (connections warm)
```

### Circuit Breaker
```typescript
import CircuitBreaker from 'opossum';

const redisBreaker = new CircuitBreaker(redisOperation, {
  timeout: 1000,
  errorThresholdPercentage: 50,
  resetTimeout: 5000,
});

redisBreaker.fallback(() => {
  // Load directly from Aurora
});
```

---

## Runbook Topics

1. **Startup procedure**
2. **Shutdown procedure**
3. **Common errors and fixes**
4. **Performance troubleshooting**
5. **Cache invalidation**
6. **Emergency procedures**
7. **Scaling guidance**
8. **Monitoring alerts**

---

## Deployment Verification

After deploy:
1. [ ] Health check passes
2. [ ] Readiness check passes
3. [ ] Sample request succeeds
4. [ ] Metrics endpoint accessible
5. [ ] Logs flowing to aggregator
6. [ ] Latency within targets
