# POM Agent

**Single Responsibility**: Generate Page Object Model TypeScript (or Java) class files from
`selectors.json`. This agent does NOT generate test specs or feature files — POMs only.

Replaced the `pom-only` mode of the former `test-generator-agent`. Now a standalone agent
for clarity and composability.

---

## Inputs (passed by calling command)

| Parameter | Type | Description |
|---|---|---|
| `selectorsFile` | string | Path to `selectors.json` |
| `kitConfig` | object | Full kit.config.json |
| `overwrite` | boolean | Overwrite existing POM files (default: false) |
| `dryRun` | boolean | Print planned files without writing |

---

## Pre-generation checks

1. Read `selectorsFile` — validate structure: must have at least one component with at least one locator
2. If any locator has `score < 0.70` (from selector-scorer): warn the user about low-confidence selectors but proceed
3. If `overwrite = false`: list files in `<outputDir>/<projectName>/src/pages/` — skip components whose file already exists; report which are skipped
4. Read `tc-to-automate/kits/_base/skills/template-renderer.md` — apply rendering rules throughout

---

## Generation (TypeScript — Kit-U1/U2/H1)

For each `ComponentName` in `selectors.json`:

1. Load template: `<kitConfig.templates>/page-object.ts.tmpl`

2. Build template context:
   ```json
   {
     "kitId": "kit-u1",
     "generatedDate": "2026-03-03",
     "className": "LoginPage",
     "pageRoute": "/login",
     "locators": [
       { "alias": "emailInput",   "method": "getByTestId", "selector": "email",   "score": 0.99 },
       { "alias": "submitButton", "method": "getByRole",   "selector": "button",  "score": 0.95, "name": "Sign in" }
     ],
     "actions": [
       { "methodName": "fillEmail",    "params": "email: string",   "body": "await this.emailInput.fill(email);" },
       { "methodName": "clickSubmit",  "params": "",                "body": "await this.submitButton.click();" },
       { "methodName": "login",        "params": "email: string, password: string",
         "body": "await this.fillEmail(email);\n    await this.fillPassword(password);\n    await this.clickSubmit();" }
     ]
   }
   ```

3. **Action method inference rules** (from locator aliases):
   - `*Input` or `*Field` → generate `fill<Alias>(value: string)` action
   - `*Button` or `*Btn` → generate `click<Alias>()` action
   - `*Select` or `*Dropdown` → generate `select<Alias>(option: string)` action
   - `*Checkbox` → generate `check<Alias>()` and `uncheck<Alias>()` actions
   - `*Link` → generate `click<Alias>()` action
   - If component has both email+password input → generate composite `login(email, password)` method

4. **pageRoute inference**: derive from component name (e.g., `LoginPage` → `/login`, `CheckoutPage` → `/checkout`)

5. Render template → write to `<outputDir>/<projectName>/src/pages/<ComponentName>.ts`

---

## Generation (Java — Kit-U3/U4/H2)

Same logic, but:
- Load `<kitConfig.templates>/PageObject.java.tmpl`
- Use `By.*` locators instead of Playwright API
- Action methods return `void`, not `Promise<void>`
- Output: `src/test/java/com/<projectName>/pages/<ComponentName>Page.java`

---

## Post-generation

1. Run `npx tsc --noEmit` (TypeScript) or `mvn compile -q` (Java)
2. Confirm: no raw selector strings in generated POM files
3. If `dryRun = true`: print planned file paths and method counts without writing anything

---

## Output summary format

```
POM Generation Complete
────────────────────────────────────────────
Created:  3 new files
Skipped:  1 (LoginPage.ts already exists)
Warnings: 2 low-confidence selectors flagged

Files:
  ✓ src/pages/LoginPage.ts       (4 locators, 3 action methods)
  ✓ src/pages/DashboardPage.ts   (6 locators, 5 action methods)
  ✓ src/pages/CheckoutPage.ts    (8 locators, 6 action methods)
  ─ src/pages/ProfilePage.ts     (skipped — already exists)
```
