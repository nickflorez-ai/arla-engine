# T15: gRPC Server

## Overview
gRPC server setup with proto loading, service registration, and graceful shutdown.

## Dependencies
- T03: Logger & Metrics

## Deliverables
- `src/grpc/server.ts`
- `proto/arla.proto`

---

## Acceptance Criteria

- [ ] Loads proto definition
- [ ] Binds to configurable port
- [ ] Registers QuestionService
- [ ] Graceful shutdown
- [ ] Health check endpoint

---

## Test Specifications

### TS-15-01: Server Starts
```typescript
await startGrpcServer();
// Server should be listening
await stopGrpcServer();
```

### TS-15-02: Proto Loads
```typescript
// Proto file should parse without error
const packageDef = protoLoader.loadSync('proto/arla.proto');
expect(packageDef).toBeDefined();
```

### TS-15-03: Graceful Shutdown
```typescript
await startGrpcServer();
await expect(stopGrpcServer()).resolves.not.toThrow();
```

---

## Implementation

See `EXECUTION_PLAN_FOR_REVIEW.md` Steps 23-24 for full implementation.

### proto/arla.proto
```protobuf
syntax = "proto3";
package arla;

service QuestionService {
  rpc GetQuestions(GetQuestionsRequest) returns (QuestionQueueResponse);
  rpc SubmitAnswer(SubmitAnswerRequest) returns (QuestionQueueResponse);
  rpc GetLoanState(GetLoanStateRequest) returns (LoanStateResponse);
  rpc HealthCheck(Empty) returns (HealthResponse);
}

message Empty {}
message HealthResponse { bool healthy = 1; }
// ... other messages as in execution plan
```

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Request overhead | < 2ms |
| Startup time | < 100ms |
