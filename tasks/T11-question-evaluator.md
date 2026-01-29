# T11: Question Evaluator

## Overview
Evaluate which questions apply to a loan based on current state. Uses parallel rule evaluation for performance.

## Dependencies
- T10: Question Config Loader
- T07: Loan State Cache
- T08: Zen Engine Wrapper

## Deliverables
- `src/questions/evaluator.ts` - Parallel question evaluator

---

## Acceptance Criteria

- [ ] Skips already-answered questions
- [ ] Evaluates rules per entity level
- [ ] Runs evaluations in parallel
- [ ] Substitutes merge fields in instructions
- [ ] Enforces latency budget
- [ ] Returns applicable questions with context

---

## Test Specifications

### TS-11-01: Skip Answered Questions
```typescript
// Test: Already-answered questions not returned
import { evaluateQuestions } from '../src/questions/evaluator';
import { LoanState } from '../src/questions/types';

const state: LoanState = {
  proposalPid: 'TEST-001',
  version: 1,
  loadedAt: new Date(),
  fields: {},
  entities: { borrowers: [], jobs: [], assets: [], liabilities: [], realEstateOwned: [] },
  answered: new Set(['Q100']), // Q100 already answered
};

const results = await evaluateQuestions(state);

// Q100 should not be in results
expect(results.find(q => q.id === 'Q100')).toBeUndefined();
```

### TS-11-02: Evaluate Per Entity
```typescript
// Test: Borrower-level question evaluated per borrower
import { evaluateQuestions } from '../src/questions/evaluator';
import { LoanState } from '../src/questions/types';

const state: LoanState = {
  proposalPid: 'TEST-001',
  version: 1,
  loadedAt: new Date(),
  fields: {},
  entities: {
    borrowers: [
      { pid: 'b1', name: 'John', fields: {} },
      { pid: 'b2', name: 'Jane', fields: {} },
    ],
    jobs: [], assets: [], liabilities: [], realEstateOwned: [],
  },
  answered: new Set(),
};

const results = await evaluateQuestions(state);

// Count borrower-level questions - should have entries for both borrowers
const borrowerQuestions = results.filter(q => q.level === 'BORROWER');
// Each question should appear twice (once per borrower)
```

### TS-11-03: Merge Fields Substituted
```typescript
// Test: {{field}} replaced with values
import { evaluateQuestions } from '../src/questions/evaluator';
import { LoanState } from '../src/questions/types';

// Assuming Q_JOB_HOURS has instructions: "How many hours per week do you work at {{employer_name}}?"

const state: LoanState = {
  proposalPid: 'TEST-001',
  version: 1,
  loadedAt: new Date(),
  fields: {},
  entities: {
    borrowers: [{ pid: 'b1', name: 'John', fields: {} }],
    jobs: [{ pid: 'j1', name: 'Acme Corp', fields: { employerName: 'Acme Corp' } }],
    assets: [], liabilities: [], realEstateOwned: [],
  },
  answered: new Set(),
};

const results = await evaluateQuestions(state);
const jobQuestion = results.find(q => q.level === 'JOB');

if (jobQuestion) {
  expect(jobQuestion.text).toContain('Acme Corp');
  expect(jobQuestion.text).not.toContain('{{');
}
```

### TS-11-04: Performance - Parallel Evaluation
```typescript
// Test: Evaluation completes within latency budget
import { evaluateQuestions } from '../src/questions/evaluator';
import { LoanState } from '../src/questions/types';

const state: LoanState = {
  proposalPid: 'TEST-001',
  version: 1,
  loadedAt: new Date(),
  fields: {},
  entities: {
    borrowers: [{ pid: 'b1', name: 'John', fields: {} }],
    jobs: [], assets: [], liabilities: [], realEstateOwned: [],
  },
  answered: new Set(),
};

const start = performance.now();
await evaluateQuestions(state);
const elapsed = performance.now() - start;

// Should complete in < 10ms for hot path
expect(elapsed).toBeLessThan(10);
```

### TS-11-05: Latency Budget Enforcement
```typescript
// Test: Returns partial results if budget exceeded
import { evaluateQuestions, setLatencyBudget } from '../src/questions/evaluator';
import { LoanState } from '../src/questions/types';

// Set very tight budget
setLatencyBudget(1); // 1ms

const state: LoanState = {
  proposalPid: 'TEST-001',
  version: 1,
  loadedAt: new Date(),
  fields: {},
  entities: { borrowers: [], jobs: [], assets: [], liabilities: [], realEstateOwned: [] },
  answered: new Set(),
};

const results = await evaluateQuestions(state);

// Should still return (possibly partial) results, not timeout
expect(Array.isArray(results)).toBe(true);

// Reset budget
setLatencyBudget(8);
```

---

## Implementation

