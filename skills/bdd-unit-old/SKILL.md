---
name: bdd-unit
description: Scaffold and implement a component-level BDD test using vitest-cucumber and React Testing Library. Use when creating or modifying .feature and .steps.tsx files for component testing.
argument-hint: <ComponentName>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npm test *)
---

# BDD Unit Test Scaffold

Scaffold a component-level BDD test for the `$0` component.

## BDD Rules (always enforced)

Read and strictly follow every rule in [bdd-rules.md](${CLAUDE_SKILL_DIR}/../../docs/bdd-rules.md) before proceeding.

## Existing shared steps

The following shared helpers and mock data are available for reuse:

!`cat src/test/sharedSteps.ts 2>/dev/null || echo "No shared steps file found."`

## Existing feature files

These feature files already exist in the project -- check them for reusable step patterns:

!`find src -name "*.feature" 2>/dev/null || echo "No feature files found."`

## Workflow (Order of Operations)

Follow these steps strictly in order:

### Step 1: Gather requirements
- Ask the user what behaviors the component should have, or read the component source to infer them.
- Find the component source file at `src/components/$0/$0.tsx` (or similar path).
- Read it to understand its props, behavior, and variants.
- If the component does not exist, stop and ask the user for guidance.

### Step 2: Extract scenarios
- Map each distinct behavior or requirement to a single Scenario (one behavior = one scenario).
- Present the mapping to the user for confirmation before writing:
  | Requirement | Scenario |
  |-------------|----------|
  | behavior 1  | Scenario name 1 |
  | behavior 2  | Scenario name 2 |

### Step 3: Check for existing tests
- Check if `src/components/$0/$0.feature` and `src/components/$0/$0.steps.tsx` already exist.
- If they exist, read them and work incrementally -- add new scenarios, do not overwrite.

### Step 4: Check shared steps
- Review the shared steps injected above in the "Existing shared steps" section.
- Also check existing feature files listed above for step patterns that can be reused.
- Reuse existing shared steps before creating new ones.

### Step 5: Design the feature file
- Create or update `src/components/$0/$0.feature`.
- Write human-readable Gherkin scenarios covering the component's key behaviors.
- Use parameterized steps with `"quoted strings"` -- never hardcode variants into step names.
- Never include CSS selectors, test IDs, or implementation details in the feature file.

### Step 6: Run the tests (expect failures)
- Run `npm test -- --run src/components/$0` to confirm the new scenarios fail with unmapped step errors.

### Step 7: Implement the step definitions
- Create or update `src/components/$0/$0.steps.tsx`.
- Follow this exact structure:

```tsx
import { render, screen, cleanup } from '@testing-library/react/pure'
import userEvent from '@testing-library/user-event'
import { loadFeature, describeFeature } from '@amiceli/vitest-cucumber'
import { ComponentName } from './ComponentName'

const feature = await loadFeature('./src/components/$0/$0.feature')

describeFeature(feature, ({ Scenario, AfterEachScenario, BeforeEachScenario }) => {
  AfterEachScenario(() => {
    cleanup()
  })

  Scenario('Scenario name matching feature file', ({ Given, When, Then, And }) => {
    Given('step with {string}', (_ctx, param: string) => {
      // RTL implementation
    })
  })
})
```

- Use `_ctx` as the first argument in all parameterized steps.
- Use string templates (`{string}`, `{int}`) for step matching -- never regex.
- Use `AfterEachScenario` for cleanup -- never `beforeEach()`.

### Step 8: Run tests again (expect green)
- Run `npm test -- --run src/components/$0` to confirm all scenarios pass.
- If tests fail, fix the step definitions and re-run until green.

### Step 9: Verify feature drift guard
- Run `npm test -- --run src/test/featureDrift` to ensure no orphaned feature or step files exist.
