# T05: Loan State Loader

## Overview
Load complete loan state from Aurora database, flattening entity hierarchies into a format suitable for rule evaluation.

## Dependencies
- T04: Aurora Connection Pool

## Deliverables
- `src/db/loader.ts` - Loan state loader from Aurora

---

## Acceptance Criteria

- [ ] Loads proposal by PID
- [ ] Loads all borrowers for the deal
- [ ] Loads all entities (jobs, assets, liabilities, REO)
- [ ] Flattens snake_case to camelCase
- [ ] Loads answered questions
- [ ] Throws on proposal not found
- [ ] Tracks query latency

---

## Test Specifications

### TS-05-01: Load Valid Proposal
```typescript
// Test: Loads proposal with all entities
// Requires: Seeded test database with proposal PID 'TEST-001'
import { loadLoanStateFromAurora } from '../src/db/loader';

const state = await loadLoanStateFromAurora('TEST-001');

expect(state.proposalPid).toBe('TEST-001');
expect(state.fields).toBeDefined();
expect(state.entities.borrowers.length).toBeGreaterThan(0);
expect(state.answered).toBeInstanceOf(Set);
```

### TS-05-02: Not Found Throws
```typescript
// Test: Throws error for missing proposal
import { loadLoanStateFromAurora } from '../src/db/loader';

await expect(
  loadLoanStateFromAurora('DOES-NOT-EXIST')
).rejects.toThrow('Proposal not found');
```

### TS-05-03: Field Name Conversion
```typescript
// Test: snake_case converted to camelCase
import { loadLoanStateFromAurora } from '../src/db/loader';

const state = await loadLoanStateFromAurora('TEST-001');

// If DB has 'first_name', it should be 'firstName' in state
expect(state.entities.borrowers[0].fields.firstName).toBeDefined();
```

### TS-05-04: Entities Loaded
```typescript
// Test: All entity types loaded
import { loadLoanStateFromAurora } from '../src/db/loader';

const state = await loadLoanStateFromAurora('TEST-001');

expect(Array.isArray(state.entities.borrowers)).toBe(true);
expect(Array.isArray(state.entities.jobs)).toBe(true);
expect(Array.isArray(state.entities.assets)).toBe(true);
expect(Array.isArray(state.entities.liabilities)).toBe(true);
expect(Array.isArray(state.entities.realEstateOwned)).toBe(true);
```

### TS-05-05: Performance
```typescript
// Test: Load completes within 100ms
import { loadLoanStateFromAurora } from '../src/db/loader';

const start = performance.now();
await loadLoanStateFromAurora('TEST-001');
const elapsed = performance.now() - start;

expect(elapsed).toBeLessThan(100);
```

---

## Implementation

