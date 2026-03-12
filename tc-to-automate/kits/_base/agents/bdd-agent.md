# BDD Agent

**Single Responsibility**: Generate Gherkin `.feature` files and TypeScript step definition files
from parsed test cases. Does NOT generate POMs or non-BDD specs.

---

## Inputs (passed by calling command)

| Parameter | Type | Description |
|---|---|---|
| `testCases` | object[] | Parsed test case objects from `intake.summary.json` |
| `pomDir` | string | Path to generated POM files (for step body generation) |
| `kitConfig` | object | Full kit.config.json |
| `style` | string | `"strict"` or `"relaxed"` |
| `extraTags` | string[] | Additional tags to apply to all scenarios |
| `dryRun` | boolean | Preview without writing |

---

## Step 1: Transform test cases to Gherkin

Use `tc-to-automate/kits/_base/skills/gherkin-transformer.md` to convert each test case's steps into Given/When/Then sequences.

Group test cases by their first `pages[0]` value → one `.feature` file per page/feature group.

**Tagging rules** (apply to every scenario):
- Always add: `@TC-<testCaseId>` (e.g., `@TC-001`)
- From test case `tags` array: `@smoke` → add `@smoke`; `@regression` → add `@regression`
- From `extraTags` parameter
- Always add `@ui` for all Kit-U1/U2 scenarios

**Style enforcement**:
- `strict`: apply domain language filter — map UI verbs to business verbs:
  - "fill email field" → "provide email address"
  - "click login button" → "submit the login form"
  - "navigate to login page" → "the user is on the login page"
- `relaxed`: use step text as-is from transformed output

---

## Step 2: Deduplicate steps

Before generating step definitions, scan `<outputDir>/<projectName>/tests/bdd/` for ALL existing step files.
Extract all step patterns (the regex/string after `Given(`, `When(`, `Then(`).

For each new step: check if a matching pattern already exists. If yes → REUSE it (import from the existing file, do not recreate). Only generate code for genuinely NEW steps.

---

## Step 3: Generate `.feature` files

For each feature group (page):

1. Load template: `<kitConfig.templates>/spec-bdd.feature.tmpl`
2. Build context:
   ```json
   {
     "kitId": "kit-u1",
     "featureName": "Login",
     "testCaseIds": ["TC-001", "TC-002"],
     "scenarios": [
       {
         "id": "TC-001",
         "title": "Login with valid credentials",
         "tags": ["ui", "smoke", "TC-001"],
         "given": ["the user is on the login page"],
         "when": ["they provide their email address", "they provide their password", "they submit the login form"],
         "then": ["they should be redirected to the dashboard"]
       }
     ]
   }
   ```
3. Render → write to `<outputDir>/<projectName>/tests/bdd/<featureName>.feature`
4. Apply `tc-to-automate/rules/bdd-gherkin.md` rules — validate one assertion per Then, no UI verbs in strict mode

---

## Step 4: Generate step definition files

1. Load template: `<kitConfig.templates>/step-definitions.ts.tmpl`
2. Build context using NEW steps only (deduplicated in Step 2)
3. For each step, map to the correct POM method (using the same action→method mapping logic as testgen-agent)
4. Render → write to `<outputDir>/<projectName>/tests/bdd/<featureName>.steps.ts`
5. If reusing existing steps from other files: add import statement pointing to that file

---

## Validation

1. `npx cucumber-js --dry-run` — all steps from all .feature files must have a matching definition
2. `npx tsc --noEmit` — step definition files must compile
3. Gherkin lint: confirm each scenario starts with `Given`, all `Then` clauses have exactly one assertion verb

---

## Dry-run output

```
BDD Dry-Run Preview
────────────────────────────────────────────────────────
Feature: Login (1 scenario)
  @ui @smoke @TC-001
  Scenario: TC-001 - Login with valid credentials
    Given the user is on the login page
    When  they provide their email address
    And   they provide their password
    And   they submit the login form
    Then  they should be redirected to the dashboard

Steps:
  NEW (4): would create tests/bdd/login.steps.ts
  REUSED (1): "they are redirected" from tests/bdd/common.steps.ts

Files that would be written:
  tests/bdd/login.feature
  tests/bdd/login.steps.ts
```
