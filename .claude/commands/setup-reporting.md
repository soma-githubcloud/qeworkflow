# /setup-reporting — Configure Test Reporting

Installs and configures the reporting stack for the active kit. For Kit-U1 this means
Playwright HTML Reporter + Allure. Safe to re-run — idempotent.

## Arguments

| Argument | Description |
|---|---|
| `--reporters <list>` | Comma-separated reporters to enable (e.g., `allure,html`) |
| `--reset` | Remove existing reporter config and start fresh |

## Execution Steps

1. **Read kit.config.json** — get `reporting`, `tech`, `outputDir`, `projectName`

2. **Determine reporters to configure**
   - Default to `reporting` array from kit.config.json
   - Override with `--reporters` if provided
   - Validate: each reporter must be supported by the kit's tech stack

3. **Spawn reporting-agent** (read `tc-to-automate/kits/_base/agents/reporting-agent.md`)
   - Pass: `reporters`, `kitConfig`, `projectPath: <outputDir>/<projectName>`, `reset`
   - Agent handles installation + config file updates

4. **Verify**
   - For Allure: check `allure-playwright` is in `package.json` devDependencies
   - Check `playwright.config.ts` contains allure reporter entry
   - For HTML: confirm `playwright.config.ts` has `{ reporter: "html" }` entry
   - Run `npm run test -- --reporter=list` (dry-run one test) to confirm reporters load

5. **Show available commands**
   ```
   npm test              → run tests with all reporters
   npm run test:allure   → run tests + generate Allure report
   npm run allure:open   → open Allure report in browser
   npx playwright show-report → open Playwright HTML report
   ```

## Supported Reporters by Kit

| Kit | Reporters |
|---|---|
| U1, U2, H1 | playwright-html, allure |
| U3, U4, A1, H2 | allure, extent |
| A2, H3 | allure, pytest-html |
