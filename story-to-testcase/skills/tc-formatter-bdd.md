# Skill: TC Formatter — BDD

Apply this skill to transform raw Gherkin scenarios (from specialist agents and gap-filler) into
properly tagged, kit-compliant `.feature` files for use as **manual test cases**.

These output files serve dual purpose:
1. **Manual test case documentation** — reviewed and executed by QA engineers
2. **Automation input** — fed to `bdd-agent` when `handoffToAutomation = true`

## Input

```
scenarioSources : array of file paths
  - outputs/<projectName>/manual-tests/01-scenarios/*.feature   (Phase 1 output)
  - outputs/<projectName>/manual-tests/03-gap-filled/*.feature  (Phase 3 output, if exists)
projectName     : string
featureSlug     : string   — kebab-case feature name (from user-story-parser)
testStyle       : string   — must include "bdd" to trigger this skill
kitRules        : string   — path to tc-to-automate/rules/bdd-gherkin.md (applied automatically)
```

## Output

One or more `.feature` files in `outputs/<projectName>/manual-tests/manual-tcs/bdd/`.

Files are grouped by functional area (test type or page/feature group), not one file per raw
source file. Prefer fewer, logically grouped files over many small files.

## Transformation Rules

### Step 1: Merge and Deduplicate Scenarios

1. Read all input `.feature` files (Phase 1 + Phase 3 gap-filled)
2. Group scenarios by test type: positive, negative, edge-cases, integration, security,
   performance, api, ui-ux
3. Deduplicate: if a gap-filled scenario is functionally identical to an existing Phase 1
   scenario, keep the gap-filled version (it has better reasoning)

### Step 2: Apply Kit Tagging Rules

Per `bdd-gherkin.md` rules:

**Feature-level tag** (place on the `Feature:` line):
- If scenarios cover UI → add `@ui`
- If scenarios cover API only → add `@api`
- If mixed → add `@ui` (UI takes precedence)

**Scenario-level tags** (place above each `Scenario:` or `Scenario Outline:`):

| Tag | When to apply |
|-----|--------------|
| `@TC-<NNN>` | Every scenario — MANDATORY. Auto-generate sequential IDs: TC-001, TC-002, ... |
| `@smoke` | Positive scenarios + primary API contract scenarios |
| `@regression` | All other scenarios (negative, edge, security, performance, integration) |
| `@module(<name>)` | Group by functional area: `@module(registration)`, `@module(login)`, etc. |
| `@negative` | All negative-scenarios and security attack scenarios |
| `@boundary` | All edge-cases scenarios |
| `@wip` | Do NOT add unless explicitly requested |

**Mutual exclusion**: `@smoke` and `@regression` cannot both appear on the same scenario.
If a scenario qualifies for both, use `@smoke`.

### Step 3: Apply Domain Language (Strict Mode)

Per `bdd-gherkin.md` strict mode rules, replace UI implementation language with domain language:

| Replace | With |
|---------|------|
| "click the login button" | "submit the login form" |
| "fill in the email field" | "provide their email address" |
| "navigate to /login" | "the user is on the login page" |
| "press Enter" | "confirm the action" |
| "click the checkbox" | "agree to the terms" |

Only apply where the replacement is semantically equivalent. In API scenarios, keep technical
step wording (HTTP methods, status codes, endpoint paths are acceptable).

**Relaxed mode** (for API and security scenarios): Keep specific HTTP verbs, status codes, and
endpoint paths — these are domain language for API testing.

### Step 4: Normalize Structure

Each scenario must:
- Start with `Given` (never `When`, `And`, or `But` as first step)
- Have at least one `When` step
- Have at least one `Then` step
- Use `And` / `But` for continuations (not repeated `Given`/`When`/`Then`)
- One assertion per `Then` step — split multi-assertion `Then` steps

### Step 5: Assign TC IDs

Generate sequential TC IDs starting from TC-001 across all scenarios in the output files.
Format: `TC-<3-digit-zero-padded-number>` (TC-001, TC-002, ... TC-099, TC-100, TC-101).

Include the TC ID in the scenario title:
```gherkin
@TC-001 @smoke @module(registration)
Scenario: TC-001 — Successful registration with valid credentials
```

### Step 6: Output File Structure

```gherkin
@ui
Feature: [Feature Name]
  [Business description — 1-2 sentences explaining what this feature does]

  # ==========================================
  # POSITIVE SCENARIOS
  # ==========================================

  @TC-001 @smoke @module(registration)
  Scenario: TC-001 — [Descriptive title — what succeeds]
    Given [precondition in domain language]
    When [user action in domain language]
    Then [expected outcome]

  @TC-002 @smoke @module(registration)
  Scenario: TC-002 — [Another positive scenario]
    ...

  # ==========================================
  # NEGATIVE SCENARIOS
  # ==========================================

  @TC-010 @regression @negative @module(registration)
  Scenario: TC-010 — [What fails and why]
    ...

  # ... continue for edge cases, integration, security, performance, API, UI ...
```

### Step 7: Save Output Files

Save to `outputs/<projectName>/manual-tests/manual-tcs/bdd/`:

Naming convention: `<featureSlug>.feature` for a single feature, or
`<featureSlug>-<testType>.feature` if splitting large feature files by type (> 20 scenarios).

**Also save a TC index**: `outputs/<projectName>/manual-tests/manual-tcs/bdd/TC-INDEX.md`

```markdown
# Test Case Index — BDD

| TC ID | Title | Type | Tags | File |
|-------|-------|------|------|------|
| TC-001 | Successful registration | Positive | @smoke @module(registration) | registration.feature:L18 |
| TC-002 | ... | | | |
```

## Validation Checklist

Before saving, verify:
- [ ] Every scenario has `@TC-<NNN>` tag
- [ ] No scenario has both `@smoke` and `@regression`
- [ ] No scenario starts with `And` or `But`
- [ ] `@ui` or `@api` is present on every `Feature:` declaration
- [ ] Each `Then` step contains exactly one assertion
- [ ] No raw CSS selectors, XPath, or element IDs in step wording
- [ ] No hardcoded URLs in step wording

## Example Output

```gherkin
@ui
Feature: User Registration
  Allows new users to create an account with email and password.
  Requires email verification and password complexity enforcement.

  # ==========================================
  # POSITIVE SCENARIOS
  # ==========================================

  @TC-001 @smoke @module(registration)
  Scenario: TC-001 — User successfully registers with valid credentials
    Given the user is on the registration page
    When the user provides a valid email address "john.doe@example.com"
    And the user provides a valid password "SecurePass123!"
    And the user provides their full name "John Doe"
    And the user agrees to the terms and conditions
    And the user submits the registration form
    Then the user account is created successfully
    And the user receives a confirmation email at "john.doe@example.com"
    And the user is redirected to the dashboard

  # ==========================================
  # NEGATIVE SCENARIOS
  # ==========================================

  @TC-010 @regression @negative @module(registration)
  Scenario: TC-010 — Registration fails when email format is invalid
    Given the user is on the registration page
    When the user provides "not-an-email" as their email address
    And the user submits the registration form
    Then the registration is rejected
    And an error message "Please enter a valid email address" is displayed
```
