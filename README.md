# BDD TypeScript-React Plugin for Claude Code

A Claude Code plugin that provides skills and an orchestrator agent for Behavior-Driven Development in TypeScript-React projects.

## What this plugin provides

### Skills (on-demand workflows)

- **`/bdd-ts-plugin:bdd-unit <ComponentName>`** -- Scaffold component-level BDD tests using `@amiceli/vitest-cucumber` and React Testing Library
- **`/bdd-ts-plugin:bdd-e2e <FlowName>`** -- Scaffold E2E BDD tests using Playwright

### Agent (requirements-driven orchestration)

- **`bdd-orchestrator`** -- Takes user requirements as input, classifies the test type (unit vs E2E), extracts scenarios, and delegates to the appropriate skill. Drives the full workflow from requirements to passing tests.

### Bundled rules

The plugin bundles 7 BDD rules (in `docs/bdd-rules.md`) that are automatically enforced by both skills and the agent:

1. Reusability First -- parameterized steps over duplicated steps
2. Strict UI Interaction Separation -- no implementation details in feature files
3. Step Matching -- string templates only, `_ctx` as first argument
4. One-to-One File Mapping -- every `.feature` has a co-located `.steps.tsx`
5. Vitest-Cucumber Architecture -- `describeFeature`/`Scenario` syntax, `AfterEachScenario` for cleanup
6. E2E File Structure -- `e2e/features/`, `e2e/steps/`, `e2e/support/`
7. Playwright Locators -- accessible locators, no raw CSS selectors

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
7. Verify no orphaned files

## Requirements

Your project needs:

- **For unit BDD tests**: `@amiceli/vitest-cucumber`, `@testing-library/react`, `vitest`
- **For E2E BDD tests**: `@playwright/test` and a Playwright+Cucumber bridge (e.g., `playwright-bdd`)

## Plugin structure

```
bdd-ts-plugin/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── skills/
│   ├── bdd-unit/
│   │   └── SKILL.md         # Component-level BDD skill
│   └── bdd-e2e/
│       └── SKILL.md         # Playwright E2E BDD skill
├── agents/
│   └── bdd-orchestrator.md  # Orchestrator agent
├── docs/
│   └── bdd-rules.md         # Bundled BDD rules
├── settings.json             # Default agent settings
└── LICENSE
```

## License

MIT
