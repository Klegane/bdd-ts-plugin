# Accessibility BDD Rules

These rules apply to all accessibility BDD tests in this project. Every `.a11y.feature` and `.a11y.steps.tsx` file must follow them.

## Feature files (.a11y.feature)

1. One a11y feature file per component, colocated: `src/components/<Name>/<Name>.a11y.feature`.
2. Always include `@a11y @wcag-aa` (or `@wcag-aaa`) tags at the Feature level.
3. Always include a baseline scenario: "No automated accessibility violations are detected".
4. One accessibility behavior per Scenario.
5. Use parameterized steps with `"quoted strings"` for variable data.
6. Describe accessibility outcomes in user-facing language, not ARIA jargon.
7. Never include `aria-*` attribute names, `role=` values, axe-core rule IDs, or CSS selectors.

**Bad:** `Then the element has aria-label="Close dialog"`
**Good:** `Then the close button has an accessible name`

## Step definitions (.a11y.steps.tsx)

1. One steps file per component, colocated: `src/components/<Name>/<Name>.a11y.steps.tsx`.
2. Import `axe` and `toHaveNoViolations` from `vitest-axe`.
3. Call `expect.extend(toHaveNoViolations)` at the top of the file.
4. Pass `render().container` to `axe()` — never a screen query result.
5. Use `runOnly: { type: 'tag', values: [...] }` for targeted axe checks.
6. Use `loadFeature` and `describeFeature` from `@amiceli/vitest-cucumber`.
7. Use `AfterEachScenario(() => { cleanup() })` — never `beforeEach`.
8. All parameterized step callbacks must use `_ctx` as the first argument.
9. Use `{string}` and `{int}` templates for step matching — never regex.
10. Use `userEvent.tab()` and `userEvent.keyboard('{Enter}')` for keyboard testing.

## Shared a11y steps

1. Reusable a11y step logic lives in `src/test/sharedA11ySteps.ts`.
2. Before writing a new step, check if a shared a11y step already covers it.
3. Common reusable steps: baseline axe audit, targeted rule check, keyboard navigability, alt text check, label association check.

## WCAG quick reference (AA)

| Principle | Key Criteria | Test With |
|-----------|-------------|-----------|
| Perceivable | 1.1.1 Non-text Content, 1.4.3 Contrast | axe audit, targeted color-contrast rule |
| Operable | 2.1.1 Keyboard, 2.4.3 Focus Order | userEvent.tab(), toHaveFocus() |
| Understandable | 3.3.1 Error Identification, 3.3.2 Labels | getByRole, getByLabelText |
| Robust | 4.1.2 Name/Role/Value | axe audit, toHaveAccessibleName() |
