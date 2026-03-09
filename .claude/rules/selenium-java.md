# Selenium Java Test Rules

Apply these rules when editing or generating `*Test.java`, `*Page.java`, or `*Steps.java` files (Kit-U3, U4, H2).

## Project Structure
- Package: `com.<projectName>.tests`, `com.<projectName>.pages`, `com.<projectName>.steps`
- Test classes: `<Feature>Test.java` (TestNG) or `<Feature>Tests.java` (JUnit5)
- Page Objects: `<PageName>Page.java`
- Step definitions: `<Feature>Steps.java`

## WebDriver Management
- Use `WebDriverManager` (io.github.bonigarcia:webdrivermanager) — never manually manage drivers
- `WebDriver` instance is created fresh per test class (not per test method)
- Always call `driver.quit()` in `@AfterClass` / `@AfterAll`
- Use `ThreadLocal<WebDriver>` when parallelism is enabled

## Locator Strategy Priority (highest to lowest)
1. `By.id("element-id")`
2. `By.name("element-name")`
3. `By.cssSelector("[data-testid='value']")`
4. `By.cssSelector("[aria-label='value']")`
5. `By.cssSelector("tag#id")` or `By.cssSelector("tag.class")`
6. `By.xpath(...)` — LAST RESORT only; use relative XPath, never absolute

## Explicit Waits (Mandatory)
- NEVER use `Thread.sleep()` — use `WebDriverWait` with `ExpectedConditions`
- Standard wait: `new WebDriverWait(driver, Duration.ofSeconds(10))`
- Common conditions: `visibilityOfElementLocated`, `elementToBeClickable`, `presenceOfElementLocated`
- Do NOT use implicit waits alongside explicit waits

## TestNG Annotations
- `@BeforeClass` → driver setup
- `@AfterClass` → `driver.quit()`
- `@Test(description = "TC-001: ...")` → include test case ID in description
- Use `@DataProvider` for parameterized tests instead of copy-paste test methods

## Assertions
- Use TestNG `Assert.*` (not JUnit's)
- `Assert.assertEquals(actual, expected, "failure message")`
- `Assert.assertTrue(condition, "failure message")`
- Every assertion must include a meaningful failure message

## Page Object Java Rules
- `private final WebDriver driver;` + `private final WebDriverWait wait;`
- Constructor: `public LoginPage(WebDriver driver) { this.driver = driver; this.wait = new WebDriverWait(driver, Duration.ofSeconds(10)); }`
- Use `@FindBy` annotations with `PageFactory.initElements()` for locator declarations
- Action methods: `public void clickLogin()`, `public void fillEmail(String email)`
