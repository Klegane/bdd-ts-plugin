# Allure Report Setup Reference

Quick reference for setting up and generating Allure test reports in a vitest BDD project.

## Dependencies

```bash
npm install -D allure-vitest allure-js-commons
```

The Allure CLI is available via `npx allure` (no global install needed). If that fails:

```bash
# macOS
brew install allure

# Windows (scoop)
scoop install allure

# Global npm
npm install -g allure-commandline
```

## Vitest reporter configuration

Add to `vitest.config.ts` (or `vite.config.ts`):

```ts
export default defineConfig({
  test: {
    reporters: [
      'default',
      ['allure-vitest/reporter', {
        resultsDir: './allure-results',
      }],
    ],
  },
})
```

Key rules:
- **Always preserve existing reporters** — add Allure alongside them, never replace.
- **`resultsDir`** defaults to `./allure-results`. Keep this default.
- **`links.issue.urlTemplate`** is optional — set it if the project has a GitHub repo.

## .gitignore entries

```gitignore
allure-results/
allure-report/
```

## Report generation commands

```bash
# Run tests (produces allure-results/)
npm test -- --run

# Generate HTML report
npx allure generate ./allure-results --clean -o ./allure-report

# Open in browser
npx allure open ./allure-report
```

Selective runs:

```bash
# Only a11y tests
npm test -- --run **/*.a11y.steps.tsx

# Only one component
npm test -- --run src/components/Button/
```

## BDD tag integration

Allure automatically picks up Gherkin tags from `.feature` files:
- `@a11y` tests are grouped under the "a11y" tag in the report.
- `@wcag-aa` / `@wcag-aaa` tags show WCAG conformance level.
- Feature names become test suites; Scenario names become test cases.

No extra configuration needed — `allure-vitest` reads Gherkin metadata automatically with `@amiceli/vitest-cucumber`.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `allure-results/` empty after tests | Reporter not configured — check `vitest.config.ts` `reporters` array |
| `npx allure` not found | Install the Allure CLI (see Dependencies above) |
| Report shows stale results | Delete `allure-results/` before running, or use `--clean` on generate |
| No Gherkin steps in report | Ensure `allure-vitest` (not `allure-jest`) is used with `@amiceli/vitest-cucumber` |
