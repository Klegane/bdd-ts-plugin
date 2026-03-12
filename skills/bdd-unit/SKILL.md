---
name: bdd-unit
description: Scaffold and implement a component-level BDD test using vitest-cucumber and React Testing Library. Use when creating or modifying .feature and .steps.tsx files for component testing. Trigger this skill whenever the user mentions BDD tests, Gherkin scenarios, feature files, step definitions, vitest-cucumber, or asks to "add tests for a component" even if they don't explicitly say "BDD". Also trigger when the user references .feature or .steps.tsx files, asks to test component behaviors/interactions, or wants to scaffold test coverage for a React component.
argument-hint: <ComponentName>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npm test *)
---

# BDD Unit Test Scaffold

Scaffold a component-level BDD test for the `$0` component.

## Why BDD at the component level?

Each component gets a `.feature` file that describes *what* it does in plain language and a `.steps.tsx` file that maps those descriptions to real interactions via React Testing Library. This keeps tests readable by non-developers while ensuring they exercise real DOM behavior. The feature file is the contract; the steps file is the implementation.

## BDD Rules (always enforced)

These rules are non-negotiable. They exist to maintain consistency across the entire test suite — skipping them leads to drift that's painful to fix later.

### Rule 1: Reusability First
Before writing a new step definition, verify if a similar step already exists. Re-use parameter types like `{string}` and `{int}` instead of writing granular, variant-specific steps.

**Bad:** `Given('a primary button', ...)` and `Given('a secondary button', ...)`
**Good:** `Given('a {string} button', (_ctx, variant: string) => ...)`

### Rule 2: Strict UI Interaction Separation
- **Step definitions** handle all technical interactions (selectors, locators, API calls).
- **Feature files** contain only human-readable, domain-specific text.
- **Never** place CSS selectors, XPath strings, URLs, or implementation details inside `.feature` files.

### Rule 3: Step Matching
- Only use string templates (`{string}`, `{int}`) for step matching — never raw regex matchers.
- Accept `_ctx` as the first argument in all parameterized steps.

### Rule 4: One-to-One File Mapping
Every `.feature` file must have exactly one corresponding `.steps.tsx` file in the same directory.
- `src/components/Button/Button.feature` -> `src/components/Button/Button.steps.tsx`
- Do **not** create orphaned or unmapped step files unless they are in `src/test/sharedSteps.ts`.

### Rule 5: Vitest-Cucumber Architecture
This project uses `@amiceli/vitest-cucumber` which converts every Step into an isolated Vitest test block.
- **NEVER** use `jest-cucumber` or standard `it()`/`test()` blocks inside `.steps.tsx` files.
- Use the `describeFeature` and `Scenario` wrapper syntax.
- Use `AfterEachScenario` from the `describeFeature` parameters. Do **not** use `beforeEach()` — it resets DOM state between individual step executions.

## Existing shared steps

The following shared helpers and mock data are available for reuse. Duplicating what already exists here wastes effort and creates maintenance burden:

!`cat src/test/sharedSteps.ts 2>/dev/null || echo "No shared steps file found."`

## Existing feature files

These feature files already exist in the project — check them for reusable step patterns before inventing new ones:

!`find src -name "*.feature" 2>/dev/null || echo "No feature files found."`

## Workflow

Follow these steps in order. Each step builds on the previous one, and skipping ahead (especially writing steps before confirming scenarios) leads to rework.

### Step 1: Gather requirements

Understand the component before writing anything.

- Find the component source at `src/components/$0/$0.tsx` (or a similar path).
- Read it to understand its props, behavior, conditional rendering, and user interactions.
- If the component doesn't exist, stop and ask the user — don't guess.
- Ask the user what behaviors matter most, or infer them from the source. Both approaches are valid, but confirm your understanding either way.

### Step 2: Extract scenarios

Map each distinct behavior to a single Scenario. The goal is one behavior per scenario — this makes failures easy to diagnose because you immediately know which behavior broke.

Present the mapping for confirmation before writing anything:

| Requirement | Scenario |
|-------------|----------|
| behavior 1  | Scenario name 1 |
| behavior 2  | Scenario name 2 |

Wait for the user's OK. They often have context about edge cases or priorities that isn't visible in the source code.

### Step 3: Check for existing tests

- Check if `src/components/$0/$0.feature` and `src/components/$0/$0.steps.tsx` already exist.
- If they do, read them and work incrementally — add new scenarios without overwriting existing ones. Existing passing tests are valuable; breaking them to reorganize is never worth it.

### Step 4: Check shared steps

- Review the shared steps injected above in the "Existing shared steps" section.
- Also check existing feature files listed above for step patterns that can be reused.
- Reuse existing shared steps before creating new ones. Consistent step language across features makes the whole test suite easier to read and maintain.

### Step 5: Design the feature file

Create or update `src/components/$0/$0.feature`.

The feature file is for humans first, machines second. Write it so a product manager could read it and understand what the component does.

- Write human-readable Gherkin scenarios covering the component's key behaviors.
- Use parameterized steps with `"quoted strings"` for variable data — never hardcode variants into step names because it makes steps impossible to reuse.
- Never include CSS selectors, test IDs, or implementation details. If you find yourself writing `When I click the element with data-testid="submit-btn"`, you're leaking implementation into the contract.

### Step 6: Run the tests (expect failures)

Run `npm test -- --run src/components/$0` to confirm the new scenarios fail with unmapped step errors. This validates that the feature file is syntactically correct and that vitest-cucumber recognizes the scenarios. If you get parsing errors instead of unmapped step errors, fix the feature file first.

### Step 7: Implement the step definitions

Create or update `src/components/$0/$0.steps.tsx` following this structure:

```tsx
import { render, screen, cleanup } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { loadFeature, describeFeature } from '@amiceli/vitest-cucumber'
import { ComponentName } from './ComponentName'

const feature = await loadFeature('./src/components/$0/$0.feature')

describeFeature(feature, ({ Scenario, AfterEachScenario }) => {
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

Key conventions and the reasoning behind them:

- **`_ctx` as the first argument** in all parameterized steps — vitest-cucumber passes context as the first arg; ignoring it with `_ctx` keeps TypeScript happy and makes the actual parameters visually obvious.
- **String templates (`{string}`, `{int}`) for step matching** — never regex. Regex is harder to read and doesn't compose well with vitest-cucumber's built-in parameter handling.
- **`AfterEachScenario` for cleanup** — never `beforeEach()`. vitest-cucumber manages its own lifecycle; mixing in Vitest's lifecycle hooks causes subtle ordering bugs.

### Step 8: Run tests again (expect green)

Run `npm test -- --run src/components/$0` to confirm all scenarios pass. If tests fail, read the error output carefully — the most common issues are:

- Step text mismatch between `.feature` and `.steps.tsx` (check quotes and spacing)
- Missing `_ctx` first argument in parameterized steps
- Component needing providers or context that aren't wrapped in the render call

Fix the step definitions and re-run until green.

### Step 9: Verify feature drift guard

Run `npm test -- --run src/test/featureDrift` to ensure no orphaned feature or step files exist. This catches situations where a component was deleted or renamed but its test files were left behind.
