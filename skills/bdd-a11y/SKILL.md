---
name: bdd-a11y
description: Scaffold and implement accessibility BDD tests for React components using vitest-cucumber, React Testing Library, and axe-core (via vitest-axe). Use when creating or modifying .a11y.feature and .a11y.steps.tsx files for WCAG compliance testing. Trigger this skill whenever the user mentions accessibility tests, a11y, WCAG, axe-core, screen reader compatibility, keyboard navigation, ARIA, color contrast, focus management, or asks to "test accessibility for a component".
argument-hint: <ComponentName>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npm test *)
---

# BDD Accessibility Test Scaffold

Scaffold an accessibility (a11y) BDD test for the `$0` component.

## Why BDD for accessibility?

Accessibility requirements are behaviors: "a user can navigate the form with keyboard alone" is a testable scenario, not a technical implementation detail. By expressing WCAG requirements as Gherkin scenarios, accessibility contracts become readable by the entire team — not just developers who know axe-core APIs. The `.a11y.feature` file is the accessibility contract; the `.a11y.steps.tsx` file wires it to automated audits via axe-core and manual RTL assertions.

## BDD Rules (always enforced)

These rules are non-negotiable. They exist to maintain consistency across the entire test suite — skipping them leads to drift that's painful to fix later.

### Rule 1: Reusability First
Before writing a new step definition, verify if a similar step already exists. Re-use parameter types like `{string}` and `{int}` instead of writing granular, variant-specific steps.

**Bad:** `Given('a primary button', ...)` and `Given('a secondary button', ...)`
**Good:** `Given('a {string} button', (_ctx, variant: string) => ...)`

### Rule 2: Strict UI Interaction Separation
- **Step definitions** handle all technical interactions (selectors, locators, axe-core calls).
- **Feature files** contain only human-readable, domain-specific text.
- **Never** place CSS selectors, ARIA attribute names, axe-core rule IDs, or implementation details inside `.feature` files.

### Rule 3: Step Matching
- Only use string templates (`{string}`, `{int}`) for step matching — never raw regex matchers.
- Accept `_ctx` as the first argument in all parameterized steps.

### Rule 5: Vitest-Cucumber Architecture
This project uses `@amiceli/vitest-cucumber` which converts every Step into an isolated Vitest test block.
- **NEVER** use `jest-cucumber` or standard `it()`/`test()` blocks inside `.steps.tsx` files.
- Use the `describeFeature` and `Scenario` wrapper syntax.
- Use `AfterEachScenario` from the `describeFeature` parameters. Do **not** use `beforeEach()` — it resets DOM state between individual step executions.

### Rule 8: Accessibility File Naming
Accessibility tests use a distinct naming convention to co-exist with unit tests:
- `src/components/Button/Button.a11y.feature` -> `src/components/Button/Button.a11y.steps.tsx`
- Shared a11y steps go in `src/test/sharedA11ySteps.ts`.
- Do **not** mix accessibility scenarios into the component's primary `.feature` file.

### Rule 9: axe-core as Baseline
Every accessibility feature file MUST include a baseline scenario that runs the full axe-core audit (`Then no automated accessibility violations are detected`). This catches a broad class of WCAG violations automatically.
- Use `vitest-axe` (`import { axe, toHaveNoViolations } from 'vitest-axe'`).
- Call `expect.extend(toHaveNoViolations)` at the top of the step definition file.
- Pass `render().container` to `axe()`, not a screen query result.

### Rule 10: Declarative Accessibility Language
Feature files describe accessibility outcomes in user-facing language, not technical ARIA jargon.

**Bad:** `Then the element has aria-label="Close dialog"`
**Good:** `Then the close button has an accessible name`

