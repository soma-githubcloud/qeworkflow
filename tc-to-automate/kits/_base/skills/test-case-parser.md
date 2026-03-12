# Test Case Parser — Reusable Skill

Parses free-form test case descriptions into a structured object array that agents can use
for code generation. Handles multiple input formats.

---

## Input Formats Recognized

### Format 1: Numbered list (most common)
```
1. Login with valid credentials
   - Go to login page
   - Enter email: user@test.com
   - Enter password: Test@123
   - Click Login
   - Verify: redirected to dashboard

2. Login with invalid password
   - Go to login page
   - Enter email: user@test.com
   - Enter password: wrongpassword
   - Click Login
   - Verify: error message "Invalid credentials" is shown
```

### Format 2: Gherkin-like (partial BDD)
```
Scenario: Login with valid credentials
  Given I am on the login page
  When I enter "user@test.com" in email
  And I enter "Test@123" in password
  And I click Login
  Then I should be on the dashboard
```

### Format 3: Plain paragraph
```
The user navigates to the login page, enters their email and password, clicks submit,
and should be redirected to the dashboard page.
```

### Format 4: Test case table (structured)
```
TC-ID | Title | Steps | Expected Result
TC-001 | Valid login | 1. Open login ... | Dashboard displayed
```

---

## Parsing Rules

1. **Test case ID**: Detect patterns `TC-001`, `TC001`, `Test 1`, `#1`, or sequential numbering. If no ID found, auto-assign `TC-<sequence>`.
2. **Title**: First line of each test case block (remove leading numbers/bullets).
3. **Preconditions**: Lines starting with "Given", "Precondition:", "Pre:", "Setup:" or before the first action step.
4. **Steps**: Action lines — typically contain verbs: go, open, click, enter, fill, select, navigate, submit, upload.
5. **Expected results**: Lines starting with "Then", "Verify:", "Assert:", "Expected:", "Should:", or containing "should be", "must be", "is shown", "is displayed".
6. **Tags**: Words like "smoke", "regression", "critical", "P1", "P2" in the title or adjacent lines.

---

## Output Schema

```json
[
  {
    "testCaseId": "TC-001",
    "title": "Login with valid credentials",
    "tags": ["smoke"],
    "preconditions": [
      "User is not logged in",
      "Application is accessible at baseURL"
    ],
    "steps": [
      { "order": 1, "action": "navigate", "target": "login page",     "value": null,           "pageContext": "LoginPage" },
      { "order": 2, "action": "fill",     "target": "email field",    "value": "user@test.com", "pageContext": "LoginPage" },
      { "order": 3, "action": "fill",     "target": "password field", "value": "Test@123",      "pageContext": "LoginPage" },
      { "order": 4, "action": "click",    "target": "login button",   "value": null,            "pageContext": "LoginPage", "transitionTo": "DashboardPage" }
    ],
    "expectedResults": [
      "User is redirected to the dashboard page",
      "Welcome message is displayed"
    ],
    "pages": ["LoginPage", "DashboardPage"],
    "pageTransitions": [
      { "atStep": 4, "from": "LoginPage", "to": "DashboardPage" }
    ],
    "inputFormat": "numbered-list"
  }
]
```

### New fields

| Field | Where | Description |
|---|---|---|
| `step.pageContext` | every step | The page the user is on when this step executes. Derived by page-transition tracking (see rules below). |
| `step.transitionTo` | navigate/click/submit steps only | Set when the step causes the browser to move to a different page. Value is the destination page name. |
| `testCase.pageTransitions` | test case root | Ordered list of all page transitions in the test. Used by pom-agent and testgen-agent to import the right POM classes and sequence calls correctly. |

---

## Page Inference Rules

### Static page name mapping
From the step `target` or surrounding text, infer the page name:
- "login page", "sign in", "login form" → `LoginPage`
- "dashboard", "home screen", "main screen" → `DashboardPage`
- "checkout", "payment" → `CheckoutPage`
- "product", "catalog", "listing", "search results" → `SearchResultsPage` or `ProductPage`
- "profile", "account settings" → `ProfilePage`
- "register", "sign up" → `RegisterPage`
- "home", "homepage", "landing page" → `HomePage`
- If a URL is present (e.g. `https://www.amazon.in`): derive from the domain/path:
  - root path `/` → `HomePage`
  - `/s?` or `/search` → `SearchResultsPage`
  - `/dp/` → `ProductDetailPage`
  - `/gp/cart` → `CartPage`
- If context is still ambiguous: use the test case title as the page name (PascalCase)

### Page transition detection
Track `currentPage` as you parse steps sequentially. Start value = page inferred from the first
step (usually a `navigate` action or precondition).

For each step:
1. Assign `step.pageContext = currentPage`
2. Detect if the step causes a page transition:
   - `action == "navigate"` with a new URL or page name → transition
   - `action == "click"` where target is "search icon", "submit button", "login button",
     "checkout button", "next", "continue", "confirm", "place order", or similar CTA → likely transition
   - Phrase "after clicking X, user lands on Y" → explicit transition
   - Expected result for this step or the next step mentions a different page → transition
3. If transition detected:
   - Infer destination page name from the URL/target/expected result
   - Set `step.transitionTo = destinationPage`
   - Update `currentPage = destinationPage` for all subsequent steps
   - Record `{ atStep: step.order, from: previousPage, to: destinationPage }` in `pageTransitions`

### Transition signal phrases (detect these in step text)
| Signal | Interpretation |
|---|---|
| "navigates to", "goes to", "opens", "visits" | navigate action + possible transition |
| "clicks search", "clicks submit", "clicks login" | click action, likely transition |
| "results page is displayed", "page should be displayed" | previous step caused a transition |
| "lands on", "is redirected to", "should see the X page" | confirm transition destination |
| "products related to X should appear" | current page = results/listing page |

---

## Action Classification

Map action verbs to automation actions:

| User Verb | Automation Action |
|---|---|
| go to, navigate to, open, visit | `navigate` |
| click, tap, press, hit | `click` |
| enter, type, fill, input | `fill` |
| select, choose, pick | `select` |
| check, enable, toggle | `check` |
| uncheck, disable | `uncheck` |
| upload, attach | `upload` |
| hover over, mouse over | `hover` |
| scroll to, scroll down | `scroll` |
| verify, assert, check that, confirm | `assert` |
| wait for, wait until | `waitFor` |

---

## Handling Ambiguity

- If a step contains BOTH an action and an assertion: split into two steps
- If a step's target cannot be mapped to a known UI element: mark with `"ambiguous": true`
- If test data (email, password) is embedded: extract to `step.value`; do not hardcode in generated tests — move to fixture/test data file
- If no expected results found: generate a TODO assertion placeholder in the spec
