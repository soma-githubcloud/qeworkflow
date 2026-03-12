# /run-smoke — Execute Smoke Test Pack

Runs the `@smoke`-tagged test suite, generates Allure + Playwright HTML reports,
and attaches a pass/fail summary with failure details directly to the response.

## Arguments

| Argument | Description |
|---|---|
| `--tags <expr>` | Override tag filter (default: `@smoke`) |
| `--browser <name>` | Browser to use (default: chromium) |
| `--headed` | Run in headed mode (useful for debugging) |
| `--report-only` | Skip test run; only regenerate reports from existing allure-results |
| `--nonbdd-only` | Run only Playwright non-BDD specs (skip cucumber-js) |
| `--bdd-only` | Run only BDD feature files (skip Playwright specs) |

## Execution Steps

1. **Read kit.config.json** — get `outputDir`, `projectName`, `tech.testRunner`, `testStyle`

2. **Confirm project is ready**:
   - Check that `outputs/<project>/playwright.config.ts` or equivalent exists
   - Check that at least one test file exists (`tests/nonbdd/*.spec.ts` or `tests/bdd/*.feature`)
   - If not: "No tests found. Run `/generate-automation-tests` first."

3. **Run tests** (skip if `--report-only`):

   **Kit-U1/U2/H1 (Playwright)**:

   Run both suites unless `--nonbdd-only` or `--bdd-only` is set. Capture each exit code independently; a non-zero exit means test failures (not a tool error).

   **Non-BDD (Playwright specs)** — skip if `--bdd-only`:
   ```bash
   cd outputs/<project>
   npx playwright test --grep "@smoke" --project=<browser>
   ```

   **BDD (Cucumber)** — skip if `--nonbdd-only`; only runs if `tests/bdd/*.feature` files exist:
   ```bash
   cd outputs/<project>
   npx cucumber-js --tags @smoke
   ```
   If no `.feature` files exist, skip silently (do not error).

   **Kit-U3/U4 (TestNG)**:
   ```bash
   mvn test -Dgroups="smoke" -q
   ```

   **Kit-A2/H3 (pytest)**:
   ```bash
   pytest -m smoke --alluredir=allure-results
   ```

4. **Generate reports** (always runs, even on test failures):
   ```bash
   npx allure generate allure-results --output allure-report
   ```

5. **Parse results** from Playwright's JSON reporter or Allure result files:
   - Total: passed / failed / skipped — aggregate across both non-BDD and BDD runs
   - List of failed test names + first error line from each

6. **Report to user**:
   ```
   Smoke Run Complete
   ──────────────────────────────────────
   Non-BDD (Playwright):  1 passed / 0 failed
   BDD (Cucumber):        1 passed / 0 failed
   ──────────────────────────────────────
   Total Passed:  2
   Total Failed:  0
   Skipped:       0

   Reports:
   → Playwright HTML: outputs/<project>/playwright-report/index.html
   → Allure:          outputs/<project>/allure-report/index.html
   → Open with: npx allure open allure-report
   ```

## Notes
- Failures in this command are TEST failures, not system errors — do not treat as a crash
- If 0 tests ran (no @smoke tags found in either runner): warn and suggest `/validate-kit` to check test discovery
- Always generate reports regardless of test outcome
- BDD results are written to `allure-results/` by `allure-cucumberjs`; non-BDD results by `allure-playwright` — both feed the same Allure report