### Rule 11: Mandatory `@a11y` Tag and Selective Execution
Every accessibility feature file MUST include the `@a11y` tag at the Feature level, along with the WCAG conformance level tag (`@wcag-aa` or `@wcag-aaa`). This enables:
- **Selective execution**: `npm test -- --run **/*.a11y.steps.tsx`
- **Report-level filtering**: the tag makes a11y tests identifiable in any test report.
- **CI separation**: run unit tests in a fast stage and a11y tests in a dedicated gate.
Never omit the `@a11y` tag, even though the `.a11y` filename infix also exists — they serve complementary purposes.

## Existing shared accessibility steps

The following shared a11y helpers are available for reuse. Duplicating what already exists here wastes effort and creates maintenance burden:

!`cat src/test/sharedA11ySteps.ts 2>/dev/null || echo "No shared a11y steps file found."`

## Existing accessibility feature files

These a11y feature files already exist in the project — check them for reusable step patterns before inventing new ones:

!`find src -name "*.a11y.feature" 2>/dev/null || echo "No a11y feature files found."`

## WCAG 2.1 Quick Reference

When designing scenarios, categorize them by WCAG principle. Not every component needs all categories — pick what is relevant to the component's accessibility surface:

| Principle | Key Criteria (AA) | Example Scenario Themes |
|-----------|-------------------|------------------------|
| **Perceivable** | 1.1.1 Non-text Content, 1.4.3 Contrast, 1.4.5 Images of Text | Alt text present, sufficient color contrast, text alternatives for icons |
| **Operable** | 2.1.1 Keyboard, 2.4.3 Focus Order, 2.4.7 Focus Visible | Full keyboard navigation, logical tab order, visible focus indicators |
| **Understandable** | 3.3.1 Error Identification, 3.3.2 Labels or Instructions | Form labels associated, error messages descriptive and linked to fields |
| **Robust** | 4.1.2 Name/Role/Value | Valid ARIA attributes, semantic HTML elements, roles match behavior |

Default WCAG level: **AA**. If the user requests AAA, note it in the Feature header as a tag: `@wcag-aaa`.

## Workflow

Follow these steps in order. Each step builds on the previous one, and skipping ahead (especially writing steps before confirming scenarios) leads to rework.

### Step 1: Gather requirements

Understand the component and its accessibility surface before writing anything.

- Find the component source at `src/components/$0/$0.tsx` (or a similar path).
- Read it to identify: semantic HTML elements used, ARIA attributes present, interactive elements (buttons, inputs, links), images/icons, dynamic content (modals, dropdowns, tooltips), and form structure.
- If the component doesn't exist, stop and ask the user — don't guess.
- Ask the user which WCAG categories matter most for this component, or infer from the source:
  - **Perceivable**: Does it have images, icons, or color-dependent information?
  - **Operable**: Is it interactive? Does it have keyboard-triggered actions?
  - **Understandable**: Does it have forms, error states, or labels?
  - **Robust**: Does it use ARIA roles, live regions, or custom widgets?
- Determine the WCAG conformance level: AA (default) or AAA (if user requests).

### Step 2: Extract scenarios

Map each accessibility concern to a single Scenario. Group by WCAG principle for clarity. The goal is one accessibility behavior per scenario — this makes failures easy to diagnose because you immediately know which requirement broke.

Present the mapping for confirmation before writing anything:

| WCAG Principle | Criterion | Scenario |
|----------------|-----------|----------|
| Baseline | Full audit | No automated accessibility violations are detected |
| Perceivable | 1.1.1 Non-text Content | All images have descriptive alt text |
| Operable | 2.1.1 Keyboard | Component is fully operable via keyboard |
| Understandable | 3.3.2 Labels | All form inputs have associated labels |

Always include a "baseline" scenario: "No automated accessibility violations are detected" — this runs the full axe-core audit and catches a wide range of issues automatically.

Wait for the user's OK. They often have context about accessibility requirements or known issues that aren't visible in the source code.

### Step 3: Check for existing accessibility tests

