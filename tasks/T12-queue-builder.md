# T12: Queue Builder

## Overview
Build the sorted question queue from evaluation results, calculate section progress, and identify combinable questions.

## Dependencies
- T11: Question Evaluator

## Deliverables
- `src/questions/queue-builder.ts`

---

## Acceptance Criteria

- [ ] Sorts by section sequence, then ordinal
- [ ] Calculates section progress (total/answered/status)
- [ ] Identifies questions that can be asked together
- [ ] Returns `nextRecommended` question
- [ ] Includes state version for cache coherence

---

## Test Specifications

### TS-12-01: Sorted Output
```typescript
const queue = await buildQuestionQueue(loanState);
for (let i = 1; i < queue.queue.length; i++) {
  const prev = queue.queue[i-1];
  const curr = queue.queue[i];
  // Either different section (in order) or same section with ordinal order
  expect(curr.section >= prev.section || curr.ordinal >= prev.ordinal).toBe(true);
}
```

### TS-12-02: Section Progress
```typescript
const queue = await buildQuestionQueue(loanState);
for (const section of queue.sections) {
  expect(section.total).toBeGreaterThanOrEqual(section.answered);
  expect(['pending', 'in_progress', 'complete']).toContain(section.status);
}
```

### TS-12-03: Combinable Questions
```typescript
const queue = await buildQuestionQueue(loanState);
for (const group of queue.canAskTogether) {
  expect(group.length).toBeGreaterThan(1);
  // All in group should have same section and level
}
```

---

## Implementation

See `EXECUTION_PLAN_FOR_REVIEW.md` Step 20 for full implementation.

Key features:
- Pre-sort by section sequence map
- Section status: pending (0 answered), in_progress (partial), complete (all answered)
- Combinable: consecutive questions with same section/level/flexibility

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Build from 100 results | < 1ms |
| Sort overhead | O(n log n) |
