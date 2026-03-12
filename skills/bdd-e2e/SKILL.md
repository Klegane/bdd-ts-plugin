---
name: bdd-e2e
description: Scaffold and implement an E2E BDD test using Playwright with Gherkin feature files. Use when creating end-to-end browser tests for user flows.
argument-hint: <FlowName>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npx playwright *)
---

# BDD E2E Test Scaffold

Scaffold an end-to-end BDD test for the `$0` user flow using Playwright.

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

### Rule 6: E2E File Structure
- Feature files go in `e2e/features/`.
- Step definitions go in `e2e/steps/`.
- Shared E2E helpers go in `e2e/support/sharedSteps.ts`.

### Rule 7: Playwright Locators
- Use Playwright's accessible locators (`getByRole`, `getByText`, `getByLabel`) — avoid raw CSS selectors and `getByTestId` where possible.
- Keep page URLs and navigation logic inside step definitions, never in feature files.

## Existing E2E shared steps

!`cat e2e/support/sharedSteps.ts 2>/dev/null || echo "No E2E shared steps file found yet."`

## Existing E2E feature files

!`find e2e -name "*.feature" 2>/dev/null || echo "No E2E feature files found yet."`

## Workflow (Order of Operations)

Follow these steps strictly in order:

### Step 1: Gather requirements
- Ask the user to describe the user flow if `$0` is ambiguous.
- Identify the pages, interactions, and expected outcomes involved.

### Step 2: Extract scenarios
- Map each distinct user interaction or outcome to a single Scenario (one behavior = one scenario).
- Present the mapping to the user for confirmation before writing:
  | Requirement | Scenario |
  |-------------|----------|
  | user action 1 | Scenario name 1 |
  | user action 2 | Scenario name 2 |

### Step 3: Check for existing E2E tests
- Look in `e2e/` (or `tests/e2e/`) for existing feature files and step definitions.
- If the directory structure doesn't exist yet, create it:
  ```
  e2e/
  ├── features/
  │   └── $0.feature
  ├── steps/
  │   └── $0.steps.ts
  └── support/
      └── sharedSteps.ts
  ```

### Step 4: Check shared E2E steps
- Review the shared steps injected above in the "Existing E2E shared steps" section.
- Also check existing feature files listed above for step patterns that can be reused.
- Reuse existing steps before creating new ones.

### Step 5: Design the feature file
- Create or update `e2e/features/$0.feature`.
- Write human-readable Gherkin scenarios describing the full user journey.
- Use parameterized steps with `"quoted strings"`.
- Never include CSS selectors, locators, or URLs in the feature file.

### Step 6: Implement the step definitions
- Create or update `e2e/steps/$0.steps.ts`.
- Use Playwright's `page` object for all browser interactions.
- Follow this structure:

```ts
import { test, expect } from '@playwright/test'
import { loadFeature, describeFeature } from '@amiceli/vitest-cucumber'

// Adjust the feature loading based on the Playwright + Cucumber integration used
const feature = await loadFeature('./e2e/features/$0.feature')

describeFeature(feature, ({ Scenario, BeforeEachScenario, AfterEachScenario }) => {
  let page: Page

  BeforeEachScenario(async ({ browser }) => {
    page = await browser.newPage()
  })

  AfterEachScenario(async () => {
    await page.close()
  })

  Scenario('Scenario name matching feature file', ({ Given, When, Then }) => {
    Given('the user is on the {string} page', async (_ctx, pageName: string) => {
      // Map pageName to URL -- keep URLs out of feature files
      await page.goto(getUrlForPage(pageName))
    })

    When('the user clicks {string}', async (_ctx, buttonText: string) => {
      await page.getByRole('button', { name: buttonText }).click()
    })

    Then('the user should see {string}', async (_ctx, text: string) => {
      await expect(page.getByText(text)).toBeVisible()
    })
  })
})
```

- Use Playwright locators (`getByRole`, `getByText`, `getByLabel`) -- avoid raw CSS selectors.
- Use `_ctx` as the first argument in parameterized steps.
- Use string templates (`{string}`, `{int}`) -- never regex matchers.
- Keep page URLs and selectors inside step definitions, never in feature files.

### Step 7: Run the E2E tests
- Run `npx playwright test e2e/` to execute the scenarios.
- If tests fail, fix the step definitions and re-run until green.

## Notes
- E2E tests are slow -- keep scenarios focused on critical user journeys.
- Prefer `getByRole` and `getByLabel` over `getByTestId` for accessibility.
- If a Playwright + Cucumber bridge package is not yet installed, inform the user and suggest options (e.g., `playwright-bdd` or a custom integration).