- Check if `src/components/$0/$0.a11y.feature` and `src/components/$0/$0.a11y.steps.tsx` already exist.
- If they do, read them and work incrementally — add new scenarios without overwriting existing ones. Existing passing tests are valuable; breaking them to reorganize is never worth it.
- Also check if the component already has unit tests (`$0.feature` / `$0.steps.tsx`) to understand what is already covered and avoid duplicating behavioral assertions that belong in the unit test.

### Step 4: Check shared accessibility steps

- Review the shared a11y steps injected above in the "Existing shared accessibility steps" section.
- Also check existing a11y feature files listed above for step patterns that can be reused.
- Common reusable a11y steps include:
  - `Then no automated accessibility violations are detected` (axe-core full audit)
  - `Then no accessibility violations for {string} are detected` (axe-core with specific rule tags)
  - `Then the component is keyboard navigable`
  - `Then all images have alt text`
  - `Then all form inputs have associated labels`
- Reuse these before creating new ones. Consistent step language across features makes the whole test suite easier to read and maintain.

### Step 5: Design the feature file

Create or update `src/components/$0/$0.a11y.feature`.

The feature file is for humans first, machines second. Write it so a product manager could read it and understand the accessibility requirements without knowing what axe-core is.

- **Mandatory tagging (Rule 11)**: Tag the Feature with `@a11y` and the WCAG level (e.g., `@wcag-aa` or `@wcag-aaa`). The `@a11y` tag is required on every accessibility feature — it enables selective execution (`npm test -- --run **/*.a11y.steps.tsx`) and makes accessibility tests instantly identifiable in test reports. Never omit it.
- Write human-readable Gherkin scenarios covering the component's accessibility requirements.
- Use parameterized steps with `"quoted strings"` for variable data — never hardcode variants into step names.
- Never include ARIA attribute names, axe-core rule IDs, CSS selectors, or implementation details. If you find yourself writing `Then the element has aria-label="Close"`, you're leaking implementation into the contract.

Example structure:

```gherkin
@a11y @wcag-aa
Feature: Button accessibility

  Scenario: No automated accessibility violations are detected
    Given the "primary" button was rendered
    Then no automated accessibility violations are detected

  Scenario: Button is keyboard operable
    Given the "primary" button was rendered
    When the user activates the button via keyboard
    Then the button action is triggered

  Scenario: Button has sufficient color contrast
    Given the "primary" button was rendered
    Then no accessibility violations for "color-contrast" are detected

  Scenario: Button has an accessible name
    Given the "primary" button was rendered
    Then the button has an accessible name
```

### Step 6: Run the tests (expect failures)

Run `npm test -- --run src/components/$0` to confirm the new a11y scenarios fail with unmapped step errors. This validates that the feature file is syntactically correct and that vitest-cucumber recognizes the scenarios. If you get parsing errors instead of unmapped step errors, fix the feature file first.

### Step 7: Implement the step definitions

Create or update `src/components/$0/$0.a11y.steps.tsx` following this structure:

