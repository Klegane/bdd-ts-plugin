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

### Rule 12: Assertion Quality
Assertions must verify something meaningful. A weak assertion that never fails is worse than no assertion — it gives false confidence.

- **NEVER** use `.toBeTruthy()` or `.toBeDefined()` to check element existence. RTL's `getBy*` queries already throw if the element is missing, so `.toBeTruthy()` is a no-op that tests nothing.
- **Always import** `@testing-library/jest-dom/vitest` at the top of step definition files to enable semantic matchers.
- **Use semantic matchers** from `@testing-library/jest-dom`:
  - `toBeInTheDocument()` — element exists in the DOM
  - `toHaveAttribute('href', '/path')` — element has a specific attribute value
  - `toHaveTextContent('text')` — element contains expected text
  - `toBeVisible()` — element is visible to the user
  - `toHaveAccessibleName()` — element has an accessible name

**Bad:** `expect(screen.getByText('Submit')).toBeTruthy()` — the `getByText` already throws if missing; `.toBeTruthy()` never fails and verifies nothing.
**Good:** `expect(screen.getByRole('button', { name: 'Submit' })).toBeInTheDocument()`
**Good:** `expect(screen.getByRole('link')).toHaveAttribute('href', 'https://example.com')`
**Good:** `expect(screen.getByRole('link')).toHaveAttribute('target', '_blank')`

### Rule 13: Data-Driven Component Guards
When a component renders items from a data array (e.g., a list of cards, navigation items, table rows):

- **Always assert the expected count** — verify the correct number of items rendered, not just that "some" exist.
- **Guard against empty arrays** — if the test iterates over a data array to make assertions, first assert the array is non-empty. An empty array means zero assertions, which is a silently passing test that verifies nothing.

**Bad:**
```tsx
games.forEach((game) => {
  expect(screen.getByRole('link', { name: new RegExp(game.title) })).toBeTruthy()
})
```
**Good:**
```tsx
expect(games.length).toBeGreaterThan(0)
const links = screen.getAllByRole('link')
expect(links).toHaveLength(games.length)
games.forEach((game) => {
  expect(screen.getByRole('link', { name: new RegExp(game.title) })).toBeInTheDocument()
})
```

### Rule 14: Interaction Coverage
When a component has interactive elements (links, buttons, inputs, toggleable widgets), the test suite MUST include at least one scenario that exercises user interaction, not just static rendering.

- Use `userEvent` from `@testing-library/user-event` for all interactions — never `fireEvent`.
- If a component wraps interactive elements (e.g., a card that is an `<a>` tag), test the interaction contract: that the link navigates, that `target` and `rel` attributes are correct, that click handlers fire.
- If a component has hover/focus states, test that the element receives focus via keyboard (`userEvent.tab()`).

## Existing shared steps

Before writing new step definitions, check these files for reusable logic:

1. Read `src/test/sharedSteps.ts` (if it exists) — it contains shared step helpers used across components.
2. Use Glob to find all `src/**/*.feature` files and scan them for step patterns you can reuse.
3. Use Glob to find all `src/**/*.steps.tsx` files and check for shared imports or helpers.

Duplicating what already exists wastes effort and creates maintenance burden. If no shared steps file exists yet, note that — you may want to propose creating one after this task.

## Workflow

Follow these steps in order. Each step builds on the previous one, and skipping ahead (especially writing steps before confirming scenarios) leads to rework.

### Step 0: Validate prerequisites

Before starting, verify these packages exist in `package.json` `devDependencies`:
- `@amiceli/vitest-cucumber`
- `@testing-library/react`
- `@testing-library/jest-dom`
- `@testing-library/user-event`
- `vitest`

If any are missing, inform the user and offer to install them before proceeding:
```bash
npm install -D @amiceli/vitest-cucumber @testing-library/react @testing-library/jest-dom @testing-library/user-event vitest
```

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

- Follow the instructions in the "Existing shared steps" section above to find reusable step logic.
- Read `src/test/sharedSteps.ts` and scan existing `.feature` files for step patterns that can be reused.
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
import { render, screen, cleanup } from '@testing-library/react/pure'
import userEvent from '@testing-library/user-event'
import { loadFeature, describeFeature } from '@amiceli/vitest-cucumber'
import '@testing-library/jest-dom/vitest'
import { ComponentName } from './ComponentName'

const feature = await loadFeature('./src/components/$0/$0.feature')

describeFeature(feature, ({ Scenario, AfterEachScenario }) => {
  AfterEachScenario(() => {
    cleanup()
  })

  // Static rendering assertion — use toBeInTheDocument(), never toBeTruthy()
  Scenario('The title is visible', ({ Given, Then }) => {
    Given('a component with title {string}', (_ctx, title: string) => {
      render(<ComponentName title={title} />)
    })

    Then('the text {string} is visible', (_ctx, text: string) => {
      expect(screen.getByText(text)).toBeInTheDocument()
    })
  })

  // Attribute assertion — use toHaveAttribute() for href, target, rel, etc.
  Scenario('The card links to the correct URL', ({ Given, Then }) => {
    Given('a card linking to {string}', (_ctx, href: string) => {
      render(<ComponentName href={href} />)
    })

    Then('the card links to {string}', (_ctx, expectedHref: string) => {
      expect(screen.getByRole('link')).toHaveAttribute('href', expectedHref)
    })
  })

  // Interaction assertion — use userEvent, never fireEvent
  Scenario('The button triggers its handler', ({ Given, When, Then }) => {
    let clicked = false

    Given('a button is rendered', () => {
      render(<ComponentName onClick={() => { clicked = true }} />)
    })

    When('the user clicks the button', async () => {
      await userEvent.click(screen.getByRole('button'))
    })

    Then('the click handler is called', () => {
      expect(clicked).toBe(true)
    })
  })
})
```

Key conventions and the reasoning behind them:

- **`@testing-library/jest-dom/vitest` import** — enables semantic matchers (`toBeInTheDocument`, `toHaveAttribute`, `toBeVisible`, `toHaveTextContent`, `toHaveAccessibleName`). Without this import, these matchers are not available.
- **`toBeInTheDocument()` over `toBeTruthy()`** — `getBy*` queries throw when an element is missing, making `toBeTruthy()` a no-op. `toBeInTheDocument()` is semantically correct and reads clearly in test output.
- **`toHaveAttribute()` for attribute checks** — more readable and produces better error messages than `getAttribute() === value`.
- **`userEvent` over `fireEvent`** — `userEvent` simulates real browser behavior (focus, blur, keydown, keyup) while `fireEvent` dispatches synthetic events that skip important side effects.
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

### Step 10: Post-completion

After all tests pass, remind the user about these next steps:

1. **Accessibility tests**: "Would you like me to add accessibility (WCAG) tests for this component? Use `/bdd-ts-plugin:bdd-a11y $0` to scaffold a11y tests that verify keyboard navigation, color contrast, and screen reader support."
2. **Allure report**: "Run `/bdd-ts-plugin:allure-report` to generate an HTML test report from these results."
3. **Drift guard**: If `src/test/featureDrift` does not exist yet, suggest the user create it or invoke `/bdd-ts-plugin:bdd-drift`.
