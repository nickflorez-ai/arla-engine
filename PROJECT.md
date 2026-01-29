# ARLA Engine - Project Overview

**AI-Ready Lending API - Low-Latency Question Engine**

## Vision

A millisecond-latency question evaluation engine for voice agents that makes mortgage applications conversational. Evaluates 100+ business rules in <10ms, enabling natural voice-driven loan origination.

## Performance Targets

| Scenario | Target | Max |
|----------|--------|-----|
| Hot path (cached) | <10ms P50 | 20ms |
| Cold start | <50ms | 100ms |
| Answer submission | <5ms | 15ms |
| Cache hit rate | >95% | - |

## Architecture

```
Voice Agent → gRPC → ARLA Engine
                        ├── In-Memory: Pre-compiled rules (Zen Engine)
                        ├── Redis: Loan state cache (msgpack)
                        └── Aurora: Persistent storage (async writes)
```

## Tech Stack

| Component | Technology |
|-----------|------------|
| Runtime | Node.js 20+ |
| Rules Engine | GoRules Zen (Rust bindings) |
| Cache | Redis 7 + msgpack |
| Database | Aurora MySQL |
| API | gRPC |
| Queue | AWS SQS |

---

## Task Dependency Graph

```
[T01] Project Setup
  │
  ├──→ [T02] Type Definitions
  │       │
  │       ├──→ [T03] Logger & Metrics
  │       │       │
  │       │       ├──→ [T04] Aurora Connection Pool
  │       │       │       │
  │       │       │       └──→ [T05] Loan State Loader
  │       │       │
  │       │       ├──→ [T06] Redis Client
  │       │       │       │
  │       │       │       └──→ [T07] Loan State Cache
  │       │       │
  │       │       └──→ [T08] Zen Engine Wrapper
  │       │               │
  │       │               └──→ [T09] ANTLR-to-Zen Compiler
  │       │
  │       └──→ [T10] Question Config Loader
  │               │
  │               └──→ [T11] Question Evaluator
  │                       │
  │                       └──→ [T12] Queue Builder
  │                               │
  │                               └──→ [T13] Answer Handler
  │
  └──→ [T14] SQS Queue Client
          │
          └──→ [T15] gRPC Server
                  │
                  └──→ [T16] gRPC Handlers
                          │
                          └──→ [T17] Main Entry Point
                                  │
                                  └──→ [T18] Integration Tests
                                          │
                                          └──→ [T19] Load Testing
                                                  │
                                                  └──→ [T20] Production Hardening
```

---

## Task Summary

| ID | Task | Dependencies | Estimated Hours |
|----|------|--------------|-----------------|
| T01 | Project Setup | - | 1 |
| T02 | Type Definitions | T01 | 2 |
| T03 | Logger & Metrics | T01 | 1 |
| T04 | Aurora Connection Pool | T02, T03 | 2 |
| T05 | Loan State Loader | T04 | 3 |
| T06 | Redis Client | T02, T03 | 1 |
| T07 | Loan State Cache | T06, T05 | 2 |
| T08 | Zen Engine Wrapper | T03 | 2 |
| T09 | ANTLR-to-Zen Compiler | T08 | 4 |
| T10 | Question Config Loader | T02, T09 | 3 |
| T11 | Question Evaluator | T10, T07, T08 | 4 |
| T12 | Queue Builder | T11 | 2 |
| T13 | Answer Handler | T12, T07 | 2 |
| T14 | SQS Queue Client | T03 | 1 |
| T15 | gRPC Server | T03 | 2 |
| T16 | gRPC Handlers | T15, T12, T13 | 3 |
| T17 | Main Entry Point | T16, T14 | 2 |
| T18 | Integration Tests | T17 | 4 |
| T19 | Load Testing | T18 | 3 |
| T20 | Production Hardening | T19 | 4 |

**Total Estimated: ~48 hours**

---

## Critical Path

The critical path for MVP:

```
T01 → T02 → T03 → T08 → T09 → T10 → T11 → T12 → T16 → T17
```

Focus here first. Cache layer (T06-T07) and Aurora (T04-T05) can be stubbed initially.

---

## Spec-Driven Development

Each task includes:
1. **Acceptance Criteria** - What "done" looks like
2. **Test Specifications** - Deterministic tests
3. **Performance Requirements** - Latency/throughput targets
4. **Security Considerations** - Input validation, access control
5. **Error Handling** - Failure modes and recovery

Tests are written **before** implementation. Implementation is complete when all tests pass.

---

## Directory Structure

```
arla-engine/
├── PROJECT.md              # This file
├── tasks/                  # Individual task specs
│   ├── T01-project-setup.md
│   ├── T02-type-definitions.md
│   └── ...
├── src/                    # Implementation
├── tests/                  # Test files
│   ├── unit/
│   ├── integration/
│   └── load/
├── config/                 # Question configs
└── proto/                  # gRPC definitions
```

---

## How to Work on a Task

1. Read the task spec in `tasks/T##-name.md`
2. Write tests first based on test specifications
3. Implement until tests pass
4. Verify performance requirements
5. Document any deviations
6. Create PR with task ID in title

---

## Related Documents

- `../arla/PRD.md` - Product Requirements Document
- `../arla/ARLA_LOW_LATENCY_ARCHITECTURE.md` - Architecture deep dive
- `../arla/EXECUTION_PLAN_FOR_REVIEW.md` - Full implementation guide
