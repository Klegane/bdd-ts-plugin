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

Before writing new E2E step definitions, check these files for reusable logic:

1. Read `e2e/support/sharedSteps.ts` (if it exists) — it contains shared E2E step helpers (page navigation, authentication, form filling).
2. Use Glob to find all `e2e/features/*.feature` files and scan them for step patterns you can reuse.
3. Use Glob to find all `e2e/steps/*.steps.ts` files and check for shared imports or helpers.

Duplicating what already exists wastes effort and creates maintenance burden.

## Workflow (Order of Operations)

Follow these steps strictly in order:

### Step 0: Validate prerequisites

Before starting, verify these packages exist in `package.json` `devDependencies`:
- `@playwright/test`
- `playwright-bdd`

If any are missing, inform the user and offer to install them before proceeding:
```bash
npm install -D @playwright/test playwright-bdd
npx playwright install
```

Also check that `playwright.config.ts` exists. If `playwright-bdd` is newly installed, it may need configuration — add `testDir` pointing to a generated `.features-gen` directory and configure the `defineBddConfig` helper per `playwright-bdd` docs.

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
- Follow the instructions in the "Existing E2E shared steps" section above to find reusable step logic.
- Read `e2e/support/sharedSteps.ts` and scan existing `e2e/features/*.feature` files for step patterns that can be reused.
- Reuse existing steps before creating new ones.

### Step 5: Design the feature file
- Create or update `e2e/features/$0.feature`.
- Write human-readable Gherkin scenarios describing the full user journey.
- Use parameterized steps with `"quoted strings"`.
- Never include CSS selectors, locators, or URLs in the feature file.

### Step 6: Implement the step definitions
- Create or update `e2e/steps/$0.steps.ts`.
- Use Playwright's `page` object for all browser interactions.
- This skill uses `playwright-bdd` to connect Playwright with Gherkin. Follow this structure:

```ts
import { createBdd } from 'playwright-bdd'

const { Given, When, Then } = createBdd()

// URL mapping — keep URLs out of feature files
const pages: Record<string, string> = {
  home: '/',
  checkout: '/checkout',
  login: '/login',
}

Given('the user is on the {string} page', async ({ page }, pageName: string) => {
  await page.goto(pages[pageName] ?? `/${pageName}`)
})

When('the user clicks {string}', async ({ page }, buttonText: string) => {
  await page.getByRole('button', { name: buttonText }).click()
})

When('the user fills in {string} with {string}', async ({ page }, label: string, value: string) => {
  await page.getByLabel(label).fill(value)
})

Then('the user should see {string}', async ({ page }, text: string) => {
  await expect(page.getByText(text)).toBeVisible()
})

Then('the user is redirected to the {string} page', async ({ page }, pageName: string) => {
  await expect(page).toHaveURL(new RegExp(pages[pageName] ?? pageName))
})
```

Key conventions:
- **`playwright-bdd`** bridges Playwright and Gherkin — it reads `.feature` files and maps them to step definitions. Do NOT use `@amiceli/vitest-cucumber` here (that is for component-level vitest tests only).
- Use Playwright locators (`getByRole`, `getByText`, `getByLabel`) — avoid raw CSS selectors.
- Playwright-bdd passes `{ page }` as the first argument (destructured fixtures), not `_ctx`. This differs from vitest-cucumber's convention.
- Use string templates (`{string}`, `{int}`) — never regex matchers.
- Keep page URLs and selectors inside step definitions, never in feature files.

### Step 7: Run the E2E tests
- Run `npx bddgen && npx playwright test` to generate test files from features and execute them.
- If tests fail, fix the step definitions and re-run until green.

## Notes
- E2E tests are slow — keep scenarios focused on critical user journeys.
- Prefer `getByRole` and `getByLabel` over `getByTestId` for accessibility.
- The `playwright-bdd` package must be installed: `npm install -D playwright-bdd`. If not installed, inform the user and install it before proceeding.
- The project's `playwright.config.ts` must include the `playwright-bdd` configuration. If missing, add it during Step 0.

## Post-completion

After all E2E tests pass, remind the user about:

1. **Accessibility tests**: "Would you like me to add accessibility (WCAG) tests for any of the components exercised in this flow? Use `/bdd-ts-plugin:bdd-a11y <ComponentName>` to scaffold a11y tests."
2. **Allure report**: "Run `/bdd-ts-plugin:allure-report` to generate an HTML test report from these results."
3. **Drift guard**: Check that each `e2e/features/*.feature` has a matching `e2e/steps/*.steps.ts` and vice versa.