### src/db/loader.ts
```typescript
import { query } from './aurora';
import { LoanState, EntityRef } from '../questions/types';
import { logger } from '../utils/logger';
import { measure } from '../utils/metrics';

/**
 * Load complete loan state from Aurora.
 * Called on Redis cache miss.
 */
export async function loadLoanStateFromAurora(proposalPid: string): Promise<LoanState> {
  return measure('aurora_load_loan_state', async () => {
    const start = performance.now();

    // Load proposal
    const proposals = await query<any>(
      'SELECT * FROM proposal WHERE pid = ?',
      [proposalPid]
    );

    if (proposals.length === 0) {
      throw new Error(`Proposal not found: ${proposalPid}`);
    }

    const proposal = proposals[0];
    const dealPid = proposal.deal_pid;

    // Load borrowers
    const borrowerRows = await query<any>(
      'SELECT * FROM borrower WHERE deal_pid = ?',
      [dealPid]
    );

    const borrowers: EntityRef[] = borrowerRows.map((r: any) => ({
      pid: r.pid,
      name: formatBorrowerName(r),
      fields: flattenRow(r),
    }));

    const borrowerPids = borrowers.map(b => b.pid);

    // Load child entities in parallel
    const [jobs, assets, liabilities, realEstateOwned] = await Promise.all([
      loadJobs(borrowerPids),
      loadAssets(borrowerPids),
      loadLiabilities(borrowerPids),
      loadRealEstateOwned(borrowerPids),
    ]);

    // Load property
    const propertyRows = await query<any>(
      'SELECT * FROM property WHERE deal_pid = ?',
      [dealPid]
    );

    // Build flat fields map
    const fields: Record<string, unknown> = {
      ...flattenRow(proposal),
      ...(propertyRows[0] ? prefixFields('property', flattenRow(propertyRows[0])) : {}),
    };

    // Load answered questions
    const answeredRows = await query<any>(
      'SELECT DISTINCT question_pid FROM deal_question_form_response WHERE deal_pid = ?',
      [dealPid]
    );
    const answered = new Set<string>(answeredRows.map((r: any) => r.question_pid));

    const duration = performance.now() - start;
    logger.info('Loan state loaded from Aurora', {
      proposalPid,
      borrowerCount: borrowers.length,
      jobCount: jobs.length,
      assetCount: assets.length,
      durationMs: Math.round(duration),
    });

    return {
      proposalPid,
      version: Date.now(),
      loadedAt: new Date(),
      fields,
      entities: { borrowers, jobs, assets, liabilities, realEstateOwned },
      answered,
    };
  });
}

async function loadJobs(borrowerPids: string[]): Promise<EntityRef[]> {
  if (borrowerPids.length === 0) return [];
  
  const placeholders = borrowerPids.map(() => '?').join(',');
  const rows = await query<any>(
    `SELECT * FROM job WHERE borrower_pid IN (${placeholders})`,
    borrowerPids
  );
  
  return rows.map((r: any) => ({
    pid: r.pid,
    name: r.employer_name || 'Unknown Employer',
    fields: flattenRow(r),
  }));
}

async function loadAssets(borrowerPids: string[]): Promise<EntityRef[]> {
  if (borrowerPids.length === 0) return [];
  
  const placeholders = borrowerPids.map(() => '?').join(',');
  const rows = await query<any>(
    `SELECT * FROM asset WHERE borrower_pid IN (${placeholders})`,
    borrowerPids
  );
  
  return rows.map((r: any) => ({
    pid: r.pid,
    name: r.institution_name || 'Unknown Institution',
    fields: flattenRow(r),
  }));
}

async function loadLiabilities(borrowerPids: string[]): Promise<EntityRef[]> {
  if (borrowerPids.length === 0) return [];
  
  const placeholders = borrowerPids.map(() => '?').join(',');
  const rows = await query<any>(
    `SELECT * FROM liability WHERE borrower_pid IN (${placeholders})`,
    borrowerPids
  );
  
  return rows.map((r: any) => ({
    pid: r.pid,
    name: r.creditor_name || 'Unknown Creditor',
    fields: flattenRow(r),
  }));
}

async function loadRealEstateOwned(borrowerPids: string[]): Promise<EntityRef[]> {
  if (borrowerPids.length === 0) return [];
  
  const placeholders = borrowerPids.map(() => '?').join(',');
  const rows = await query<any>(
    `SELECT * FROM real_estate_owned WHERE borrower_pid IN (${placeholders})`,
    borrowerPids
  );
  
  return rows.map((r: any) => ({
    pid: r.pid,
    name: r.street_address || 'Unknown Address',
    fields: flattenRow(r),
  }));
}

/**
 * Convert snake_case row to camelCase fields.
 */
function flattenRow(row: Record<string, unknown>): Record<string, unknown> {
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(row)) {
    result[snakeToCamel(key)] = value;
  }
  return result;
}

function snakeToCamel(str: string): string {
  return str.replace(/_([a-z])/g, (_, letter) => letter.toUpperCase());
}

function formatBorrowerName(row: any): string {
  const first = row.first_name || '';
  const last = row.last_name || '';
  return `${first} ${last}`.trim() || 'Unknown Borrower';
}

function prefixFields(
  prefix: string,
  fields: Record<string, unknown>
): Record<string, unknown> {
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(fields)) {
    result[`${prefix}_${key}`] = value;
  }
  return result;
}
```

---

## Security Considerations

- [ ] All queries use parameterized inputs (no SQL injection)
- [ ] PII fields loaded but not logged
- [ ] Access controlled at API layer

---

## Performance Requirements

| Metric | Target |
|--------|--------|
| Full state load | < 100ms |
| Entity queries | Parallel execution |
| Field flattening | < 1ms |

---

## Error Handling

| Error | Handling |
|-------|----------|
| Proposal not found | Throw with clear message |
| DB query fails | Let error bubble up (logged at caller) |
| Empty entities | Return empty array (not error) |

---

## Notes

- Entity loads run in parallel via `Promise.all()`
- Property fields prefixed with `property_` to avoid collisions
- Returns `Set<string>` for answered questions (efficient lookup)
