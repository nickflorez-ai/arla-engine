# T09: ANTLR-to-Zen Compiler

## Overview
Convert Octane ANTLR criteria strings to GoRules Zen Engine JSON format. This is the bridge between human-readable rules and the runtime engine.

## Dependencies
- T08: Zen Engine Wrapper

## Deliverables
- `src/rules/compiler.ts` - ANTLR to Zen converter

---

## Acceptance Criteria

- [ ] Handles empty criteria (always true)
- [ ] Handles single conditions
- [ ] Handles AND rules (Matches all)
- [ ] Handles OR rules (Matches any)
- [ ] Converts field names to snake_case
- [ ] Converts values to uppercase
- [ ] Throws on unrecognized patterns

---

## Test Specifications

### TS-09-01: Empty Criteria (Always True)
```typescript
// Test: Empty string produces always-true rule
import { antlrToZen } from '../src/rules/compiler';

const rule = antlrToZen('');
expect(rule.nodes).toBeDefined();
expect(rule.edges).toBeDefined();

// Should have decision node with no conditions
const decision = rule.nodes.find(n => n.type === 'decisionNode');
expect(decision?.content?.rules?.length || 0).toBe(0);
```

### TS-09-02: Single Condition - Equals
```typescript
// Test: "Field is Value" pattern
import { antlrToZen } from '../src/rules/compiler';

const rule = antlrToZen('Citizenship Type is US Citizen');
const decision = rule.nodes.find(n => n.type === 'decisionNode');

expect(decision?.content?.rules[0]?.conditions?.citizenship_type?.value).toBe('US_CITIZEN');
expect(decision?.content?.rules[0]?.conditions?.citizenship_type?.operator).toBe('==');
```

### TS-09-03: Single Condition - Not Equals
```typescript
// Test: "Field is not Value" pattern
import { antlrToZen } from '../src/rules/compiler';

const rule = antlrToZen('Loan Type is not FHA');
const decision = rule.nodes.find(n => n.type === 'decisionNode');

expect(decision?.content?.rules[0]?.conditions?.loan_type?.value).toBe('FHA');
expect(decision?.content?.rules[0]?.conditions?.loan_type?.operator).toBe('!=');
```

### TS-09-04: AND Rule
```typescript
// Test: "Matches all" pattern
import { antlrToZen } from '../src/rules/compiler';

const criteria = `Matches all of the following rules:
    Citizenship Type is Non-Permanent Resident
    Visa Type is H-1B`;

const rule = antlrToZen(criteria);
const decision = rule.nodes.find(n => n.type === 'decisionNode');

// AND rule has single rule with multiple conditions
expect(decision?.content?.rules.length).toBe(1);
expect(decision?.content?.rules[0]?.conditions?.citizenship_type).toBeDefined();
expect(decision?.content?.rules[0]?.conditions?.visa_type).toBeDefined();
```

### TS-09-05: OR Rule
```typescript
// Test: "Matches any" pattern
import { antlrToZen } from '../src/rules/compiler';

const criteria = `Matches any of the following rules:
    Loan Purpose is Purchase
    Loan Purpose is Refinance`;

const rule = antlrToZen(criteria);
const decision = rule.nodes.find(n => n.type === 'decisionNode');

// OR rule has multiple rules with single condition each
expect(decision?.content?.rules.length).toBe(2);
```

### TS-09-06: Numeric Comparisons
```typescript
// Test: Greater/less than patterns
import { antlrToZen } from '../src/rules/compiler';

const rule = antlrToZen('Loan Amount > 500000');
const decision = rule.nodes.find(n => n.type === 'decisionNode');

expect(decision?.content?.rules[0]?.conditions?.loan_amount?.operator).toBe('>');
expect(decision?.content?.rules[0]?.conditions?.loan_amount?.value).toBe(500000);
```

### TS-09-07: Is Not Set
```typescript
// Test: "is not set" pattern
import { antlrToZen } from '../src/rules/compiler';

const rule = antlrToZen('Employment End Date is not set');
const decision = rule.nodes.find(n => n.type === 'decisionNode');

expect(decision?.content?.rules[0]?.conditions?.employment_end_date?.value).toBeNull();
```

