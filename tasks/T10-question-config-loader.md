# T10: Question Config Loader

## Overview
Load question and section configurations from YAML files, compile rules at startup, and index for fast lookup.

## Dependencies
- T02: Type Definitions
- T09: ANTLR-to-Zen Compiler

## Deliverables
- `src/questions/config-loader.ts` - Config loader with pre-indexing

---

## Acceptance Criteria

- [ ] Loads sections from `config/sections/*.yaml`
- [ ] Loads questions from `config/questions/**/*.yaml`
- [ ] Compiles all criteria to Zen rules
- [ ] Fails startup on invalid criteria
- [ ] Pre-indexes questions by entity level
- [ ] Provides fast lookup functions

---

## Test Specifications

### TS-10-01: Load Sections
```typescript
// Test: Sections load from YAML
import { loadAllConfigs, getAllSections } from '../src/questions/config-loader';

await loadAllConfigs();
const sections = getAllSections();

expect(sections.length).toBeGreaterThan(0);
expect(sections[0].id).toBeDefined();
expect(sections[0].name).toBeDefined();
expect(sections[0].sequence).toBeDefined();
```

### TS-10-02: Load Questions
```typescript
// Test: Questions load from YAML
import { loadAllConfigs, getAllQuestions, getQuestion } from '../src/questions/config-loader';

await loadAllConfigs();
const questions = getAllQuestions();

expect(questions.length).toBeGreaterThan(0);

const q = getQuestion(questions[0].id);
expect(q).toBeDefined();
expect(q?.instructions).toBeDefined();
```

### TS-10-03: Questions Indexed by Level
```typescript
// Test: Questions indexed by entity level
import { loadAllConfigs, getQuestionsByLevel } from '../src/questions/config-loader';

await loadAllConfigs();

const borrowerQuestions = getQuestionsByLevel('BORROWER');
expect(Array.isArray(borrowerQuestions)).toBe(true);

for (const q of borrowerQuestions) {
  expect(q.level).toBe('BORROWER');
}
```

### TS-10-04: Sections Sorted by Sequence
```typescript
// Test: Sections returned in sequence order
import { loadAllConfigs, getAllSections } from '../src/questions/config-loader';

await loadAllConfigs();
const sections = getAllSections();

for (let i = 1; i < sections.length; i++) {
  expect(sections[i].sequence).toBeGreaterThanOrEqual(sections[i - 1].sequence);
}
```

### TS-10-05: Invalid Criteria Fails Startup
```typescript
// Test: Invalid criteria throws during load
// Setup: Create a question file with invalid criteria
import { loadAllConfigs } from '../src/questions/config-loader';

// Assuming there's a test fixture with invalid criteria
await expect(loadAllConfigs('./test-fixtures/invalid-config')).rejects.toThrow();
```

### TS-10-06: Rule Compiled for Each Question
```typescript
// Test: Each question has a compiled rule
import { loadAllConfigs, getAllQuestions } from '../src/questions/config-loader';
import { isRuleCompiled } from '../src/rules/engine';

await loadAllConfigs();
const questions = getAllQuestions();

for (const q of questions) {
  expect(isRuleCompiled(`question:${q.id}`)).toBe(true);
}
```

### TS-10-07: Performance
```typescript
// Test: Config loading completes quickly
import { loadAllConfigs } from '../src/questions/config-loader';

const start = performance.now();
await loadAllConfigs();
const elapsed = performance.now() - start;

// Should load 100+ questions in < 500ms
expect(elapsed).toBeLessThan(500);
```

---

## Implementation

