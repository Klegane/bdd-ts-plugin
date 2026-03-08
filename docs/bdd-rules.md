# BDD Rules

When generating, analyzing, or modifying any BDD `.feature` or step definition files, you MUST adhere to these rules.

## Shared Rules (all BDD tests)

### Rule 1: Reusability First
Before writing a new step definition, always verify if a similar step already exists in the relevant shared steps file or in the current step file. Re-use existing parameter types like `{string}` and `{int}` instead of writing new granular steps.

**Bad:**
```tsx
Given('a primary button', () => { ... })
Given('a secondary button', () => { ... })
```

**Good:**
```tsx
Given('a {string} button', (_ctx, variant: string) => { ... })
```

### Rule 2: Strict UI Interaction Separation
- **Step definitions** should handle all technical interactions (selectors, locators, API calls).
- **Feature files (`.feature`)** should only contain human-readable, domain-specific text.
- You must **never** place CSS selectors, XPath strings, URLs, or test implementation details inside Feature files.

### Rule 3: Step Matching
- Only use string templates (e.g., `Given('I click {string}', (_ctx, text: string) => ...)`) for step matching, never Raw Regex matchers.
- You must accept `_ctx` as the first argument in all parameterized steps.

## Unit Test Rules (component-level tests)

### Rule 4: One-to-One File Mapping
Every `.feature` file must have exactly one corresponding `.steps.tsx` file in the same directory.
- `src/components/Button/Button.feature` -> `src/components/Button/Button.steps.tsx`
- Do **not** create global, orphaned, or unmapped step files unless they are explicitly placed in `src/test/sharedSteps.ts`.

### Rule 5: Vitest-Cucumber Architecture
This project uses `@amiceli/vitest-cucumber` which converts every Step into an isolated Vitest test block.
- **NEVER** use `jest-cucumber` or standard `it()`/`test()` blocks inside `.steps.tsx` files.
- You must use the `describeFeature` and `Scenario` wrapper syntax.
- Use `BeforeEachScenario` or `AfterEachScenario` from the `describeFeature` parameters. Do **not** use the global `beforeEach()` as it resets DOM state between individual step executions.

## E2E Test Rules (Playwright end-to-end tests)

### Rule 6: E2E File Structure
- Feature files go in `e2e/features/`.
- Step definitions go in `e2e/steps/`.
- Shared E2E helpers go in `e2e/support/sharedSteps.ts`.

### Rule 7: Playwright Locators
- Use Playwright's accessible locators (`getByRole`, `getByText`, `getByLabel`) -- avoid raw CSS selectors and `getByTestId` where possible.
- Keep page URLs and navigation logic inside step definitions, never in feature files.