### TS-09-08: Field Name Normalization
```typescript
// Test: Field names converted to snake_case
import { antlrToZen } from '../src/rules/compiler';

const rule = antlrToZen('Borrower Birthdate is 1990-01-01');
const decision = rule.nodes.find(n => n.type === 'decisionNode');

expect(decision?.content?.rules[0]?.conditions?.borrower_birthdate).toBeDefined();
```

### TS-09-09: Integration with Zen Engine
```typescript
// Test: Compiled rule evaluates correctly
import { antlrToZen } from '../src/rules/compiler';
import { initRulesEngine, compileRule, evaluateRule, closeRulesEngine } from '../src/rules/engine';

initRulesEngine();

const zenRule = antlrToZen('Loan Type is Conventional');
await compileRule('test-rule', zenRule);

const result1 = await evaluateRule('test-rule', { loan_type: 'CONVENTIONAL' });
expect(result1).toBe(true);

const result2 = await evaluateRule('test-rule', { loan_type: 'FHA' });
expect(result2).toBe(false);

closeRulesEngine();
```

---

## Implementation

### src/rules/compiler.ts
```typescript
import { logger } from '../utils/logger';

export interface ZenRule {
  nodes: ZenNode[];
  edges: ZenEdge[];
}

interface ZenNode {
  id: string;
  type: string;
  content?: any;
  position?: { x: number; y: number };
}

interface ZenEdge {
  id: string;
  sourceId: string;
  targetId: string;
}

interface ParsedCondition {
  field: string;
  operator: string;
  value: unknown;
}

/**
 * Convert Octane ANTLR criteria to Zen Engine JSON format.
 */
export function antlrToZen(antlrCriteria: string): ZenRule {
  if (!antlrCriteria || antlrCriteria.trim() === '') {
    return createAlwaysTrueRule();
  }

  const lines = antlrCriteria
    .split('\n')
    .map(l => l.trim())
    .filter(Boolean);

  const firstLine = lines[0];

  if (firstLine.startsWith('Matches all of the following rules:')) {
    return parseAndRule(lines.slice(1));
  }

  if (firstLine.startsWith('Matches any of the following rules:')) {
    return parseOrRule(lines.slice(1));
  }

  // Single condition
  return parseSingleCondition(firstLine);
}

function createBaseStructure(): { nodes: ZenNode[]; edges: ZenEdge[] } {
  return {
    nodes: [
      { id: 'input', type: 'inputNode', position: { x: 0, y: 0 } },
      { id: 'output', type: 'outputNode', position: { x: 400, y: 0 } },
    ],
    edges: [
      { id: 'e1', sourceId: 'input', targetId: 'decision' },
      { id: 'e2', sourceId: 'decision', targetId: 'output' },
    ],
  };
}

function createAlwaysTrueRule(): ZenRule {
  const base = createBaseStructure();
  base.nodes.splice(1, 0, {
    id: 'decision',
    type: 'decisionNode',
    content: {
      hitPolicy: 'first',
      rules: [],
    },
    position: { x: 200, y: 0 },
  });
  return base;
}

function parseAndRule(conditionLines: string[]): ZenRule {
  const conditions = conditionLines.map(parseConditionLine);
  
  // AND: single rule with all conditions
  const conditionsMap: Record<string, ParsedCondition> = {};
  for (const c of conditions) {
    conditionsMap[c.field] = c;
  }

  const base = createBaseStructure();
  base.nodes.splice(1, 0, {
    id: 'decision',
    type: 'decisionNode',
    content: {
      hitPolicy: 'first',
      rules: [{
        _id: 'rule1',
        _description: 'AND rule',
        conditions: conditionsMap,
        output: { result: true },
      }],
    },
    position: { x: 200, y: 0 },
  });

  return base;
}

function parseOrRule(conditionLines: string[]): ZenRule {
  const conditions = conditionLines.map(parseConditionLine);
  
  // OR: multiple rules with single condition each
  const rules = conditions.map((c, i) => ({
    _id: `rule${i + 1}`,
    conditions: { [c.field]: c },
    output: { result: true },
  }));

  const base = createBaseStructure();
  base.nodes.splice(1, 0, {
    id: 'decision',
    type: 'decisionNode',
    content: {
      hitPolicy: 'first',
      rules,
    },
    position: { x: 200, y: 0 },
  });

  return base;
}

function parseSingleCondition(line: string): ZenRule {
  const condition = parseConditionLine(line);

  const base = createBaseStructure();
  base.nodes.splice(1, 0, {
    id: 'decision',
    type: 'decisionNode',
    content: {
      hitPolicy: 'first',
      rules: [{
        _id: 'rule1',
        conditions: { [condition.field]: condition },
        output: { result: true },
      }],
    },
    position: { x: 200, y: 0 },
  });

  return base;
}

function parseConditionLine(line: string): ParsedCondition {
  // "is not set"
  const isNotSet = line.match(/^(.+?)\s+is\s+not\s+set$/i);
  if (isNotSet) {
    return {
      field: normalizeFieldName(isNotSet[1]),
      operator: '==',
      value: null,
    };
  }

  // "is not Value"
  const isNot = line.match(/^(.+?)\s+is\s+not\s+(.+)$/i);
  if (isNot) {
    return {
      field: normalizeFieldName(isNot[1]),
      operator: '!=',
      value: normalizeValue(isNot[2]),
    };
  }

  // "is Value"
  const isEquals = line.match(/^(.+?)\s+is\s+(.+)$/i);
  if (isEquals) {
    return {
      field: normalizeFieldName(isEquals[1]),
      operator: '==',
      value: normalizeValue(isEquals[2]),
    };
  }

  // Numeric comparisons: >=, <=, >, <
  const comparison = line.match(/^(.+?)\s*(>=|<=|>|<)\s*(.+)$/);
  if (comparison) {
    const numValue = parseFloat(comparison[3]);
    if (isNaN(numValue)) {
      throw new Error(`Invalid numeric value in: ${line}`);
    }
    return {
      field: normalizeFieldName(comparison[1]),
      operator: comparison[2],
      value: numValue,
    };
  }

  // Unrecognized pattern
  logger.warn('Unrecognized ANTLR pattern, treating as boolean check', { line });
  return {
    field: normalizeFieldName(line),
    operator: '==',
    value: true,
  };
}

function normalizeFieldName(name: string): string {
  return name
    .trim()
    .toLowerCase()
    .replace(/\s+/g, '_')
    .replace(/-/g, '_');
}

function normalizeValue(value: string): string | number | boolean {
  const trimmed = value.trim();
  
  // Boolean
  if (trimmed.toLowerCase() === 'true') return true;
  if (trimmed.toLowerCase() === 'false') return false;
  
  // Number
  const num = parseFloat(trimmed);
  if (!isNaN(num) && trimmed.match(/^-?\d+(\.\d+)?$/)) {
    return num;
  }
  
  // String - uppercase and normalize
  return trimmed
    .toUpperCase()
    .replace(/\s+/g, '_')
    .replace(/-/g, '_');
}

/**
 * Validate that a criteria string can be parsed.
 */
export function validateCriteria(antlrCriteria: string): { valid: boolean; error?: string } {
  try {
    antlrToZen(antlrCriteria);
    return { valid: true };
  } catch (error) {
    return { valid: false, error: (error as Error).message };
  }
}
```

---

## Security Considerations

- [ ] No code execution from criteria strings
- [ ] Input sanitized before parsing
- [ ] Invalid patterns logged but don't crash

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Parse single condition | < 0.1ms |
| Parse complex rule | < 1ms |

---

## Error Handling

| Error | Handling |
|-------|----------|
| Empty string | Return always-true rule |
| Invalid numeric | Throw error |
| Unrecognized pattern | Warn, treat as boolean |
| Null/undefined | Return always-true rule |

---

## Known Limitations

The current compiler handles:
- ✅ `Field is Value`
- ✅ `Field is not Value`
- ✅ `Field is not set`
- ✅ `Field > Number`
- ✅ `Matches all of the following rules:`
- ✅ `Matches any of the following rules:`

Not yet supported:
- ❌ Nested AND/OR
- ❌ Date comparisons
- ❌ List operators (IN, CONTAINS)
- ❌ Complex expressions

---

## Notes

- This is a simplified parser. Full ANTLR coverage requires exporting from Octane's AST.
- `validateCriteria()` can be used to test criteria at config load time
- Unrecognized patterns warn but don't fail - log these for manual review
