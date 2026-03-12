---
name: allure-report
description: Set up and generate Allure test reports for BDD test suites. Handles one-time installation of allure-vitest, vitest reporter configuration, and on-demand report generation. Trigger this skill when the user mentions Allure, test reports, test reporting, coverage reports, or asks to "generate a report" after running tests.
argument-hint: "[generate|setup]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npm *, npx allure *)
---

# Allure Report Generation

Generate an Allure test report for the current project's BDD test suite.

## What this skill does

Allure produces rich HTML reports from test results, with per-scenario breakdowns, history trends, and failure analysis. For BDD projects, Allure maps each Gherkin scenario to a visual test case — making it easy to see which behaviors pass, fail, or are broken, without reading console output.

This skill handles two concerns:
1. **Setup** (one-time): install `allure-vitest`, configure the reporter in `vitest.config.ts`, and ensure the Allure CLI is available.
2. **Generation** (on-demand): run tests, generate the HTML report, and serve it.

## Workflow

### Step 1: Check if Allure is already configured

Look for signs that Allure is set up:

- Check `package.json` for `allure-vitest` in `devDependencies`.
- Check `vitest.config.ts` (or `vitest.config.mts` / `vite.config.ts`) for an Allure reporter entry.
- Check if `allure-results/` directory exists (indicates tests have previously run with the Allure reporter).

If all three checks pass, skip to **Step 4** (generation).
If any check fails, proceed to **Step 2** (setup).

### Step 2: Install dependencies

Install the required packages:

```bash
npm install -D allure-vitest allure-js-commons
```

Verify the Allure CLI is available. The recommended approach is using `npx` which ships with npm — no global install needed:

```bash
npx allure --version
```

If `npx allure` is not found, inform the user they need the Allure CLI:

```bash
# macOS
brew install allure

# Windows (scoop)
scoop install allure

# Or via npm globally
npm install -g allure-commandline
```

### Step 3: Configure the Vitest reporter

Read the project's Vitest config file (`vitest.config.ts`, `vitest.config.mts`, or the `test` section in `vite.config.ts`).

Add the Allure reporter to the `reporters` array. **Do not remove existing reporters** — Allure should be added alongside the default reporter so console output is preserved.

Example configuration:

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    reporters: [
      'default',
      ['allure-vitest/reporter', {
        resultsDir: './allure-results',
        links: {
          issue: { urlTemplate: 'https://github.com/OWNER/REPO/issues/%s' },
        },
      }],
    ],
  },
})
```

Key configuration decisions:
- **`resultsDir`**: defaults to `./allure-results`. Keep this default unless the user has a reason to change it.
- **`links`**: optional. If the project has a GitHub repository, configure the issue link template so Allure can link test failures to issues.
- **Preserve existing reporters**: always keep `'default'` (or whatever reporters are already configured) so the user doesn't lose console output.

After modifying the config, remind the user to add `allure-results/` and `allure-report/` to `.gitignore` if they aren't already:

```gitignore
allure-results/
allure-report/
```

### Step 4: Generate the report

Run the tests to produce fresh Allure results, then generate and open the report:

```bash
# Run all tests (generates allure-results/)
npm test -- --run

# Generate HTML report from results
npx allure generate ./allure-results --clean -o ./allure-report

# Open the report in the default browser
npx allure open ./allure-report
```

Useful variations:

```bash
# Run only unit tests
npm test -- --run src/components/

# Run only a11y tests
npm test -- --run **/*.a11y.steps.tsx

# Run a specific component's tests
npm test -- --run src/components/Button/

# Then generate from whatever ran:
npx allure generate ./allure-results --clean -o ./allure-report
```

The `--clean` flag removes previous report output before generating. This ensures the report matches the latest test run.

### Step 5: Report summary

After generating the report, provide the user with:
- The path to the generated report: `./allure-report/index.html`
- The command to re-open it later: `npx allure open ./allure-report`
- A note that `allure-results/` accumulates across runs — use `--clean` on generate or delete the directory to reset history.

## Allure with BDD tags

Allure automatically picks up Gherkin tags from `.feature` files. This means:
- `@a11y` tests appear grouped under the "a11y" tag in the Allure report, making it easy to filter accessibility results.
- `@wcag-aa` / `@wcag-aaa` tags show the conformance level in the report.
- Feature names become test suites; Scenario names become test cases.

No extra configuration is needed for this — `allure-vitest` reads the Gherkin metadata automatically when used with `@amiceli/vitest-cucumber`.

## Troubleshooting

Common issues and their fixes:

- **`allure-results/` is empty after running tests**: the reporter is not configured correctly in `vitest.config.ts`. Verify the `reporters` array includes `'allure-vitest/reporter'`.
- **`npx allure` command not found**: install the Allure CLI (see Step 2).
- **Report shows old results**: delete `allure-results/` before running tests, or use `--clean` when generating.
- **Allure report doesn't show Gherkin steps**: ensure `allure-vitest` (not `allure-jest` or other adapters) is used, and that tests use `@amiceli/vitest-cucumber`'s `describeFeature`/`Scenario` syntax.
