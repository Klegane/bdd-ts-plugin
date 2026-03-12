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

## Accessibility Test Rules (a11y component tests)

### Rule 8: Accessibility File Naming
Accessibility tests use a distinct naming convention to co-exist with unit tests:
- `src/components/Button/Button.a11y.feature` -> `src/components/Button/Button.a11y.steps.tsx`
- This follows the same one-to-one mapping as Rule 4, but with the `.a11y` infix.
- Shared a11y steps go in `src/test/sharedA11ySteps.ts`.
- Do **not** mix accessibility scenarios into the component's primary `.feature` file. Accessibility is a separate concern with its own assertions.

### Rule 9: axe-core as Baseline
Every accessibility feature file MUST include a baseline scenario that runs the full axe-core audit:

```gherkin
Scenario: No automated accessibility violations are detected
  Given the component is rendered
  Then no automated accessibility violations are detected
```

This catches a broad class of WCAG violations automatically. Additional targeted scenarios (keyboard navigation, focus order, specific ARIA patterns) supplement this baseline but never replace it.

- Step definitions for axe-core assertions must use `vitest-axe` (`import { axe, toHaveNoViolations } from 'vitest-axe'`).
- Call `expect.extend(toHaveNoViolations)` at the top of the step definition file.
- Pass the rendered `container` (from `render().container`) to `axe()`, not a screen query result.

### Rule 10: Declarative Accessibility Language
Feature files describe accessibility requirements in user-facing language, not technical ARIA jargon:

**Bad:**
```gherkin
Then the element has aria-label="Close dialog"
Then the div has role="alert"
```

**Good:**
```gherkin
Then the close button has an accessible name
Then the error notification is announced to screen readers
```

Step definitions translate these human-readable assertions into technical checks (ARIA attribute verification, role queries, axe-core rule targeting). The feature file is the accessibility contract for the whole team; the step file is the technical implementation.

### Rule 11: Mandatory `@a11y` Tag and Selective Execution
Every accessibility feature file MUST include the `@a11y` tag at the Feature level, along with the WCAG conformance level tag:

```gherkin
@a11y @wcag-aa
Feature: Button accessibility
  ...
```

This tag serves two purposes:
1. **Identification**: makes it instantly clear in any test report or file listing that these are accessibility tests, separate from behavioral unit tests.
2. **Selective execution**: allows teams to control when accessibility tests run.

Common execution patterns:
- **Run only a11y tests**: `npm test -- --run **/*.a11y.steps.tsx`
- **Run everything except a11y tests**: configure vitest's `exclude` option or use glob negation in the test command.
- **CI pipeline separation**: run unit tests in a fast pipeline stage and a11y tests in a dedicated accessibility gate.

Never omit the `@a11y` tag from an accessibility feature file, even if the file is already named with the `.a11y` infix. The tag and the filename convention serve complementary purposes — the filename enables file-level filtering, and the tag enables report-level filtering and documentation.
