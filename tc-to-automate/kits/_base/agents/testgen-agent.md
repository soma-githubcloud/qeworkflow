# TestGen Agent (Non-BDD Spec Generator)

**Single Responsibility**: Generate non-BDD Playwright spec files from `intake.summary.json`
and existing Page Object files. Does NOT generate feature files or step definitions — use
`bdd-agent.md` for BDD artifacts.

Previously the `spec-only` mode of `test-generator-agent.md`. Now a standalone agent.

---

## Inputs (passed by calling command)

| Parameter | Type | Description |
|---|---|---|
| `intakeFile` | string | Path to `intake.summary.json` |
| `pomDir` | string | Path to generated POM files directory |
| `kitConfig` | object | Full kit.config.json |
| `tags` | string[] | Tag filter — only generate tests matching these tags |
| `dryRun` | boolean | Print planned files without writing |

---

## Pre-generation checks

1. Read `intake.summary.json` — extract `testCases` array
2. If `tags` filter is provided (e.g., `["@smoke"]`): filter `testCases` to only those whose `tags` array includes any of the filter tags
3. Scan `pomDir` — catalog available POM class names (e.g., `LoginPage`, `DashboardPage`)
4. For each test case: verify all required POM classes exist in `pomDir`. If a POM is missing, abort and report: "Missing POM: LoginPage.ts. Run `/generate-page-objects` first."

---

## Generation

For each test case in filtered `testCases`:

1. Load template: `<kitConfig.templates>/spec-nonbdd.ts.tmpl`

2. Build template context:
   ```json
   {
     "kitId": "kit-u1",
     "generatedDate": "2026-03-03",
     "testCaseId": "TC-001",
     "title": "Login with valid credentials",
     "featureName": "Login",
     "preconditions": ["User is not logged in"],
     "pages": [
       { "className": "LoginPage", "instanceName": "loginPage", "importPath": "@pages/LoginPage" }
     ],
     "steps": [
       { "action": "navigate", "target": "login page",  "pageInstance": "loginPage", "method": "navigate" },
       { "action": "fill",     "target": "email field", "pageInstance": "loginPage", "method": "fillEmail", "value": "user@test.com" },
       { "action": "click",    "target": "login button","pageInstance": "loginPage", "method": "clickSubmit" },
       { "action": "assert",   "target": "redirected",  "assertion": "await expect(page).toHaveURL(/\\/dashboard/);" }
     ],
     "expectedResults": ["User is redirected to dashboard"]
   }
   ```

3. **Action → method mapping**: use the page's action methods (from POM file analysis) to map each step's `target` to the correct POM method name

4. **Assertion generation**: for expected results containing "redirect", "URL", "navigate" → use `toHaveURL`; for text content → `toHaveText`; for visibility → `toBeVisible`; for absence → `not.toBeVisible()`

5. Render template → write to `<outputDir>/<projectName>/tests/nonbdd/<TC-ID>-<slug>.spec.ts`

---

## Rules

- One spec file per test case (not one file per page)
- All steps wrapped in `test.step()` for Allure/HTML report hierarchy
- Assertions use `expect` from `@playwright/test` only
- No raw selector strings in spec files — always call POM methods
- `--dry-run`: print planned file names and step counts without writing

---

## Validation

After writing all spec files:
1. `npx tsc --noEmit` — all imports must resolve; types must be correct
2. `npx playwright test --list` — all generated specs discoverable
3. Report any type errors; fix unambiguous import path issues automatically
