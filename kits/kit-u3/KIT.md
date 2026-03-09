# Kit-U3: UI Greenfield — Selenium Java + TestNG/JUnit5

## Key Differences from Kit-U1

| Property | Kit-U1 | Kit-U3 |
|---|---|---|
| Language | TypeScript | Java |
| Test runner | Playwright | Selenium + TestNG/JUnit5 |
| Tier-1 locator | Playwright MCP | CDP / Selenium IDE |
| BDD runner | Cucumber TS | Cucumber-Java |
| Build tool | npm | Maven |
| Reporting | Playwright HTML + Allure | Allure + ExtentReports |

## Tier-1 Locator Strategy: CDP/Selenium IDE

When an App URL is available, use one of:
- **Option A** (preferred): Launch Selenium IDE, record user navigation to each page, export locators
- **Option B**: Use Chrome DevTools Protocol (CDP) via Selenium's DevTools interface to inspect live DOM

Both options produce `By.*` locators — apply the `selenium-java.md` priority rules.

## Java Naming Conventions

- Classes: `PascalCase` (e.g., `LoginPage`, `LoginTest`)
- Methods: `camelCase` (e.g., `clickSubmit()`, `fillEmail()`)
- Packages: `com.<projectName>.pages`, `com.<projectName>.tests`
- Test annotations: `@Test(description = "TC-001: ...")`, `@BeforeClass`, `@AfterClass`

## Maven Project Structure

```
src/
  test/
    java/
      com/<projectName>/
        pages/          ← Page Object classes
        tests/          ← TestNG/JUnit5 test classes
        steps/          ← Cucumber step definitions (BDD)
        config/         ← WebDriver factory, ExtentReport config
    resources/
      features/         ← .feature files (BDD)
      allure.properties
      testng.xml        ← TestNG suite runner
```

## Test Switch: TestNG vs JUnit5

Controlled by `tech.testRunner` in kit.config.json:
- `"testng"` → use `@Test`, `Assert.*`, `testng.xml`
- `"junit5"` → use `@Test`, `@BeforeEach`, `Assertions.*`, `@Suite`
