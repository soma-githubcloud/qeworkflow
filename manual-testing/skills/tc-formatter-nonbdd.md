# Skill: TC Formatter — Non-BDD

Apply this skill to transform raw Gherkin scenarios (from specialist agents and gap-filler) into
structured **non-BDD test case documents** — the traditional TC-ID / Steps / Expected Result
format used in test management tools (JIRA, TestRail, Azure Test Plans, etc.).

These output files serve dual purpose:
1. **Manual test case documentation** — imported into test management tools or reviewed by QA
2. **Automation input** — fed to `intake-agent` → `testgen-agent` when `handoffToAutomation = true`

## Input

```
scenarioSources : array of file paths
  - outputs/<projectName>/manual-tests/01-scenarios/*.feature   (Phase 1 output)
  - outputs/<projectName>/manual-tests/03-gap-filled/*.feature  (Phase 3 output, if exists)
projectName     : string
featureSlug     : string   — kebab-case feature name
```

## Output

One or more `.md` files in `outputs/<projectName>/manual-tests/manual-tcs/nonbdd/`.

## Gherkin → TC Conversion Rules

### Mapping

| Gherkin element | TC document field |
|-----------------|------------------|
| Feature name | Test Suite / Feature Name |
| Scenario name | TC Title |
| Given steps | Preconditions |
| When + And steps | Test Steps (numbered list) |
| Then + And steps | Expected Results (numbered list, matching step count) |
| Feature description | Test Suite Description |

### Conversion Algorithm

For each `Scenario:` block:
1. Extract all `Given` / `And` (following Given) → Preconditions list
2. Extract all `When` / `And` (following When) → Steps list (numbered 1, 2, 3...)
3. Extract all `Then` / `And` (following Then) → Expected Results (numbered to match steps)
4. If `Then` count < `When` count: last expected result applies to all remaining steps
5. Scenario name → TC Title (strip "TC-NNN —" prefix if present; keep description)

### TC ID Assignment

Assign sequential IDs: TC-001, TC-002, TC-003 ...
Group IDs by test type:
- TC-001 to TC-019: Positive
- TC-020 to TC-039: Negative
- TC-040 to TC-059: Edge Cases
- TC-060 to TC-079: Integration
- TC-080 to TC-099: Security
- TC-100 to TC-119: Performance
- TC-120 to TC-139: API
- TC-140 to TC-159: UI/UX

### Priority Assignment

| Test Type | Default Priority |
|-----------|-----------------|
| Positive | High |
| Negative | High |
| Security | Critical |
| API | High |
| Edge Cases | Medium |
| Integration | Medium |
| Performance | Medium |
| UI/UX | High |

## Output File Format

**File**: `outputs/<projectName>/manual-tests/manual-tcs/nonbdd/<featureSlug>.md`

```markdown
# Test Cases: [Feature Name]

**Project**: [projectName]
**Feature**: [featureName]
**Generated**: [timestamp]
**Total TCs**: [N]

---

## Positive Test Cases

---

### TC-001: [Scenario Title]

| Field | Value |
|-------|-------|
| **TC ID** | TC-001 |
| **Title** | [Scenario title — what is being tested] |
| **Type** | Positive |
| **Priority** | High |
| **Module** | [module name inferred from feature] |

**Description**: [Feature description or scenario context — 1 sentence]

**Preconditions**:
1. [First Given step converted to plain English]
2. [Second Given step converted to plain English]

**Test Steps**:

| Step | Action |
|------|--------|
| 1 | [First When step in plain English] |
| 2 | [Second When step in plain English] |
| 3 | [Third When step in plain English] |

**Expected Results**:

| Step | Expected Result |
|------|----------------|
| 1 | [Corresponding Then/And step] |
| 2 | [Next expected result] |
| 3 | [Final expected result] |

**Test Data**:
- Email: john.doe@example.com
- Password: SecurePass123!
- [Other specific test data used in the steps]

---

### TC-002: [Scenario Title]
[same format]

---

## Negative Test Cases

---

### TC-020: [Scenario Title]

| Field | Value |
|-------|-------|
| **TC ID** | TC-020 |
| **Title** | [What fails and why] |
| **Type** | Negative |
| **Priority** | High |
| **Module** | [module] |

**Preconditions**: [preconditions]

**Test Steps**:
| Step | Action |
|------|--------|
| 1 | [action] |

**Expected Results**:
| Step | Expected Result |
|------|----------------|
| 1 | [error condition] |

---

[Continue for Edge Cases, Integration, Security, Performance, API, UI/UX sections]
```

### TC Index File

Also save: `outputs/<projectName>/manual-tests/manual-tcs/nonbdd/TC-INDEX.md`

```markdown
# Test Case Index — Non-BDD

| TC ID | Title | Type | Priority | Module |
|-------|-------|------|----------|--------|
| TC-001 | Successful registration with valid credentials | Positive | High | registration |
| TC-002 | ... | | | |
```

## Plain English Conversion

When converting Gherkin steps to plain English for the TC document:
- Remove Given/When/Then/And keywords
- Capitalize first word
- Rephrase passive Gherkin steps into active instructions:
  - "the user enters 'john@example.com' in the email field" → "Enter 'john@example.com' in the email field"
  - "the API returns HTTP 201 Created" → "Verify the API returns HTTP 201 Created"
  - "an error message is displayed" → "Verify that an error message is displayed"

For Preconditions (Given → state, not action):
- Keep as state descriptions, not instructions:
  - "the user is on the registration page" → "User is on the registration page"
  - "a user account exists for john.doe@example.com" → "A user account exists for john.doe@example.com"

## Example Output (single TC)

```markdown
### TC-001: Successful registration with valid credentials

| Field | Value |
|-------|-------|
| **TC ID** | TC-001 |
| **Title** | Successful registration with valid credentials |
| **Type** | Positive |
| **Priority** | High |
| **Module** | registration |

**Description**: Verifies that a new user can successfully register with valid email and password.

**Preconditions**:
1. User is on the registration page
2. No existing account exists for "john.doe@example.com"

**Test Steps**:

| Step | Action |
|------|--------|
| 1 | Enter "john.doe@example.com" in the email address field |
| 2 | Enter "SecurePass123!" in the password field |
| 3 | Enter "John Doe" in the full name field |
| 4 | Agree to the terms and conditions |
| 5 | Submit the registration form |

**Expected Results**:

| Step | Expected Result |
|------|----------------|
| 1–4 | Fields accept input without error |
| 5 | Account is created and user is redirected to dashboard |

**Test Data**:
- Email: john.doe@example.com
- Password: SecurePass123!
- Full Name: John Doe
```
