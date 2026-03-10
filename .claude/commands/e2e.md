# /e2e — Full Pipeline: User Story → Manual TCs → Automation Artifacts

**End-to-end command.** Accepts a user story as input, generates manual test cases (BDD or
non-BDD), then optionally feeds them into the automation generation pipeline.

## Arguments

| Argument | Description |
|----------|-------------|
| `--story <path\|inline>` | **Required.** Path to a user story file, or inline user story text |
| `--project <name>` | Override project name from `kit.config.json` |
| `--style bdd\|nonbdd\|both` | Manual TC format AND automation test style (default: from `kit.config.json`) |
| `--url <url>` | Live app URL for Tier-1 locator extraction during automation phase |
| `--dom <path>` | DOM snapshot directory/file for Tier-2 locator extraction |
| `--types <list>` | Comma-separated specialist types to run: `positive,negative,edge,security,api,ui,integration,performance` (default: auto from analyzer) |
| `--mode 1\|2\|3\|4` | Orchestrator mode (default: 1 = full 5-phase workflow with evaluation + gap-filling) |
| `--skip-automation` | Stop after manual TC generation — do not run automation pipeline |
| `--dry-run` | Preview planned files without writing any |

### `--story` Resolution

Supported input types (all resolved before parsing):

| Input | How handled |
|-------|-------------|
| Inline text (contains "As a", "I want", "Feature:", or "Acceptance Criteria") | Used as-is |
| `.md` / `.story.md` / `.txt` | Read file as plain text |
| `.pdf` | Extract text content page-by-page; strip headers/footers |
| `.yml` / `.yaml` | Parse as YAML; look for keys `story`, `userStory`, `feature`, `acceptanceCriteria`; serialize to text block before parsing |
| File path that does not exist | Error: `"User story file not found at [path]"` |
| Unrecognized extension | Attempt to read as plain text; warn if content looks binary |

```
Resolution order:
1. If value is a file path → determine extension → apply handler above
2. Else treat as inline text
```

### Examples

```bash
# Full pipeline: user story → manual TCs → automation (requires --url or --dom for locators)
/e2e --story user-stories/checkout.md --url https://app.example.com

# User story → manual TCs only (no automation)
/e2e --story user-stories/checkout.md --skip-automation

# Specific test types + both TC formats
/e2e --story user-stories/login.md --style both --types positive,negative,security

# Inline user story
/e2e --story "As a shopper I want to add items to cart so that I can purchase them"

# Quick mode (no evaluation/gap-filling) — just generate and format
/e2e --story user-stories/checkout.md --mode 2 --skip-automation
```

---

## Execution Pipeline

### Step 0: Load Kit Context

- Read `kit.config.json` — get `id`, `tech`, `testStyle`, `outputDir`, `projectName`
- Read `kits/<kit-id>/KIT.md` for naming conventions
- Confirm to user: `Active kit: <id> | <tech.testRunner> | <testStyle>`
- Resolve `projectName`: `--project` argument overrides `kit.config.json`
- Resolve `testStyle`: `--style` argument overrides `kit.config.json`
- `outputBase` = `<outputDir>/<projectName>/manual-tests/`

### Step 1: Parse User Story

Apply `manual-testing/skills/user-story-parser.md`:
- Read `--story` input (file or inline)
- Extract: `featureName`, `featureSlug`, `acceptanceCriteria`, `technicalContext`
- Report: "User story loaded: [featureName] | [AC count] acceptance criteria found"
- If `--dry-run`: show parsed fields and planned files, then stop

### Step 2: Manual Test Case Generation (Phase 1–3)

Read and execute `manual-testing/agents/orchestration/quality-master-orchestrator.agent.md`.

Pass:
```
userStoryText   : full text from --story
projectName     : resolved project name
outputBase      : <outputDir>/<projectName>/manual-tests/
testStyle       : resolved testStyle
testTypes       : --types value (or "auto")
mode            : --mode value (default: 1)
handoffToAutomation : false (orchestrator generates TCs only; automation is Step 4 here)
dryRun          : --dry-run value
```

The master orchestrator runs:
- Phase 0: `user-story-analyzer` → `00-analysis/`
- Phase 1: `quality-orchestrator` → `01-scenarios/`
- Phase 2 (mode 1 only): `quality-evaluator` → `02-evaluation/`
- Phase 3 (mode 1, if gaps exist): `gap-filler` → `03-gap-filled/`

