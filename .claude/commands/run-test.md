# /run-test — Execute Test Test Pack

Runs the `@test`-tagged test suite, generates Allure + Playwright HTML reports,
and attaches a pass/fail summary with failure details directly to the response.

## Arguments

| Argument | Description |
|---|---|
| `--tags <expr>` | Override tag filter (default: `@test`) |
| `--browser <name>` | Browser to use (default: chromium) |
| `--headed` | Run in headed mode (useful for debugging) |
| `--report-only` | Skip test run; only regenerate reports from existing allure-results |

## Execution Steps

1. **Read kit.config.json** — get `outputDir`, `projectName`, `tech.testRunner`

2. **Confirm project is ready**:
   - Check that `outputs/<project>/playwright.config.ts` or equivalent exists
   - Check that at least one test file exists (`tests/nonbdd/*.spec.ts` or `tests/bdd/*.feature`)
   - If not: "No tests found. Run `/generate-automation-tests` first."

3. **Run tests** (skip if `--report-only`):

   **Kit-U1/U2/H1 (Playwright)**:
   ```bash
   cd outputs/<project>
   npx playwright test --grep "@test" --project=<browser>
   ```
   Capture exit code. A non-zero exit means test failures (not a tool error).

   **Kit-U3/U4 (TestNG)**:
   ```bash
   mvn test -Dgroups="test" -q
   ```

   **Kit-A2/H3 (pytest)**:
   ```bash
   pytest -m test --alluredir=allure-results
   ```

4. **Generate reports** (always runs, even on test failures):
   ```bash
   npx allure generate allure-results --output allure-report
   ```

5. **Parse results** from Playwright's JSON reporter or Allure result files:
   - Total: passed / failed / skipped
   - List of failed test names + first error line from each

6. **Report to user**:
   ```
   Test Run Complete
   ──────────────────────────────────────
   Passed:  12 / 15
   Failed:  3
   Skipped: 0

   FAILED TESTS:
   ✗ TC-003: Checkout with credit card → AssertionError: expected URL /confirmation, got /error
   ✗ TC-007: Password reset email → Timeout waiting for email-sent banner
   ✗ TC-012: Search with special chars → expect(locator).toHaveText failed

   Reports:
   → Playwright HTML: outputs/<project>/playwright-report/index.html
   → Allure:          outputs/<project>/allure-report/index.html
   → Open with: npm run allure:open
   ```

## Notes
- Failures in this command are TEST failures, not system errors — do not treat as a crash
- If all 0 tests ran (no @test tags found): warn and suggest `/validate-kit` to check test discovery
- Always generate reports regardless of test outcome
