# BDD TypeScript-React Plugin for Claude Code

A Claude Code plugin that provides skills and an orchestrator agent for Behavior-Driven Development in TypeScript-React projects.

## What this plugin provides

### Skills (on-demand workflows)

- **`/bdd-ts-plugin:bdd-unit <ComponentName>`** -- Scaffold component-level BDD tests using `@amiceli/vitest-cucumber` and React Testing Library
- **`/bdd-ts-plugin:bdd-a11y <ComponentName>`** -- Scaffold accessibility BDD tests using `@amiceli/vitest-cucumber`, React Testing Library, and `vitest-axe` (axe-core) for WCAG compliance
- **`/bdd-ts-plugin:bdd-e2e <FlowName>`** -- Scaffold E2E BDD tests using Playwright
- **`/bdd-ts-plugin:allure-report`** -- Set up and generate Allure HTML test reports (handles one-time `allure-vitest` installation, vitest reporter configuration, and on-demand report generation)

### Agent (requirements-driven orchestration)

- **`bdd-orchestrator`** -- Takes user requirements as input, classifies the test type (unit vs accessibility vs E2E), extracts scenarios, delegates to the appropriate skill, and automatically generates an Allure report after tests pass.

### Bundled rules

The plugin enforces 11 BDD rules, inlined directly into the agent and each skill (each skill carries only its relevant subset; the orchestrator carries the full set):

1. Reusability First -- parameterized steps over duplicated steps
2. Strict UI Interaction Separation -- no implementation details in feature files
3. Step Matching -- string templates only, `_ctx` as first argument
4. One-to-One File Mapping -- every `.feature` has a co-located `.steps.tsx`
5. Vitest-Cucumber Architecture -- `describeFeature`/`Scenario` syntax, `AfterEachScenario` for cleanup
6. E2E File Structure -- `e2e/features/`, `e2e/steps/`, `e2e/support/`
7. Playwright Locators -- accessible locators, no raw CSS selectors
8. Accessibility File Naming -- `.a11y.feature` / `.a11y.steps.tsx` co-located with component, shared a11y steps in `src/test/sharedA11ySteps.ts`
9. axe-core as Baseline -- every a11y feature must include a full axe-core audit scenario
10. Declarative Accessibility Language -- user-facing descriptions in features, technical ARIA/axe details in steps only
11. Mandatory `@a11y` Tag -- every a11y feature is tagged `@a11y @wcag-aa` for selective execution and report filtering

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

Scaffold a unit BDD test for a component:

```
/bdd-ts-plugin:bdd-unit Button
```

Scaffold an accessibility BDD test for a component:

```
/bdd-ts-plugin:bdd-a11y Button
```

Scaffold an E2E BDD test for a user flow:

```
/bdd-ts-plugin:bdd-e2e Checkout
```

### Using the orchestrator agent

Simply describe your requirements and the agent handles everything:

> "I need BDD tests for a SearchBar component. It should have an input with a placeholder, trigger search on Enter, and a clear button that resets the input."

The agent will:
1. Classify this as a unit test (component behavior)
2. Extract scenarios and present them for your confirmation
3. Write the `.feature` file
4. Run tests (expect red)
5. Write the `.steps.tsx` file
6. Run tests until green
7. Offer to add accessibility tests (if not already requested)
8. Verify no orphaned files
9. Generate an Allure HTML report

## Requirements

Your project needs:

- **For unit BDD tests**: `@amiceli/vitest-cucumber`, `@testing-library/react`, `vitest`
- **For accessibility BDD tests**: `@amiceli/vitest-cucumber`, `@testing-library/react`, `vitest`, `vitest-axe`
- **For E2E BDD tests**: `@playwright/test` and a Playwright+Cucumber bridge (e.g., `playwright-bdd`)
- **For Allure reports**: `allure-vitest`, `allure-js-commons`, and the [Allure CLI](https://docs.qameta.io/allure/) (available via `npx allure` or installed globally)

## Plugin structure

```
bdd-ts-plugin/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── skills/
│   ├── bdd-unit/
│   │   └── SKILL.md         # Component-level BDD skill
│   ├── bdd-a11y/
│   │   └── SKILL.md         # Accessibility BDD skill
│   ├── bdd-e2e/
│   │   └── SKILL.md         # Playwright E2E BDD skill
│   └── allure-report/
│       └── SKILL.md         # Allure report setup & generation
├── agents/
│   └── bdd-orchestrator.md  # Orchestrator agent (carries all 11 rules)
├── settings.json             # Default agent settings
└── LICENSE
```

## License

MIT