### Step 3: Format Manual Test Cases (Phase 4)

After master orchestrator completes, apply TC formatter skills:

**If `testStyle` includes `"bdd"`**:
- Apply `manual-testing/skills/tc-formatter-bdd.md`
- Input: `01-scenarios/` + `03-gap-filled/` (if exists)
- Output: `manual-tcs/bdd/<featureSlug>.feature` + `manual-tcs/bdd/TC-INDEX.md`

**If `testStyle` includes `"nonbdd"`**:
- Apply `manual-testing/skills/tc-formatter-nonbdd.md`
- Input: same
- Output: `manual-tcs/nonbdd/<featureSlug>.md` + `manual-tcs/nonbdd/TC-INDEX.md`

Report:
```
✅ Manual Test Cases Generated

Project: [projectName]
Feature: [featureName]
Location: outputs/[projectName]/manual-tests/manual-tcs/

BDD (.feature): [N] scenarios in [N] files
Non-BDD (.md): [N] test cases in [N] files
```

### Step 4: Automation Pipeline (unless `--skip-automation`)

Feed formatted manual TCs as input to the automation pipeline.

**Confirm before proceeding** (unless `--dry-run`):
```
Manual TCs are ready. Proceeding to generate automation artifacts.
Locator strategy: [playwright-mcp | dom-based | ai-guided]
```

Run the following agents in order (same as `/generate-automation-tests`):

**Step 4a: intake-agent**
Read and execute `kits/_base/agents/intake-agent.md`. Pass:
- `appUrl`: `--url` value (or null)
- `domFiles`: resolved from `--dom` (or empty)
- `testCasesRaw`: path to formatted manual TCs
  - BDD testStyle → path to `manual-tcs/bdd/<featureSlug>.feature`
  - nonbdd testStyle → path to `manual-tcs/nonbdd/<featureSlug>.md`
  - both → both paths
- `kitConfig`: full `kit.config.json` contents

Output: `<outputDir>/<projectName>/intake.summary.json`

**Step 4b: locator-agent** (if `selectors.json` not already present or user chooses to regenerate)
Read and execute `kits/_base/agents/locator-agent.md` with strategy from `intake.summary.json`.

**Step 4c: pom-agent**
Read and execute `kits/_base/agents/pom-agent.md` → generates POM files in `src/pages/`.

**Step 4d: testgen-agent and/or bdd-agent**
Based on `testStyle` from `intake.summary.json`:
- `"nonbdd"` → run `testgen-agent` only
- `"bdd"` → run `bdd-agent` only
- `"both"` → run both agents

**Step 4e: reporting-agent**
Configure Allure + HTML reporters (skip if already configured).

### Step 5: Validate (if automation ran)

- Run `npx tsc --noEmit` inside generated project
- Run `npx playwright test --list` to confirm test discovery
- Fix unambiguous import path issues automatically

### Step 6: Summary

Print a consolidated summary table:

```
📋 Manual Test Cases
   Location: outputs/[projectName]/manual-tests/manual-tcs/
   BDD: [N] scenarios in [N] .feature files
   Non-BDD: [N] TCs in [N] .md files
   Quality Score: [N]/100 (or "N/A — evaluation skipped")

🤖 Automation Artifacts (if applicable)
   POMs: [N] files in src/pages/
   Specs: [N] .spec.ts files in tests/nonbdd/
   Features: [N] .feature files in tests/bdd/
   Steps: [N] .steps.ts files in tests/bdd/

📁 All outputs: outputs/[projectName]/
```

---

## Important Rules

- Always run `user-story-parser` first — it normalizes the story before any agent sees it
- Always pass the **complete** user story text to specialist agents — never a summary
- Manual TCs are generated BEFORE automation artifacts — they are the input contract
- `--skip-automation` is respected strictly — do not run any kits/_base/agents/* steps
- If `--url` and `--dom` are both absent and `--skip-automation` is NOT set, warn the user:
  `"No URL or DOM provided. Automation locators will use AI-guided strategy (Tier 3 — scores capped at 0.75). Add --url or --dom for higher-confidence selectors."`
- The canonical selector file is `selectors.json` — never `locators.json`
- If `--dry-run`: print planned file list for ALL steps (manual + automation), write nothing
