# BDD TypeScript-React Plugin for Claude Code

A Claude Code plugin that provides skills, agents, and an orchestrator for Behavior-Driven Development in TypeScript-React projects.

## What this plugin provides

### Skills (on-demand workflows)

- **`/bdd-ts-plugin:bdd-unit <ComponentName>`** -- Scaffold component-level BDD tests using `@amiceli/vitest-cucumber` and React Testing Library
- **`/bdd-ts-plugin:bdd-a11y <ComponentName>`** -- Scaffold accessibility BDD tests using `@amiceli/vitest-cucumber`, React Testing Library, and `vitest-axe` (axe-core) for WCAG compliance
- **`/bdd-ts-plugin:bdd-e2e <FlowName>`** -- Scaffold E2E BDD tests using Playwright with `playwright-bdd`
- **`/bdd-ts-plugin:bdd-drift [unit|a11y|e2e|all]`** -- Detect orphaned or mismatched BDD test files across all test suites
- **`/bdd-ts-plugin:allure-report [generate|setup]`** -- Set up and generate Allure HTML test reports

### Agents (autonomous workflows)

- **`bdd-orchestrator`** -- Takes user requirements as input, classifies the test type (unit vs accessibility vs E2E), extracts scenarios, delegates to the appropriate skill, runs the quality gate, and generates an Allure report.
- **`bdd-quality-gate`** -- Validates generated BDD test files against all plugin rules. Catches weak assertions, wrong imports, leaked implementation details, missing tags, and structural issues before tests run.

### Bundled rules (14)

The plugin enforces 14 BDD rules. The orchestrator carries the full set; each skill carries its relevant subset.

#### Shared rules (all BDD tests)
1. **Reusability First** -- parameterized steps over duplicated steps
2. **Strict UI Interaction Separation** -- no implementation details in feature files
3. **Step Matching** -- string templates only (`{string}`, `{int}`), `_ctx` as first argument

#### Unit test rules
4. **One-to-One File Mapping** -- every `.feature` has a co-located `.steps.tsx`
5. **Vitest-Cucumber Architecture** -- `describeFeature`/`Scenario` syntax, `AfterEachScenario` for cleanup

#### E2E test rules
6. **E2E File Structure** -- `e2e/features/`, `e2e/steps/`, `e2e/support/`
7. **Playwright Locators** -- accessible locators (`getByRole`, `getByLabel`), no raw CSS selectors

#### Accessibility test rules
8. **Accessibility File Naming** -- `.a11y.feature` / `.a11y.steps.tsx` co-located with component
9. **axe-core as Baseline** -- every a11y feature must include a full axe-core audit scenario
10. **Declarative Accessibility Language** -- user-facing language in features, ARIA/axe details in steps only
11. **Mandatory `@a11y` Tag** -- every a11y feature tagged `@a11y @wcag-aa` for selective execution

#### Code quality rules (unit and a11y step definitions)
12. **Assertion Quality** -- no `.toBeTruthy()`/`.toBeDefined()` for element presence; use `toBeInTheDocument()`, `toHaveAttribute()`, `toBeVisible()`, `toHaveAccessibleName()`
13. **Data-Driven Component Guards** -- assert array non-empty + assert rendered item count before iterating
14. **Interaction Coverage** -- interactive components must have at least one `userEvent` scenario

## Installation

### Option A: From a marketplace

```bash
# Add the marketplace (one time)
/plugin marketplace add Klegane/bdd-ts-plugin

# Install the plugin
/plugin install bdd-ts-plugin
```

### Option B: Local testing

```bash
claude --plugin-dir /path/to/bdd-ts-plugin
```

### Option C: Project-level (shared with team via git)

Add to your project's `.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "bdd-ts-plugin": true
  }
}
```

## Usage

### Using skills directly

```bash
/bdd-ts-plugin:bdd-unit Button          # Component-level BDD tests
/bdd-ts-plugin:bdd-a11y Button          # Accessibility BDD tests
/bdd-ts-plugin:bdd-e2e Checkout         # E2E Playwright BDD tests
/bdd-ts-plugin:bdd-drift all            # Check for orphaned test files
/bdd-ts-plugin:allure-report generate   # Generate Allure HTML report
```

### Using the orchestrator agent

Describe your requirements and the agent handles the full pipeline:

> "I need BDD tests for a SearchBar component. It should have an input with a placeholder, trigger search on Enter, and a clear button that resets the input."

The orchestrator will:
1. Classify the test type (unit / a11y / E2E)
2. Extract scenarios and present them for your confirmation
3. Delegate to the appropriate skill
4. Offer accessibility tests (if not already requested)
5. Run the quality gate to validate output against all rules
6. Run tests until green
7. Check for feature drift
8. Generate an Allure HTML report
9. Report a summary of everything created

## Requirements

Your project needs these dependencies (each skill checks for them automatically at Step 0):

- **Unit BDD tests**: `@amiceli/vitest-cucumber`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `vitest`
- **Accessibility BDD tests**: all unit deps + `vitest-axe`
- **E2E BDD tests**: `@playwright/test`, `playwright-bdd`
- **Allure reports**: `allure-vitest`, `allure-js-commons`, Allure CLI (available via `npx allure`)

## Plugin structure

```
bdd-ts-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── bdd-unit/
│   │   ├── SKILL.md
│   │   ├── evals/
│   │   │   ├── evals.json
│   │   │   └── trigger-eval.json
│   │   └── docs/
│   │       └── bdd-rules.md
│   ├── bdd-a11y/
│   │   ├── SKILL.md
│   │   ├── evals/
│   │   │   ├── evals.json
│   │   │   └── trigger-eval.json
│   │   └── docs/
│   │       └── a11y-rules.md
│   ├── bdd-e2e/
│   │   ├── SKILL.md
│   │   ├── evals/
│   │   │   ├── evals.json
│   │   │   └── trigger-eval.json
│   │   └── docs/
│   │       └── e2e-rules.md
│   ├── bdd-drift/
│   │   ├── SKILL.md
│   │   ├── evals/
│   │   │   ├── evals.json
│   │   │   └── trigger-eval.json
│   │   └── docs/
│   │       └── drift-detection.md
│   └── allure-report/
│       ├── SKILL.md
│       ├── evals/
│       │   ├── evals.json
│       │   └── trigger-eval.json
│       └── docs/
│           └── allure-setup.md
├── agents/
│   ├── bdd-orchestrator.md
│   └── bdd-quality-gate.md
├── docs/
│   └── bdd-rules.md
├── settings.json
└── LICENSE
```

## License

MIT
