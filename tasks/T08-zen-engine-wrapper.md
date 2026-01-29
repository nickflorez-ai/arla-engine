# T08: Zen Engine Wrapper

## Overview
Wrapper around GoRules Zen Engine for rule compilation and evaluation with proper error handling and metrics.

## Dependencies
- T03: Logger & Metrics

## Deliverables
- `src/rules/engine.ts` - Zen Engine wrapper

---

## Acceptance Criteria

- [ ] Initialize engine once at startup
- [ ] Compile rules to bytecode
- [ ] Evaluate rules with context
- [ ] Fail hard on invalid rules (no silent skip)
- [ ] Track evaluation latency
- [ ] Return compiled rule count

---

## Test Specifications

### TS-08-01: Engine Initialization
```typescript
// Test: Engine initializes without error
import { initRulesEngine, closeRulesEngine, getCompiledRuleCount } from '../src/rules/engine';

initRulesEngine();
expect(getCompiledRuleCount()).toBe(0);
closeRulesEngine();
```

### TS-08-02: Compile Valid Rule
```typescript
// Test: Valid rule compiles successfully
import { initRulesEngine, compileRule, getCompiledRuleCount, closeRulesEngine } from '../src/rules/engine';

initRulesEngine();

const rule = {
  nodes: [
    { id: 'input', type: 'inputNode' },
    { id: 'output', type: 'outputNode' },
  ],
  edges: [
    { id: 'e1', sourceId: 'input', targetId: 'output' },
  ],
};

await compileRule('test-rule', rule);
expect(getCompiledRuleCount()).toBe(1);

closeRulesEngine();
```

### TS-08-03: Compile Invalid Rule Throws
```typescript
// Test: Invalid rule throws error
import { initRulesEngine, compileRule, closeRulesEngine } from '../src/rules/engine';

initRulesEngine();

const invalidRule = { invalid: 'structure' };

await expect(compileRule('bad-rule', invalidRule)).rejects.toThrow();

closeRulesEngine();
```

### TS-08-04: Evaluate Rule
```typescript
// Test: Rule evaluation returns boolean
import { initRulesEngine, compileRule, evaluateRule, closeRulesEngine } from '../src/rules/engine';

initRulesEngine();

// Rule that checks if citizenship_type == 'US_CITIZEN'
const rule = {
  nodes: [
    { id: 'input', type: 'inputNode', position: { x: 0, y: 0 } },
    {
      id: 'decision',
      type: 'decisionNode',
      content: {
        hitPolicy: 'first',
        rules: [{
          _id: 'rule1',
          conditions: { citizenship_type: { operator: '==', value: 'US_CITIZEN' } },
          output: { result: true },
        }],
      },
      position: { x: 200, y: 0 },
    },
    { id: 'output', type: 'outputNode', position: { x: 400, y: 0 } },
  ],
  edges: [
    { id: 'e1', sourceId: 'input', targetId: 'decision' },
    { id: 'e2', sourceId: 'decision', targetId: 'output' },
  ],
};

await compileRule('citizenship-check', rule);

// Should match
const result1 = await evaluateRule('citizenship-check', { citizenship_type: 'US_CITIZEN' });
expect(result1).toBe(true);

// Should not match
const result2 = await evaluateRule('citizenship-check', { citizenship_type: 'NON_RESIDENT' });
expect(result2).toBe(false);

closeRulesEngine();
```

### TS-08-05: Evaluation Performance
```typescript
// Test: Rule evaluation is fast
import { initRulesEngine, compileRule, evaluateRule, closeRulesEngine } from '../src/rules/engine';

initRulesEngine();

const simpleRule = {
  nodes: [
    { id: 'input', type: 'inputNode' },
    { id: 'output', type: 'outputNode' },
  ],
  edges: [
    { id: 'e1', sourceId: 'input', targetId: 'output' },
  ],
};

await compileRule('perf-test', simpleRule);

const iterations = 1000;
const start = performance.now();

for (let i = 0; i < iterations; i++) {
  await evaluateRule('perf-test', { field: 'value' });
}

const elapsed = performance.now() - start;
const perEval = elapsed / iterations;

expect(perEval).toBeLessThan(0.1); // < 0.1ms per evaluation

closeRulesEngine();
```

### TS-08-06: Missing Rule Returns False
```typescript
// Test: Missing rule returns false, not error
import { initRulesEngine, evaluateRule, closeRulesEngine } from '../src/rules/engine';

initRulesEngine();

const result = await evaluateRule('non-existent-rule', {});
expect(result).toBe(false);

closeRulesEngine();
```