### src/questions/config-loader.ts
```typescript
import fs from 'fs';
import path from 'path';
import yaml from 'yaml';
import { config } from '../config';
import { QuestionConfig, Section, FormField, EntityLevel } from './types';
import { antlrToZen, validateCriteria } from '../rules/compiler';
import { initRulesEngine, compileRule, getCompiledRuleCount } from '../rules/engine';
import { logger } from '../utils/logger';

// Storage
const sections = new Map<string, Section>();
const questions = new Map<string, QuestionConfig>();
const questionsByLevel = new Map<EntityLevel, QuestionConfig[]>();

// Entity levels for indexing
const ALL_LEVELS: EntityLevel[] = [
  'PROPOSAL', 'BORROWER', 'JOB', 'ASSET',
  'LIABILITY', 'PROPERTY', 'REAL_ESTATE_OWNED'
];

/**
 * Load all configs and compile rules.
 * MUST be called at startup before serving requests.
 */
export async function loadAllConfigs(configPath?: string): Promise<void> {
  const basePath = configPath || config.configPath;
  const start = performance.now();

  // Initialize rules engine
  initRulesEngine();

  // Clear existing
  sections.clear();
  questions.clear();
  for (const level of ALL_LEVELS) {
    questionsByLevel.set(level, []);
  }

  // Load sections
  const sectionsPath = path.join(basePath, 'sections');
  if (fs.existsSync(sectionsPath)) {
    const files = fs.readdirSync(sectionsPath).filter(f => f.endsWith('.yaml'));
    for (const file of files) {
      const section = loadSectionFile(path.join(sectionsPath, file));
      sections.set(section.id, section);
    }
  }

  // Load questions recursively
  const questionsPath = path.join(basePath, 'questions');
  if (fs.existsSync(questionsPath)) {
    await loadQuestionsRecursive(questionsPath);
  }

  const duration = performance.now() - start;
  logger.info('Configs loaded', {
    sections: sections.size,
    questions: questions.size,
    rules: getCompiledRuleCount(),
    durationMs: Math.round(duration),
  });
}

function loadSectionFile(filePath: string): Section {
  const content = fs.readFileSync(filePath, 'utf-8');
  const parsed = yaml.parse(content);

  if (!parsed.id || !parsed.name || parsed.sequence === undefined) {
    throw new Error(`Invalid section file: ${filePath}`);
  }

  return {
    id: parsed.id,
    name: parsed.name,
    sequence: parsed.sequence,
    description: parsed.description,
  };
}

async function loadQuestionsRecursive(dirPath: string): Promise<void> {
  const entries = fs.readdirSync(dirPath, { withFileTypes: true });

  for (const entry of entries) {
    const fullPath = path.join(dirPath, entry.name);

    if (entry.isDirectory()) {
      await loadQuestionsRecursive(fullPath);
    } else if (entry.name.endsWith('.yaml')) {
      const question = await loadQuestionFile(fullPath);
      questions.set(question.id, question);

      // Index by level
      const levelList = questionsByLevel.get(question.level) || [];
      levelList.push(question);
      questionsByLevel.set(question.level, levelList);
    }
  }
}

async function loadQuestionFile(filePath: string): Promise<QuestionConfig> {
  const content = fs.readFileSync(filePath, 'utf-8');
  const parsed = yaml.parse(content);

  // Validate required fields
  if (!parsed.id || !parsed.name || !parsed.section || parsed.ordinal === undefined || !parsed.level) {
    throw new Error(`Invalid question file (missing required fields): ${filePath}`);
  }

  // Validate criteria
  const criteriaStr = parsed.criteria || '';
  const validation = validateCriteria(criteriaStr);
  if (!validation.valid) {
    throw new Error(`Invalid criteria in ${filePath}: ${validation.error}`);
  }

  // Convert and compile
  const criteriaZen = antlrToZen(criteriaStr);
  const ruleId = `question:${parsed.id}`;

  try {
    await compileRule(ruleId, criteriaZen);
  } catch (err) {
    throw new Error(`Failed to compile rule for ${parsed.id} in ${filePath}: ${(err as Error).message}`);
  }

  // Map form fields
  const formFields: FormField[] = (parsed.form_fields || []).map((f: any) => ({
    order: f.order,
    label: f.label,
    accessField: f.access_field,
    prepopulate: f.prepopulate ?? false,
  }));

  return {
    id: parsed.id,
    name: parsed.name,
    section: parsed.section,
    ordinal: parsed.ordinal,
    level: parsed.level,
    instructions: parsed.instructions || '',
    type: parsed.type || 'text',
    formFields,
    criteriaZen,
    flexibility: parsed.flexibility || 'conversational',
    options: parsed.options,
    canCombineWith: parsed.can_combine_with,
    extractionHints: parsed.extraction_hints,
  };
}

// ============ Lookup Functions ============

/**
 * Get all questions.
 */
export function getAllQuestions(): QuestionConfig[] {
  return Array.from(questions.values());
}

/**
 * Get a question by ID.
 */
export function getQuestion(id: string): QuestionConfig | undefined {
  return questions.get(id);
}

/**
 * Get questions by entity level (pre-indexed).
 */
export function getQuestionsByLevel(level: EntityLevel): QuestionConfig[] {
  return questionsByLevel.get(level) || [];
}

/**
 * Get all sections sorted by sequence.
 */
export function getAllSections(): Section[] {
  return Array.from(sections.values()).sort((a, b) => a.sequence - b.sequence);
}

/**
 * Get a section by ID.
 */
export function getSection(id: string): Section | undefined {
  return sections.get(id);
}

/**
 * Get count of loaded questions.
 */
export function getQuestionCount(): number {
  return questions.size;
}

/**
 * Get count of loaded sections.
 */
export function getSectionCount(): number {
  return sections.size;
}
```

---

## Security Considerations

- [ ] YAML parsed safely (yaml lib handles this)
- [ ] File paths validated
- [ ] No code execution from config files

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Load 100 questions | < 500ms |
| getQuestion() lookup | O(1) |
| getQuestionsByLevel() | O(1) |

---

## Error Handling

| Error | Handling |
|-------|----------|
| Missing required field | Throw with file path |
| Invalid criteria | Throw with details |
| File not found | Skip (not all dirs required) |
| YAML parse error | Throw with file path |

---

## Notes

- Questions pre-indexed by entity level for O(1) lookup
- Rule compilation fails hard at startup
- Empty criteria allowed (means always applicable)
- Form fields optional (some questions are info-only)
