# T02: Type Definitions

## Overview
Define all TypeScript interfaces and types used throughout the application. This is the single source of truth for data structures.

## Dependencies
- T01: Project Setup

## Deliverables
- `src/questions/types.ts` - All type definitions

---

## Acceptance Criteria

- [ ] All entity types defined (Section, Question, LoanState, etc.)
- [ ] All enum types defined (EntityLevel, QuestionType, Flexibility)
- [ ] Types are strict (no `any` except where unavoidable)
- [ ] JSDoc comments on all public types
- [ ] Compiles without errors

---

## Test Specifications

### TS-02-01: Type Compilation
```typescript
// Test: Types compile and can be imported
import { 
  EntityLevel, 
  QuestionType, 
  LoanState, 
  QuestionConfig,
  QuestionQueue 
} from '../src/questions/types';

// Should not throw
const level: EntityLevel = 'BORROWER';
const type: QuestionType = 'single_choice';
```

### TS-02-02: LoanState Serialization Round-Trip
```typescript
// Test: LoanState can be serialized and deserialized
import { LoanState } from '../src/questions/types';
import { encode, decode } from '@msgpack/msgpack';

const state: LoanState = {
  proposalPid: 'test-123',
  version: 1,
  loadedAt: new Date(),
  fields: { citizenship_type: 'US_CITIZEN' },
  entities: {
    borrowers: [{ pid: 'b1', name: 'John', fields: {} }],
    jobs: [],
    assets: [],
    liabilities: [],
    realEstateOwned: []
  },
  answered: new Set(['Q100'])
};

// Serialize (convert Set to Array for msgpack)
const serializable = {
  ...state,
  answered: Array.from(state.answered),
  loadedAt: state.loadedAt.toISOString()
};
const packed = encode(serializable);
const unpacked = decode(packed);

// Deserialize
const restored: LoanState = {
  ...unpacked,
  answered: new Set(unpacked.answered),
  loadedAt: new Date(unpacked.loadedAt)
};

expect(restored.proposalPid).toBe('test-123');
expect(restored.answered.has('Q100')).toBe(true);
```

### TS-02-03: QuestionConfig Validation
```typescript
// Test: QuestionConfig has all required fields
import { QuestionConfig } from '../src/questions/types';

const config: QuestionConfig = {
  id: 'Q100',
  name: 'Citizenship',
  section: 'borrower_info',
  ordinal: 100,
  level: 'BORROWER',
  instructions: 'What is your citizenship?',
  type: 'single_choice',
  formFields: [{
    order: 1,
    label: 'Citizenship Type',
    accessField: 'citizenship_type',
    prepopulate: false
  }],
  criteriaZen: {},
  flexibility: 'exact'
};

// TypeScript will error if any required field is missing
expect(config.id).toBe('Q100');
```

---

## Implementation

### src/questions/types.ts
```typescript
/**
 * Entity levels in the loan hierarchy.
 * Determines what context is available for rule evaluation.
 */
export type EntityLevel =
  | 'PROPOSAL'
  | 'BORROWER'
  | 'JOB'
  | 'ASSET'
  | 'LIABILITY'
  | 'PROPERTY'
  | 'REAL_ESTATE_OWNED';

/**
 * Question input types.
 */
export type QuestionType =
  | 'single_choice'
  | 'multi_choice'
  | 'text'
  | 'number'
  | 'date'
  | 'currency'
  | 'boolean';

/**
 * How strictly the AI should match answers.
 */
export type Flexibility = 
  | 'exact'           // Must match exactly (SSN, dates)
  | 'conversational'  // Can be extracted from natural language
  | 'inferred'        // Can be inferred from context
  | 'optional';       // Not required

/**
 * Section definition for grouping questions.
 */
export interface Section {
  id: string;
  name: string;
  sequence: number;
  description?: string;
}

/**
 * Form field mapping to Octane access fields.
 */
export interface FormField {
  order: number;
  label: string;
  accessField: string;
  prepopulate: boolean;
}

/**
 * Option for single/multi choice questions.
 */
export interface Option {
  value: string;
  label: string;
}

/**
 * Complete question configuration.
 */
export interface QuestionConfig {
  id: string;
  name: string;
  section: string;
  ordinal: number;
  level: EntityLevel;
  instructions: string;
  type: QuestionType;
  formFields: FormField[];
  criteriaZen: object;
  flexibility: Flexibility;
  options?: Option[];
  canCombineWith?: string[];
  extractionHints?: string[];
}

/**
 * Reference to an entity (borrower, job, asset, etc.)
 */
export interface EntityRef {
  pid: string;
  name: string;
  fields: Record<string, unknown>;
}

/**
 * Complete loan state cached in Redis.
 */
export interface LoanState {
  proposalPid: string;
  version: number;
  loadedAt: Date;
  fields: Record<string, unknown>;
  entities: {
    borrowers: EntityRef[];
    jobs: EntityRef[];
    assets: EntityRef[];
    liabilities: EntityRef[];
    realEstateOwned: EntityRef[];
  };
  answered: Set<string>;
}

/**
 * Serializable version of LoanState for Redis/msgpack.
 */
export interface LoanStateSerialized {
  proposalPid: string;
  version: number;
  loadedAt: string;
  fields: Record<string, unknown>;
  entities: {
    borrowers: EntityRef[];
    jobs: EntityRef[];
    assets: EntityRef[];
    liabilities: EntityRef[];
    realEstateOwned: EntityRef[];
  };
  answered: string[];
}

/**
 * Question ready for the queue (after evaluation).
 */
export interface QuestionQueueItem {
  id: string;
  name: string;
  section: string;
  ordinal: number;
  level: EntityLevel;
  entityPid?: string;
  entityName?: string;
  text: string;
  type: QuestionType;
  options?: Option[];
  accessField: string;
  flexibility: Flexibility;
}

/**
 * Section progress for UI display.
 */
export interface SectionProgress {
  id: string;
  name: string;
  sequence: number;
  total: number;
  answered: number;
  status: 'pending' | 'in_progress' | 'complete';
}

/**
 * Complete question queue response.
 */
export interface QuestionQueue {
  loanId: string;
  timestamp: Date;
  stateVersion: number;
  sections: SectionProgress[];
  queue: QuestionQueueItem[];
  canAskTogether: string[][];
  nextRecommended: string;
}

/**
 * Answer submission from AI agent.
 */
export interface AnswerSubmission {
  questionId: string;
  entityPid?: string;
  answer: unknown;
  rawInput?: string;
  confidence?: number;
}

/**
 * Audit log entry for compliance.
 */
export interface AuditEntry {
  timestamp: Date;
  agentId: string;
  sessionId: string;
  action: string;
  proposalPid: string;
  questionId?: string;
  fieldsModified?: string[];
  rawInput?: string;
  confidence?: number;
}
```

---

## Security Considerations

- [ ] No `any` types that could bypass type checking
- [ ] Sensitive fields (SSN) marked if needed
- [ ] AuditEntry type ensures compliance logging

---

## Performance Requirements

N/A - Type definitions only.

---

## Error Handling

N/A - Type definitions only.

---

## Notes

- Added `LoanStateSerialized` for msgpack serialization (Set → Array, Date → string)
- Added `AuditEntry` for compliance logging
- All types have JSDoc comments for IDE support