---

## Implementation

### src/rules/engine.ts
```typescript
import { ZenEngine, ZenDecision } from '@gorules/zen-engine';
import { logger } from '../utils/logger';
import { measure, recordLatency } from '../utils/metrics';

let engine: ZenEngine | null = null;
const compiledRules = new Map<string, ZenDecision>();

/**
 * Initialize the Zen rules engine.
 * Must be called before compiling or evaluating rules.
 */
export function initRulesEngine(): void {
  if (engine) {
    logger.warn('Rules engine already initialized');
    return;
  }
  
  engine = new ZenEngine();
  logger.info('Zen rules engine initialized');
}

/**
 * Compile a rule and store in memory.
 * FAILS HARD on invalid rules - this is intentional.
 * 
 * @throws Error if rule is invalid
 */
export async function compileRule(ruleId: string, ruleJson: object): Promise<void> {
  if (!engine) {
    throw new Error('Rules engine not initialized. Call initRulesEngine() first.');
  }

  const start = performance.now();

  try {
    const decision = await engine.createDecision(JSON.stringify(ruleJson));
    compiledRules.set(ruleId, decision);

    const elapsed = performance.now() - start;
    recordLatency('rule_compile', elapsed);

    logger.debug('Rule compiled', { ruleId, durationMs: elapsed });
  } catch (err) {
    const error = err as Error;
    logger.error('Failed to compile rule - FATAL', {
      ruleId,
      error: error.message,
    });
    
    // Fail hard - invalid rules should prevent startup
    throw new Error(`Rule compilation failed for ${ruleId}: ${error.message}`);
  }
}

/**
 * Evaluate a compiled rule against context.
 * Returns true if rule conditions are satisfied, false otherwise.
 * Returns false for missing rules (with warning).
 */
export async function evaluateRule(
  ruleId: string,
  context: Record<string, unknown>
): Promise<boolean> {
  const decision = compiledRules.get(ruleId);

  if (!decision) {
    logger.warn('Rule not found', { ruleId });
    return false;
  }

  return measure('rule_evaluate', async () => {
    try {
      const result = await decision.evaluate(context);
      
      // Zen Engine returns { result: <output> } or null
      // For our decision tables, output is { result: true/false }
      if (result && typeof result === 'object' && 'result' in result) {
        return result.result === true;
      }
      
      // Default case: rule matched but no explicit output
      return true;
    } catch (err) {
      logger.error('Rule evaluation failed', {
        ruleId,
        error: (err as Error).message,
      });
      return false;
    }
  });
}

/**
 * Evaluate multiple rules in parallel.
 */
export async function evaluateRulesParallel(
  evaluations: Array<{ ruleId: string; context: Record<string, unknown> }>
): Promise<boolean[]> {
  return Promise.all(
    evaluations.map(({ ruleId, context }) => evaluateRule(ruleId, context))
  );
}

/**
 * Get count of compiled rules.
 */
export function getCompiledRuleCount(): number {
  return compiledRules.size;
}

/**
 * Check if a rule is compiled.
 */
export function isRuleCompiled(ruleId: string): boolean {
  return compiledRules.has(ruleId);
}

/**
 * Clear all compiled rules and close engine.
 */
export function closeRulesEngine(): void {
  compiledRules.clear();
  engine = null;
  logger.info('Zen rules engine closed');
}

/**
 * Get all compiled rule IDs.
 */
export function getCompiledRuleIds(): string[] {
  return Array.from(compiledRules.keys());
}
```

---

## Security Considerations

- [ ] Rule content validated before compilation
- [ ] No user input directly in rules
- [ ] Context sanitized before evaluation

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Rule compilation | < 10ms per rule |
| Rule evaluation | < 0.1ms per rule |
| Memory per rule | < 10KB |

---

## Error Handling

| Error | Handling |
|-------|----------|
| Invalid rule JSON | Throw (fail startup) |
| Missing rule | Log warning, return false |
| Evaluation error | Log error, return false |
| Engine not init | Throw with clear message |

---

## Notes

- `compileRule` fails hard - this is critical for catching config errors at startup
- `evaluateRule` fails soft - individual question failures shouldn't crash the system
- `evaluateRulesParallel` enables batch evaluation for performance
- Zen Engine is ~38M evals/sec, so < 0.1ms per rule is expected
