# E2E BDD Rules

These rules apply to all end-to-end BDD tests in this project. Every feature file in `e2e/features/` and step file in `e2e/steps/` must follow them.

## File structure

1. Feature files go in `e2e/features/<FlowName>.feature`.
2. Step definitions go in `e2e/steps/<FlowName>.steps.ts`.
3. Shared E2E helpers live in `e2e/support/sharedSteps.ts`.
4. One feature file per user flow; one steps file per feature file.

## Feature files (.feature)

1. One user behavior per Scenario — don't test multiple journeys in a single scenario.
2. Use parameterized steps with `"quoted strings"` for variable data.
3. Never include URLs, page paths, CSS selectors, or `data-testid` attributes.
4. Describe navigation and outcomes in user-facing language.
5. Keep steps short and declarative.

**Bad:** `When the user navigates to "http://localhost:3000/checkout"`
**Good:** `When the user navigates to the checkout page`

## Step definitions (.steps.ts)

1. Use Playwright's `page` object for all browser interactions.
2. Use accessible locators (`getByRole`, `getByText`, `getByLabel`) — avoid raw CSS selectors and `getByTestId` where possible.
3. Map page names to URLs inside step definitions, never in feature files.
4. All parameterized step callbacks must use `_ctx` as the first argument.
5. Use `{string}` and `{int}` templates for step matching — never regex.
6. Use `AfterEachScenario` to close pages / clean up state.

## Shared steps

1. Reusable E2E step logic lives in `e2e/support/sharedSteps.ts`.
2. Before writing a new step, check if a shared step already covers it.
3. Common reusable steps: page navigation, form filling, authentication, waiting for network idle.

## Playwright locator priority

| Priority | Locator | Example |
|----------|---------|---------|
| 1 | `getByRole` | `page.getByRole('button', { name: 'Submit' })` |
| 2 | `getByLabel` | `page.getByLabel('Email')` |
| 3 | `getByText` | `page.getByText('Welcome back')` |
| 4 | `getByPlaceholder` | `page.getByPlaceholder('Search...')` |
| 5 | `getByTestId` | Only as a last resort |
