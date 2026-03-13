# BDD TypeScript-React Plugin — Technical Specification

This document describes what the plugin does, why each decision was made, and the technical details needed to recreate it from scratch.

---

## 1. Purpose

This Claude Code plugin automates BDD test scaffolding for TypeScript-React projects. It generates Gherkin `.feature` files and their corresponding step definition files (`.steps.tsx`) from natural-language requirements, enforcing a strict rule set that prevents common testing anti-patterns.

### Problem it solves

Writing BDD tests for React components requires:
- Understanding Gherkin syntax and BDD conventions
- Knowing the `@amiceli/vitest-cucumber` API (which differs from `jest-cucumber` and standard `cucumber-js`)
- Following React Testing Library best practices (accessible queries, `userEvent` over `fireEvent`, `/pure` imports)
- Avoiding weak assertions (`.toBeTruthy()` on `getBy*` queries) that give false confidence
- Keeping feature files free of implementation details
- Maintaining file pairing discipline (every `.feature` needs a `.steps.tsx`)

This plugin encodes all of these concerns into skills that Claude Code can execute, ensuring every generated test follows the same standards.

### Why BDD specifically

BDD tests serve as executable specifications: the `.feature` file is a human-readable contract that product managers can validate, while the `.steps.tsx` file is the automation layer. This separation means:
- Tests survive refactoring (rename CSS classes, change component internals — the feature file doesn't change)
- Non-developers can review and contribute to test scenarios
- Test failures describe which *behavior* broke, not which DOM element moved

---

## 2. Architecture

### Plugin structure

```
bdd-ts-plugin/
├── .claude-plugin/plugin.json      # Plugin manifest (name, version, entry points)
├── agents/
│   ├── bdd-orchestrator.md         # Main agent — classifies, delegates, verifies
│   └── bdd-quality-gate.md         # Validation agent — checks output against rules
├── skills/
│   ├── bdd-unit/                   # Component-level BDD tests
│   ├── bdd-a11y/                   # WCAG accessibility BDD tests
│   ├── bdd-e2e/                    # Playwright E2E BDD tests
│   ├── bdd-drift/                  # Orphaned file detection
│   └── allure-report/              # Test report generation
├── docs/
│   └── bdd-rules.md                # Canonical rule reference (11 core rules)
├── settings.json                   # Default agent settings
└── README.md
```

Each skill directory follows the Agent Skills 2.0 format:
```
skills/<name>/
├── SKILL.md                        # Skill definition with frontmatter + workflow
├── evals/
│   ├── evals.json                  # Evaluation test cases (3-6 per skill)
│   └── trigger-eval.json           # Trigger classification tests (12-18 per skill)
└── docs/
    └── <topic>.md                  # Condensed reference documentation
```

### Execution model

```
User request
    ↓
┌─── Direct invocation ──────────────┐  ┌─── Via orchestrator ──────────────────────┐
│ /bdd-ts-plugin:bdd-unit Button     │  │ "Test the Button component"               │
│    ↓                               │  │    ↓                                      │
│ Skill executes (Step 0..10)        │  │ Phase 1: Classify → unit                  │
│    ↓                               │  │ Phase 2: Extract scenarios → confirm       │
│ Post-completion reminders          │  │ Phase 3: Delegate to bdd-unit skill        │
│ (a11y offer, Allure, drift)        │  │ Phase 3.5: Offer a11y tests               │
└────────────────────────────────────┘  │ Phase 4: Quality gate validation           │
                                        │ Phase 5: Run tests until green             │
                                        │ Phase 6: Drift check                       │
                                        │ Phase 7: Allure report                     │
                                        │ Phase 8: Summary                           │
                                        └───────────────────────────────────────────┘
```

Key difference: direct skill invocation skips the quality gate, drift check, Allure report, and a11y offer. The orchestrator provides the full pipeline. Each skill adds post-completion reminders to bridge this gap when invoked directly.

---

## 3. Rule system

### Why 14 rules instead of guidelines

Rules are encoded as hard constraints, not suggestions. Every rule exists because its violation was observed in real test output and caused a specific category of problem:

| Rule | Problem it prevents |
|------|-------------------|
| 1. Reusability First | Duplicated step definitions that diverge over time |
| 2. Strict UI Separation | Brittle tests that break when CSS changes |
| 3. Step Matching | Regex matchers that are hard to read and maintain |
| 4. One-to-One File Mapping | Orphaned test files that create confusion |
| 5. Vitest-Cucumber Architecture | Mixing `beforeEach` with `AfterEachScenario` causes ordering bugs |
| 6. E2E File Structure | E2E tests mixed with unit tests create confusing test runs |
| 7. Playwright Locators | `getByTestId` doesn't verify accessibility |
| 8. A11y File Naming | A11y tests mixed with unit tests make selective execution impossible |
| 9. axe-core as Baseline | Without a baseline audit, obvious violations slip through |
| 10. Declarative A11y Language | ARIA jargon in features makes them unreadable by non-devs |
| 11. Mandatory @a11y Tag | Without tags, a11y tests can't be filtered in CI or reports |
| 12. Assertion Quality | `.toBeTruthy()` on `getBy*` is a no-op — it can never fail |
| 13. Data-Driven Guards | Empty arrays produce zero assertions, silently passing |
| 14. Interaction Coverage | Static-only tests miss broken click handlers |

### Rule distribution across files

Each skill carries only the rules relevant to it (to avoid noise), but the orchestrator carries all 14 as the authoritative source:

| Rule | Orchestrator | bdd-unit | bdd-a11y | bdd-e2e | bdd-drift |
|------|:---:|:---:|:---:|:---:|:---:|
| 1-3 (Shared) | All | All | All | All | — |
| 4-5 (Unit) | All | All | 5 only | — | — |
| 6-7 (E2E) | All | — | — | All | — |
| 8-11 (A11y) | All | — | All | — | — |
| 12-14 (Quality) | All | All | 12 | — | — |

### Import conventions

All `.steps.tsx` files (unit and a11y) MUST use:
```tsx
import { render, screen, cleanup } from '@testing-library/react/pure'
import userEvent from '@testing-library/user-event'
import { loadFeature, describeFeature } from '@amiceli/vitest-cucumber'
import '@testing-library/jest-dom/vitest'
```

The `/pure` import is critical — `@testing-library/react` (without `/pure`) auto-registers cleanup in `afterEach`, which conflicts with vitest-cucumber's `AfterEachScenario`. The `/pure` variant disables this, giving the test author full control.

The `@testing-library/jest-dom/vitest` import (not `@testing-library/jest-dom`) registers matchers via Vitest's `expect.extend`, not Jest's. Without this specific import path, matchers like `toBeInTheDocument()` are undefined.

---

## 4. Skills — technical details

### bdd-unit

**Purpose**: Scaffold `.feature` + `.steps.tsx` for a single React component.

**Workflow** (11 steps):
0. Validate prerequisites (check `package.json` for deps)
1. Gather requirements (read component source)
2. Extract scenarios (one behavior per scenario, present table)
3. Check existing tests (incremental updates, not overwrites)
4. Check shared steps (read `src/test/sharedSteps.ts`, scan existing features)
5. Design feature file (declarative Gherkin, parameterized steps)
6. Run tests — expect failures (validates feature syntax)
7. Implement step definitions (RTL + vitest-cucumber)
8. Run tests — expect green
9. Verify drift guard
10. Post-completion reminders

**Key technical decisions**:
- `AfterEachScenario` (not `beforeEach`) — vitest-cucumber runs each Step as a separate Vitest test. `beforeEach` would reset state between steps within the same scenario, which breaks multi-step scenarios.
- `_ctx` as first argument — vitest-cucumber passes a context object as the first parameter to every step callback. Using `_ctx` (underscore prefix) signals "intentionally unused" to TypeScript and makes actual parameters (variant, text, etc.) visually prominent.
- `{string}` and `{int}` templates — vitest-cucumber has built-in parameter type parsing that extracts quoted strings and integers. Regex matchers bypass this, producing less readable code and losing type safety.

### bdd-a11y

**Purpose**: Scaffold `.a11y.feature` + `.a11y.steps.tsx` for WCAG accessibility testing.

**Key technical decisions**:
- Separate file infix (`.a11y.`) — enables file-level selective execution (`**/*.a11y.steps.tsx`) without affecting unit test glob patterns.
- `@a11y @wcag-aa` tags — enable report-level filtering (Allure picks up Gherkin tags automatically) and CI pipeline separation.
- `vitest-axe` (not `@axe-core/react`) — `vitest-axe` provides Vitest-compatible matchers (`toHaveNoViolations`). `@axe-core/react` is designed for runtime logging, not test assertions.
- `render().container` passed to `axe()` — axe-core needs a DOM node. Screen queries return elements, not containers. Passing `screen.getByRole(...)` to `axe()` would only audit that single element, not the full component tree.
- `expect.extend({ toHaveNoViolations })` — must be called as an object spread, not `expect.extend(toHaveNoViolations)`. The matcher module exports a named function, not an object.
- `runOnly: { type: 'tag', values: [...] }` — enables targeted axe checks (e.g., only color-contrast) for scenarios that test specific WCAG criteria. The baseline scenario runs all rules; subsequent scenarios can narrow scope.

### bdd-e2e

**Purpose**: Scaffold `e2e/features/*.feature` + `e2e/steps/*.steps.ts` for Playwright E2E tests.

**Key technical decisions**:
- `playwright-bdd` (not `@amiceli/vitest-cucumber`) — vitest-cucumber is a Vitest integration; Playwright has its own runner. `playwright-bdd` bridges Playwright's test runner with Gherkin `.feature` files via `createBdd()`.
- `{ page }` (not `_ctx`) — playwright-bdd passes Playwright fixtures as the first argument, destructured. This differs from vitest-cucumber's convention.
- `npx bddgen && npx playwright test` — playwright-bdd pre-generates test files from features before Playwright runs them. The `bddgen` step is required.
- URL mapping in step definitions — feature files say "the checkout page"; step definitions map `"checkout"` → `/checkout`. This keeps URLs out of feature files (Rule 2).

### bdd-drift

**Purpose**: Detect orphaned or mismatched test files across all suites.

**Technical approach**:
- Glob-based scanning: finds all `.feature` and `.steps.tsx` files, then verifies each has a counterpart.
- Component existence check: for unit/a11y tests, also verifies the source component (`.tsx`) still exists.
- Three scan modes: `unit` (src, excluding `.a11y.`), `a11y` (src, `.a11y.` files only), `e2e` (`e2e/` directory), `all` (everything).
- Excludes shared step files (`sharedSteps.ts`, `sharedA11ySteps.ts`, `e2e/support/sharedSteps.ts`) from orphan detection.

### allure-report

**Purpose**: Set up Allure reporting and generate HTML reports.

**Key technical decisions**:
- `allure-vitest/reporter` (not `allure-jest`) — must match the test runner (Vitest, not Jest).
- Preserve existing reporters — always add Allure alongside `'default'` (and any other configured reporters). Never replace.
- `--clean` flag on generate — ensures the report matches the latest test run, not accumulated history.
- BDD tag integration is automatic — `allure-vitest` reads Gherkin tags from vitest-cucumber's metadata. No additional configuration needed for `@a11y`, `@wcag-aa` to appear in reports.

---

## 5. Agents — technical details

### bdd-orchestrator

**Purpose**: End-to-end workflow from requirements to passing tests + report.

**8-phase pipeline**:
1. **Classify** — determines unit vs. a11y vs. E2E based on requirement language. Uses ordering: unit > a11y > E2E to prevent "keyboard navigation" from being misrouted to E2E.
2. **Extract scenarios** — maps each requirement to a Gherkin Scenario. Presents a confirmation table. Does NOT write files until user approves.
3. **Delegate** — invokes the appropriate skill with confirmed scenario context.
4. **Quality gate** — invokes `bdd-quality-gate` agent to validate output against all 14 rules before running tests.
5. **Verify** — runs tests. Fixes step definitions if they fail. Repeats until green.
6. **Drift check** — invokes `bdd-drift all` to catch orphaned files.
7. **Allure report** — invokes `allure-report` skill for HTML report generation.
8. **Summary** — reports what was created, quality gate results, test results, drift status, and report location.

**Phase 3.5: Offer a11y** — after unit or E2E tests pass, always offers to add accessibility tests. Skipped if user already requested a11y.

### bdd-quality-gate

**Purpose**: Validate generated files against all rules before tests run.

**Validation categories**:
- Feature file rules (no selectors, parameterized steps, declarative language, tags)
- Step definition rules (correct imports, `AfterEachScenario`, `_ctx`, templates, semantic matchers)
- Assertion quality (no `.toBeTruthy()`, semantic matchers used)
- Data-driven guards (empty-array protection, count assertions)
- Interaction coverage (at least one `userEvent` scenario)
- A11y-specific (vitest-axe imports, `expect.extend`, `container` passed to `axe()`)
- File structure (correct directories, paired files)

**Output**: checklist report with pass/fail for each check. If any fail, the orchestrator stops and fixes before running tests.

---

## 6. Evaluation system

Each skill has two evaluation files:

### evals.json
- 3-6 test cases per skill
- Each case has: `prompt` (user input), `expected_output` (what should happen), `assertions` (specific checkable outcomes)
- Happy path cases: test normal skill execution
- Error cases: test dependency missing, component not found, empty results

### trigger-eval.json
- 12-18 classification examples per skill
- Each entry: `{ "query": "...", "should_trigger": true/false }`
- Positive examples: queries that should activate this skill
- Negative examples: queries that should NOT activate this skill (belong to a different skill or are unrelated)

---

## 7. Recreating from scratch

### Minimum viable plugin

To recreate the core functionality:

1. **Create the directory structure** with `.claude-plugin/plugin.json` declaring the plugin name and version.

2. **Write the bdd-unit skill** (`skills/bdd-unit/SKILL.md`):
   - YAML frontmatter with name, description, argument-hint, allowed-tools
   - Rules 1-5, 12-14 inlined
   - 11-step workflow (Step 0 through Step 10)
   - Code template showing `@testing-library/react/pure`, `@testing-library/jest-dom/vitest`, vitest-cucumber setup

3. **Write the orchestrator** (`agents/bdd-orchestrator.md`):
   - YAML frontmatter with name and description
   - All 14 rules inlined
   - Unit test guidelines (AAA, zero control logic, CQS, observable behavior)
   - BDD guidelines (declarative, single When, ubiquitous language)
   - Anti-patterns list
   - 8-phase workflow

4. **Add remaining skills** (bdd-a11y, bdd-e2e, bdd-drift, allure-report) following the same pattern.

5. **Add the quality gate agent** for output validation.

6. **Add evals** for each skill (Agent Skills 2.0 format).

### Critical implementation notes

- **Never use `@testing-library/react` without `/pure`** in vitest-cucumber contexts. The auto-cleanup will conflict.
- **Never use `@amiceli/vitest-cucumber` in Playwright E2E tests.** Use `playwright-bdd` instead.
- **Shell injection syntax (`!` backticks`) does not work** in Claude Code skill markdown. Use explicit instructions to read files with the Read/Glob tools.
- **Rules must be inlined** into each file because the `docs/` folder is not uploaded to the marketplace. External references will break.
- **Rule numbering must be consistent** across all files. The orchestrator is the authoritative source. Skills use the same numbers for shared rules and extend from 12+ for skill-specific rules.

### Dependencies map

```
bdd-unit requires:
  @amiceli/vitest-cucumber
  @testing-library/react
  @testing-library/jest-dom
  @testing-library/user-event
  vitest

bdd-a11y requires:
  (all bdd-unit deps)
  vitest-axe

bdd-e2e requires:
  @playwright/test
  playwright-bdd

allure-report requires:
  allure-vitest
  allure-js-commons
  allure CLI (npx allure)
```

---

## 8. Design decisions log

| Decision | Alternatives considered | Why this choice |
|----------|------------------------|-----------------|
| Separate skills (not one merged skill) | Single monolithic skill | Different toolchains (RTL vs Playwright), different file conventions, cleaner trigger classification |
| `playwright-bdd` for E2E | `@amiceli/vitest-cucumber` with Playwright | vitest-cucumber is designed for Vitest; Playwright has its own runner |
| Rules inlined (not external reference) | `docs/bdd-rules.md` referenced from skills | `docs/` not uploaded to marketplace; broken references at runtime |
| Quality gate as separate agent | Validation embedded in orchestrator | Separation of concerns; reusable independently; keeps orchestrator focused on workflow |
| bdd-drift as standalone skill | Drift check embedded in other skills | Can be invoked independently; covers all test types; doesn't bloat individual skill files |
| 14 rules (not 11) | Keep assertion quality as anti-pattern | Formal rules are enforceable; anti-patterns are advisory. Promoted based on real test output violations. |
| `/pure` import for RTL | Standard `@testing-library/react` | Auto-cleanup conflicts with `AfterEachScenario` in vitest-cucumber |
| `@testing-library/jest-dom/vitest` | `@testing-library/jest-dom` | Vitest-specific import path registers matchers via Vitest's `expect.extend` |
| Post-completion reminders in skills | Only orchestrator offers follow-ups | Direct skill invocation skips orchestrator; reminders bridge the gap |
| Error scenario evals | Only happy-path evals | Skills must handle missing deps, nonexistent components, empty results gracefully |
