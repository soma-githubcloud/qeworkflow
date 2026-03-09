# Framework Analyzer Agent (Kit-U2)

**Single Responsibility**: Analyze an existing Playwright TypeScript project's structure and
produce a `frameworkMap` that the test-generator-agent uses to integrate correctly.

## Steps

1. **Detect project root**: Look for `playwright.config.ts` or `playwright.config.js`

2. **Extract config**: Read `playwright.config.ts`:
   - `testDir` — where test specs live
   - `baseURL` — application URL
   - `reporter` — currently configured reporters
   - `projects` — browser targets in use

3. **Catalog existing Page Objects**:
   - Scan all `.ts` files in the detected POM directory
   - Extract: class name, locator property names, action method signatures
   - Detect: base class (if any), import style, locator API used

4. **Detect naming conventions**:
   - File naming: `LoginPage.ts` vs `login.page.ts` vs `loginPage.ts`
   - Locator style: `getByTestId` vs `locator()` vs `$('...')`
   - Directory: `pages/` vs `page-objects/` vs `src/pages/`

5. **Output frameworkMap** (used by test-generator-agent):
   ```json
   {
     "testDir": "./tests",
     "pomDir": "./pages",
     "baseURL": "https://existing-app.com",
     "fileNamingStyle": "PascalCase",
     "locatorStyle": "getByTestId",
     "existingPages": ["LoginPage", "DashboardPage"],
     "baseClass": null,
     "importStyle": "alias",
     "aliasPrefix": "@pages"
   }
   ```

6. **Report to user**: Show the detected structure summary and ask for confirmation before proceeding
