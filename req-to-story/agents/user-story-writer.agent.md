# user-story-writer — User Story Generation Agent

**Role**: Step 4 of the `/gen-user-stories` pipeline. Reads `feature-map.json` and generates
one properly formatted user story file per feature, applying quality rules and preserving
traceability back to the source requirement document.

---

## Input Parameters

```
featureMapPath         : string  — path to feature-map.json from feature-splitter
reqAnalysisPath        : string  — path to req-analysis.json (for NFRs + domain)
projectName            : string
outputBase             : string  — outputs/<projectName>/user-stories/
outputMode             : string  — "auto" | "single" | "separate" (default: "auto")
dryRun                 : boolean
```

---

## User Story File Format

Each generated file follows this exact structure:

```markdown
# US-<NNN>: <Feature Title>

**Source**: `<source filename>` | **Feature ID**: F-<NNN> | **Section**: <sourceSection>
**Generated**: <ISO date> | **Status**: Draft

---

## User Story

As a **<actor/role>**
I want to **<goal — what the user wants to accomplish>**
So that **<benefit — the value or outcome they receive>**

---

## Acceptance Criteria

- **AC-01**: <Specific, testable condition — present tense, active voice>
- **AC-02**: <Another condition>
- **AC-03**: <Another condition>
<!-- minimum 3 ACs; maximum 10 per story -->

---

## Technical Context

| Aspect | Details |
|--------|---------|
| Has UI | Yes / No |
| UI Components | <list if applicable> |
| Has API | Yes / No |
| API Endpoints | <list if applicable> |
| External Systems | <list if applicable> |
| Database | <entities affected, if known> |
| Performance SLAs | <from NFRs, if applicable> |
| Security / Compliance | <from NFRs, if applicable> |
| Accessibility | <from NFRs, if applicable> |

---

## Notes

> **Open Questions** (resolve before development):
> - <question 1 from feature-map openQuestions>
> - <question 2>

> **Constraints**:
> - <constraints from source document>

> **Out of Scope** (for this story):
> - <items explicitly excluded>
```

---

## Execution Steps

### Step 1: Read Inputs

Read `feature-map.json` (features array) and `req-analysis.json` (NFRs, domain, actors).

### Step 2: Process Each Feature

For each feature in `features[]`, generate one user story:

#### 2a. Assign US ID
Sequential: `US-001`, `US-002`, ... (padded to 3 digits minimum)

#### 2b. Write "As a / I want / So that" Statement

Use `feature.actor`, `feature.goal`, `feature.benefit` from feature-map.

If `goal` or `benefit` is missing from feature-map, infer from `feature.description`:
- `goal` = what the actor does with this feature (verb + object)
- `benefit` = why it matters to them (outcome, not implementation)

Rules:
- `As a` → use the specific role, never "user" if a more specific role is available
- `I want to` → action verb + object, user perspective (not system perspective)
- `So that` → business/personal value, not technical outcome

**Anti-patterns to avoid:**
| Bad | Good |
|-----|------|
| "As a user" when role is known | "As a **Patient**" |
| "I want the system to store data" | "I want to save my preferences" |
| "So that the database is updated" | "So that my settings persist across sessions" |

#### 2c. Write Acceptance Criteria

Start from `feature.acceptanceCriteriaDraft`. For each draft item:
- Rewrite as a specific, testable condition in present tense
- Each AC = one verifiable assertion (not compound "and" statements)
- Add context: "When X, then Y" or "Given Z, <system> shows/returns/prevents..."

Expand draft ACs to cover:
1. Happy path (primary success scenario)
2. At least one validation rule
3. At least one error/edge condition (if inferable from source)

Format: `**AC-NN**: <condition>`

Minimum 3 ACs per story. Maximum 10.

#### 2d. Populate Technical Context

From `req-analysis.json#nonFunctionalRequirements` + `feature.description`:

- `Has UI`: yes if feature involves screens, forms, or user interactions
- `Has API`: yes if feature mentions endpoints, requests, or service calls
- `External Systems`: any third-party services, databases, or integrations mentioned
- `Performance SLAs`: pull relevant NFRs (e.g., "response < 2s")
- `Security / Compliance`: pull relevant NFRs (e.g., "HIPAA", "requires authentication")
- Mark fields as `N/A` if not applicable, never leave blank

#### 2e. Write Notes Section

From `feature.openQuestions` and `feature.description`:
- Open Questions: items that need stakeholder clarification before development begins
- Constraints: technical, regulatory, or business constraints mentioned in source
- Out of Scope: any explicit exclusions mentioned in source for this feature

If no open questions: omit that subsection entirely.

### Step 3: Apply Quality Rules

Read and enforce `req-to-story/rules/user-story-quality.md` before saving each file.

Reject and rewrite if:
- `So that` describes a technical outcome, not user value
- Any AC is vague ("works correctly", "is displayed", "functions as expected")
- ACs contain compound assertions ("and")
- Story has < 3 ACs

