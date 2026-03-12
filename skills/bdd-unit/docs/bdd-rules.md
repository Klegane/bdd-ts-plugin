# BDD Rules

These rules apply to all component-level BDD tests in this project. Every `.feature` and `.steps.tsx` file must follow them.

## Feature files (.feature)

1. One feature file per component, colocated: `src/components/<Name>/<Name>.feature`.
2. One behavior per Scenario — don't test multiple things in a single scenario.
3. Use parameterized steps with `"quoted strings"` for variable data.
4. Never hardcode variants into step names — use parameters instead.
5. Never include CSS selectors, `data-testid`, class names, or any implementation detail.
6. Scenario names should describe the behavior from the user's perspective.
7. Keep steps short and declarative. Prefer `When I click the submit button` over `When I find the button element and trigger a click event on it`.

## Step definitions (.steps.tsx)

1. One steps file per component, colocated: `src/components/<Name>/<Name>.steps.tsx`.
2. Use `loadFeature` and `describeFeature` from `@amiceli/vitest-cucumber` — never `describe`/`it` from vitest directly.
3. Use `AfterEachScenario(() => { cleanup() })` — never `beforeEach` or `afterEach` from vitest.
4. All parameterized step callbacks must use `_ctx` as the first argument.
5. Use `{string}` and `{int}` templates for step matching — never regex.
6. Import `render`, `screen`, `cleanup` from `@testing-library/react/pure`.
7. Import `userEvent` from `@testing-library/user-event`.
8. Import `@testing-library/jest-dom/vitest` for semantic matchers.
9. Prefer `screen.getByRole` and accessible queries over `getByTestId`.

## Assertion quality

1. **NEVER** use `.toBeTruthy()` or `.toBeDefined()` to assert element existence — `getBy*` already throws if missing, so `.toBeTruthy()` is a no-op.
2. Use `toBeInTheDocument()` for presence checks.
3. Use `toHaveAttribute('name', 'value')` for attribute assertions (href, target, rel, type, etc.).
4. Use `toHaveTextContent()` for text assertions when you don't need an exact element match.
5. Use `toBeVisible()` when testing that an element is not just in the DOM but visible.
6. Use `toHaveAccessibleName()` for accessible name verification.
7. Use `userEvent` (never `fireEvent`) for all interaction assertions — it simulates real browser behavior.

## Data-driven components

1. When testing a component that renders items from a data array, first assert that the array is non-empty: `expect(items.length).toBeGreaterThan(0)`.
2. Assert the expected count of rendered items: `expect(screen.getAllByRole('listitem')).toHaveLength(items.length)`.
3. Never iterate over a data array to make assertions without the above guards — an empty array means zero assertions and a silently passing test.

## Shared steps

1. Reusable step logic lives in `src/test/sharedSteps.ts`.
2. Before writing a new step, check if a shared step already covers it.
3. If you write a step that could be useful across 3+ components, propose moving it to shared steps.

## Drift guard

1. Every `.feature` file must have a matching `.steps.tsx` file and vice versa.
2. Run `npm test -- --run src/test/featureDrift` to catch orphaned files.
3. Never delete a component without deleting its `.feature` and `.steps.tsx` files.