```tsx
import { render, screen, cleanup } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { loadFeature, describeFeature } from '@amiceli/vitest-cucumber'
import { axe, toHaveNoViolations } from 'vitest-axe'
import { expect } from 'vitest'
import { ComponentName } from './ComponentName'

expect.extend(toHaveNoViolations)

const feature = await loadFeature('./src/components/$0/$0.a11y.feature')

describeFeature(feature, ({ Scenario, AfterEachScenario }) => {
  let container: HTMLElement

  AfterEachScenario(() => {
    cleanup()
  })

  Scenario('No automated accessibility violations are detected', ({ Given, Then }) => {
    Given('the {string} button was rendered', (_ctx, variant: string) => {
      const result = render(<ComponentName variant={variant} />)
      container = result.container
    })

    Then('no automated accessibility violations are detected', async () => {
      const results = await axe(container)
      expect(results).toHaveNoViolations()
    })
  })

  Scenario('Button is keyboard operable', ({ Given, When, Then }) => {
    Given('the {string} button was rendered', (_ctx, variant: string) => {
      render(<ComponentName variant={variant} />)
    })

    When('the user activates the button via keyboard', async () => {
      await userEvent.tab()
      await userEvent.keyboard('{Enter}')
    })

    Then('the button action is triggered', () => {
      // Assert the expected outcome of keyboard activation
    })
  })

  Scenario('Button has sufficient color contrast', ({ Given, Then }) => {
    Given('the {string} button was rendered', (_ctx, variant: string) => {
      const result = render(<ComponentName variant={variant} />)
      container = result.container
    })

    Then('no accessibility violations for {string} are detected', async (_ctx, ruleTags: string) => {
      const results = await axe(container, {
        runOnly: {
          type: 'tag',
          values: [ruleTags],
        },
      })
      expect(results).toHaveNoViolations()
    })
  })

  Scenario('Button has an accessible name', ({ Given, Then }) => {
    Given('the {string} button was rendered', (_ctx, variant: string) => {
      render(<ComponentName variant={variant} />)
    })

    Then('the button has an accessible name', () => {
      const button = screen.getByRole('button')
      expect(button).toHaveAccessibleName()
    })
  })
})
```

Key conventions and the reasoning behind them:

- **`vitest-axe` imports** — import `axe` and `toHaveNoViolations` from `vitest-axe`. Call `expect.extend(toHaveNoViolations)` at the top level so the matcher is available in all scenarios.
- **`container` capture** — the `render()` return value's `.container` must be stored for `axe()` calls. The `axe()` function needs a DOM node, not a screen query result.
- **Parameterized axe rules** — use `runOnly: { type: 'tag', values: [...] }` to run axe against specific WCAG tags (e.g., `"color-contrast"`, `"wcag2aa"`, `"wcag21aa"`).
- **Keyboard testing** — use `userEvent.tab()` to move focus and `userEvent.keyboard('{Enter}')` or `userEvent.keyboard(' ')` to activate. Assert focus with `expect(element).toHaveFocus()`.
- **`_ctx` as the first argument** in all parameterized steps — vitest-cucumber passes context as the first arg; ignoring it with `_ctx` keeps TypeScript happy and makes the actual parameters visually obvious.
- **String templates (`{string}`, `{int}`) for step matching** — never regex.
- **`AfterEachScenario` for cleanup** — never `beforeEach()`.

### Step 8: Run tests again (expect green)

Run `npm test -- --run src/components/$0` to confirm all a11y scenarios pass. If tests fail, read the error output carefully — the most common issues are:

- `vitest-axe` not installed — inform the user to run `npm install -D vitest-axe`
- `axe()` called on `null` container — ensure `render()` result is captured before the `Then` step
- Step text mismatch between `.a11y.feature` and `.a11y.steps.tsx` (check quotes and spacing)
- Legitimate accessibility violations found by axe-core — these are real issues that should be fixed in the component, not in the test

Fix the step definitions and re-run until green.

### Step 9: Verify feature drift guard

Run `npm test -- --run src/test/featureDrift` to ensure no orphaned feature or step files exist. This catches situations where a component was deleted or renamed but its test files were left behind.

Note: The feature drift check may need to be updated to also scan for `.a11y.feature` / `.a11y.steps.tsx` pairs. If the drift test does not cover these yet, flag it to the user.

## Selective execution

After completing the workflow, inform the user of the available execution commands:

- **Run only a11y tests**: `npm test -- --run **/*.a11y.steps.tsx`
- **Run a11y tests for a specific component**: `npm test -- --run src/components/$0/$0.a11y.steps.tsx`
- **Run all tests (unit + a11y)**: `npm test -- --run src/components/$0`
- **CI pipeline separation**: a11y tests can be isolated into a dedicated pipeline stage using the `**/*.a11y.steps.tsx` glob pattern.

The `@a11y` tag in the feature file and the `.a11y` infix in filenames work together — the filename enables file-level filtering in the test runner, and the tag enables report-level filtering and makes the test purpose self-documenting.