### Step 4: Resolve Output Mode

Read `outputMode` parameter (`auto` | `single` | `separate`). Apply the following logic:

```
auto:
  if sourceFiles.count == 1  → mode = "grouped-single"   (one output file for the whole source)
  if sourceFiles.count > 1   → mode = "grouped-by-source" (one output file per source file)

single:
  → mode = "combined"   (always one file: US-all-<projectName>.md)

separate:
  → mode = "separate"   (always one file per story: US-NNN-<slug>.md)
```

### Step 5: Save Files

#### Mode: `grouped-single` or `grouped-by-source`

Write stories as H2 sections inside a grouped file. Each group file covers one source document.

**File name**: `US-<source-slug>.md`
- `source-slug` = source filename (no extension) converted to kebab-case, max 50 chars
- Example: `prd-v2.pdf` → `US-prd-v2.md`; `checkout-feature.md` → `US-checkout-feature.md`

**File structure**:
```markdown
# User Stories — <Source Document Name>

**Source**: `<source filename>` | **Document Type**: <type>
**Generated**: <date> | **Total Stories**: N | **Status**: Draft

---

## US-001: <Feature Title>

**Feature ID**: F-001 | **Section**: <sourceSection> | **Actor**: <actor>

### User Story
As a **<actor>** ...

### Acceptance Criteria
- **AC-01**: ...

### Technical Context
| Aspect | Details |
...

### Notes
...

---

## US-002: <Feature Title>
...
```

US IDs remain globally sequential (`US-001`, `US-002`, ...) even when grouped in one file.

#### Mode: `combined`

Write all stories from all sources into a single file: `US-all-<projectName>.md`

Structure is the same as grouped-single but has a cover section listing all source documents:
```markdown
# User Stories — <projectName>

**Sources**: `file1.pdf`, `file2.yaml`, ...
**Total Stories**: N | **Generated**: <date>
...
```

#### Mode: `separate`

Write each story to its own file: `US-<NNN>-<feature-slug>.md`

`feature-slug` = `feature.title` → kebab-case, max 40 chars.
Example: "Patient Login with MFA" → `US-001-patient-login-with-mfa.md`

File contains the single story only (no H2 wrapper — the H1 is the story title).

### Step 6: Write US-INDEX.md

Always written regardless of output mode. Links resolve correctly for all three modes.

```markdown
# User Stories Index

**Source**: <source file(s)>
**Document Type**: <type>
**Generated**: <date>
**Output Mode**: <auto/single/separate>
**Total Stories**: N

| ID | Title | Actor | ACs | File | Source Section |
|----|-------|-------|-----|------|----------------|
| US-001 | Patient Login with MFA | Patient | 5 | [US-prd-v2.md#us-001](US-prd-v2.md#us-001) | §3.2 |
| US-002 | Appointment Scheduling | Patient | 4 | [US-prd-v2.md#us-002](US-prd-v2.md#us-002) | §3.3 |

## Deferred Features

| Feature ID | Title | Reason |
|------------|-------|--------|
| F-013 | Audit Trail Export | Exceeded limit of 20 |
```

**Link format per mode**:
- `grouped-single` / `grouped-by-source`: `[US-<source-slug>.md#us-NNN](<file>#us-NNN)`
- `combined`: `[US-all-<project>.md#us-NNN](US-all-<project>.md#us-NNN)`
- `separate`: `[US-NNN-<slug>.md](US-NNN-<slug>.md)`

### Step 7: If --dry-run

Print the planned file list without writing:
```
[DRY RUN] Output mode: auto → grouped-single (1 source file)

Would generate:
  US-prd-v2.md                           (contains 6 stories)
    ├── US-001: Patient Login with MFA    [Patient | 5 ACs]
    ├── US-002: Appointment Scheduling    [Patient | 4 ACs]
    └── ...
  US-INDEX.md
```

### Step 7: Report

```
✅ User Stories Generated

Stories: N files written to outputs/<projectName>/user-stories/
Index:   outputs/<projectName>/user-stories/US-INDEX.md

📋 Summary:
  US-001: Patient Login with MFA           — 5 ACs | 2 open questions
  US-002: Appointment Scheduling           — 4 ACs
  ...

💡 Next Steps:
  Review stories with stakeholders and resolve open questions.
  Then generate manual test cases:
    /gen-manual-tests --story outputs/<projectName>/user-stories/US-001-<slug>.md
  Or run full pipeline on all stories:
    /gen-user-stories --req <same-file> --then-manual-tests
```

---

## Notes

- Each story file is self-contained and can be passed directly to `/gen-manual-tests` or `/e2e`
- The US-INDEX.md uses relative markdown links so it works as a navigation hub in any markdown viewer
- `openQuestions` from the source are preserved verbatim — do not try to answer them
- If a feature has very sparse detail in the source, mark ACs as `[TO BE CONFIRMED]` rather than inventing requirements