### src/questions/evaluator.ts
```typescript
import {
  QuestionConfig,
  LoanState,
  EntityLevel,
  QuestionQueueItem,
  EntityRef,
} from './types';
import { getQuestionsByLevel } from './config-loader';
import { evaluateRulesParallel } from '../rules/engine';
import { logger } from '../utils/logger';
import { measure, recordLatency } from '../utils/metrics';

// Latency budget in milliseconds
let LATENCY_BUDGET_MS = 8;

/**
 * Set the latency budget for question evaluation.
 */
export function setLatencyBudget(ms: number): void {
  LATENCY_BUDGET_MS = ms;
}

const ALL_LEVELS: EntityLevel[] = [
  'PROPOSAL', 'BORROWER', 'JOB', 'ASSET',
  'LIABILITY', 'PROPERTY', 'REAL_ESTATE_OWNED'
];

/**
 * Evaluate all questions against loan state.
 * Returns questions that apply (criteria = true, not answered).
 */
export async function evaluateQuestions(loanState: LoanState): Promise<QuestionQueueItem[]> {
  return measure('evaluate_questions', async () => {
    const start = performance.now();
    const results: QuestionQueueItem[] = [];
    let budgetExceeded = false;

    for (const level of ALL_LEVELS) {
      // Check budget
      if (performance.now() - start > LATENCY_BUDGET_MS) {
        budgetExceeded = true;
        logger.warn('Latency budget exceeded during evaluation', {
          proposalPid: loanState.proposalPid,
          evaluatedLevels: ALL_LEVELS.indexOf(level),
          elapsedMs: performance.now() - start,
        });
        break;
      }

      const levelResults = await evaluateLevel(loanState, level, start);
      results.push(...levelResults);
    }

    if (budgetExceeded) {
      recordLatency('evaluate_budget_exceeded', 1);
    }

    logger.debug('Questions evaluated', {
      proposalPid: loanState.proposalPid,
      totalQuestions: results.length,
      durationMs: performance.now() - start,
    });

    return results;
  });
}

async function evaluateLevel(
  loanState: LoanState,
  level: EntityLevel,
  startTime: number
): Promise<QuestionQueueItem[]> {
  const questions = getQuestionsByLevel(level);
  if (questions.length === 0) return [];

  // Get entities for this level
  const entities = getEntitiesForLevel(loanState, level);
  if (entities.length === 0) return [];

  // Build evaluation batch
  const evaluations: Array<{
    question: QuestionConfig;
    entity: EntityRef | null;
    ruleId: string;
    context: Record<string, unknown>;
  }> = [];

  for (const question of questions) {
    // Skip answered
    if (loanState.answered.has(question.id)) continue;

    for (const entity of entities) {
      evaluations.push({
        question,
        entity,
        ruleId: `question:${question.id}`,
        context: buildContext(loanState, entity),
      });

      // Budget check within loop
      if (performance.now() - startTime > LATENCY_BUDGET_MS) {
        break;
      }
    }
  }

  if (evaluations.length === 0) return [];

  // Parallel evaluation
  const ruleResults = await evaluateRulesParallel(
    evaluations.map(e => ({ ruleId: e.ruleId, context: e.context }))
  );

  // Collect results
  const results: QuestionQueueItem[] = [];

  for (let i = 0; i < evaluations.length; i++) {
    if (ruleResults[i]) {
      const { question, entity } = evaluations[i];
      results.push(createQueueItem(question, entity, loanState));
    }
  }

  return results;
}

function getEntitiesForLevel(
  loanState: LoanState,
  level: EntityLevel
): (EntityRef | null)[] {
  switch (level) {
    case 'PROPOSAL':
    case 'PROPERTY':
      return [null]; // No entity context needed
    case 'BORROWER':
      return loanState.entities.borrowers;
    case 'JOB':
      return loanState.entities.jobs;
    case 'ASSET':
      return loanState.entities.assets;
    case 'LIABILITY':
      return loanState.entities.liabilities;
    case 'REAL_ESTATE_OWNED':
      return loanState.entities.realEstateOwned;
    default:
      return [null];
  }
}

function buildContext(
  loanState: LoanState,
  entity: EntityRef | null
): Record<string, unknown> {
  // Start with loan-level fields
  const context: Record<string, unknown> = { ...loanState.fields };

  // Overlay entity fields
  if (entity) {
    Object.assign(context, entity.fields);
  }

  return context;
}

function createQueueItem(
  question: QuestionConfig,
  entity: EntityRef | null,
  loanState: LoanState
): QuestionQueueItem {
  return {
    id: question.id,
    name: question.name,
    section: question.section,
    ordinal: question.ordinal,
    level: question.level,
    entityPid: entity?.pid,
    entityName: entity?.name,
    text: evaluateMergeFields(question.instructions, loanState, entity),
    type: question.type,
    options: question.options,
    accessField: question.formFields[0]?.accessField || '',
    flexibility: question.flexibility,
  };
}

function evaluateMergeFields(
  template: string,
  loanState: LoanState,
  entity: EntityRef | null
): string {
  return template.replace(/\{\{(.+?)\}\}/g, (match, fieldName) => {
    const normalized = fieldName.trim().toLowerCase().replace(/\s+/g, '_');

    // Check entity fields first
    if (entity?.fields[normalized] !== undefined) {
      return String(entity.fields[normalized]);
    }

    // Then loan-level fields
    if (loanState.fields[normalized] !== undefined) {
      return String(loanState.fields[normalized]);
    }

    // Keep original if not found
    return match;
  });
}
```

---

## Security Considerations

- [ ] Context doesn't leak between evaluations
- [ ] No PII in logs

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Evaluate 100 questions | < 5ms |
| Per-rule evaluation | < 0.1ms |
| Total with 2 borrowers | < 8ms |

---

## Error Handling

| Error | Handling |
|-------|----------|
| Rule not found | Log warning, skip question |
| Evaluation error | Log, skip question |
| Budget exceeded | Return partial results |

---

## Notes

- Uses `evaluateRulesParallel` for batch evaluation
- Pre-indexed questions by level avoid iteration overhead
- Budget enforcement prevents runaway latency
- Merge field replacement happens after rule evaluation
