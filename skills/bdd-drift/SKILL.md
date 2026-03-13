---
name: bdd-drift
description: Detect orphaned or mismatched BDD test files across unit, accessibility, and E2E test suites. Use when you want to verify that every .feature file has a matching .steps.tsx file and vice versa, or to find test files that were left behind after a component was deleted or renamed.
argument-hint: "[unit|a11y|e2e|all]"
allowed-tools: Read, Glob, Grep
---

# BDD Feature Drift Detection

Detect orphaned or mismatched BDD test files in the project. The argument `$0` specifies which test suite to check: `unit`, `a11y`, `e2e`, or `all` (default).

## What is feature drift?

Feature drift occurs when:
- A `.feature` file exists without a matching `.steps.tsx` file (the feature is defined but never implemented)
- A `.steps.tsx` file exists without a matching `.feature` file (step definitions exist for a feature that was deleted)
- A component was deleted or renamed but its test files were left behind
- An a11y test file exists for a component that no longer exists

Drift makes the test suite unreliable — orphaned files create confusion, and unmapped features give the false impression that behaviors are tested when they're not.

## Workflow

### Step 1: Scan for file pairs

Based on `$0` (defaults to `all`), scan the appropriate directories:

**Unit tests** (`$0` = `unit` or `all`):
- Use Glob to find all `src/**/*.feature` files (exclude `*.a11y.feature`)
- Use Glob to find all `src/**/*.steps.tsx` files (exclude `*.a11y.steps.tsx`)
- For each `.feature`, check if a matching `.steps.tsx` exists in the same directory
- For each `.steps.tsx`, check if a matching `.feature` exists in the same directory
- Exception: `src/test/sharedSteps.ts` is allowed to exist without a matching `.feature`

**Accessibility tests** (`$0` = `a11y` or `all`):
- Use Glob to find all `src/**/*.a11y.feature` files
- Use Glob to find all `src/**/*.a11y.steps.tsx` files
- For each `.a11y.feature`, check if a matching `.a11y.steps.tsx` exists in the same directory
- For each `.a11y.steps.tsx`, check if a matching `.a11y.feature` exists in the same directory
- Exception: `src/test/sharedA11ySteps.ts` is allowed without a matching feature

**E2E tests** (`$0` = `e2e` or `all`):
- Use Glob to find all `e2e/features/*.feature` files
- Use Glob to find all `e2e/steps/*.steps.ts` files
- For each feature file, check if a matching steps file exists (by name: `Checkout.feature` → `Checkout.steps.ts`)
- For each steps file, check if a matching feature file exists
- Exception: `e2e/support/sharedSteps.ts` is allowed without a matching feature

### Step 2: Check for deleted components

For each unit and a11y test file found, verify the source component still exists:
- If `src/components/Button/Button.feature` exists, check that `src/components/Button/Button.tsx` (or `.ts`, `.jsx`) also exists
- If the component file is gone, the test files are orphaned

### Step 3: Report results

Present results in a clear table:

```
## Drift Report

### Unit tests
| File | Status | Issue |
|------|--------|-------|
| src/components/Button/Button.feature | OK | Matching .steps.tsx found |
| src/components/OldCard/OldCard.steps.tsx | ORPHANED | No matching .feature found |
| src/components/Deleted/Deleted.feature | ORPHANED | Component file not found |

### Accessibility tests
| File | Status | Issue |
|------|--------|-------|
| ... | ... | ... |

### E2E tests
| File | Status | Issue |
|------|--------|-------|
| ... | ... | ... |

### Summary
- Total files scanned: X
- Matched pairs: Y
- Orphaned files: Z
```

### Step 4: Suggest fixes

For each orphaned file, suggest one of:
1. **Delete** the orphaned file (if the component/flow no longer exists)
2. **Create** the missing counterpart (if the feature/steps should exist)
3. **Rename** the file (if it was a naming mismatch)

Do not delete files automatically — always present the suggestions and wait for user confirmation.
