# Feature Drift Detection Reference

## What is feature drift?

Feature drift occurs when BDD test files become out of sync with the codebase:
- `.feature` without matching `.steps.tsx` (undefined scenarios)
- `.steps.tsx` without matching `.feature` (orphaned step definitions)
- Test files for components that no longer exist

## File pairing conventions

| Test type | Feature file | Steps file | Shared steps |
|-----------|-------------|------------|-------------|
| Unit | `src/components/<Name>/<Name>.feature` | `src/components/<Name>/<Name>.steps.tsx` | `src/test/sharedSteps.ts` |
| A11y | `src/components/<Name>/<Name>.a11y.feature` | `src/components/<Name>/<Name>.a11y.steps.tsx` | `src/test/sharedA11ySteps.ts` |
| E2E | `e2e/features/<Flow>.feature` | `e2e/steps/<Flow>.steps.ts` | `e2e/support/sharedSteps.ts` |

## How to run

```bash
# Check all test types
/bdd-ts-plugin:bdd-drift all

# Check specific test type
/bdd-ts-plugin:bdd-drift unit
/bdd-ts-plugin:bdd-drift a11y
/bdd-ts-plugin:bdd-drift e2e
```

## Automated drift guard

For CI integration, create `src/test/featureDrift.test.ts` that glob-scans for unpaired files and fails if any are found. This ensures drift is caught on every push, not just when someone remembers to run the skill.
