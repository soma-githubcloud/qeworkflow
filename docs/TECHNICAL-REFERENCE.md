# SQE (Synthetic Quality Engineer) — Technical Reference
## Kit U1: UI Greenfield — Playwright TypeScript

**Version**: 2.0 | **Kit ID**: `kit-u1` | **Date**: 2026-03-13

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Three Capabilities](#2-three-capabilities)
3. [CLAUDE.md — The Orchestrator](#3-claudemd--the-orchestrator)
4. [Slash Commands](#4-slash-commands)
5. [Code Generation Rules](#5-code-generation-rules)
6. [settings.json — MCP Servers, Permissions & Hooks](#6-settingsjson--mcp-servers-permissions--hooks)
7. [Tools Used](#7-tools-used)
8. [Capability 1 — Req-to-Story Module](#8-capability-1--req-to-story-module)
9. [Capability 2 — Story-to-Testcase Module](#9-capability-2--story-to-testcase-module)
10. [Capability 3 — TC-to-Automate Agents](#10-capability-3--tc-to-automate-agents)
11. [Skills](#11-skills)
12. [Kit Configuration (kit.config.json)](#12-kit-configuration-kitconfigjson)
13. [Output Structure](#13-output-structure)
14. [Plugins & Integrations](#14-plugins--integrations)
15. [Guardrails](#15-guardrails)
16. [Locator Strategy Tiers](#16-locator-strategy-tiers)
17. [Generated Project Config](#17-generated-project-config)

---

## 1. Architecture Overview

The SQE Kit is an AI-driven quality engineering system built on top of **Claude Code**. It
supports three input paths across three self-contained capability modules:

```
INPUT TYPES
───────────────────────────────────────────────────────────────────────────────
 Type 0: Requirement Doc          Type 1: User Story        Type 2: Test Cases
 (.pdf/.docx/.xlsx/.md/           (.md/.txt/.pdf/.yml/      (inline text /
  .png/.jpg/etc.)                  inline text)              .md file / spec)
───────────────────────────────────────────────────────────────────────────────
         │                               │                          │
         ▼                               │                          │
┌─────────────────────┐                 │                          │
│ CAPABILITY 1        │                 │                          │
│ req-to-story/       │                 │                          │
│                     │                 │                          │
│ req-ingestor        │                 │                          │
│ req-analyzer        │                 │                          │
│ feature-splitter    │                 │                          │
│ user-story-writer   │                 │                          │
│                     │                 │                          │
│ → US-NNN.md files   │─────────────────┘                          │
└─────────────────────┘  (feeds into Type 1 pipeline)              │
                                        │                          │
                                        ▼                          │
                         ┌─────────────────────────┐               │
                         │ CAPABILITY 2             │               │
                         │ story-to-testcase/       │               │
                         │                          │               │
                         │ user-story-parser        │               │
                         │ quality-master-orch.     │               │
                         │ ├─ user-story-analyzer   │               │
                         │ ├─ quality-orchestrator  │               │
                         │ │   └─ 8 specialists     │               │
                         │ ├─ quality-evaluator     │               │
                         │ └─ gap-filler            │               │
                         │ tc-formatter skill       │               │
                         │                          │               │
                         │ → manual-tcs/            │               │
                         └─────────────────────────┘               │
                                        │ (when automation)        │
                                        └──────────────────────────┘
                                                     │
                         ┌───────────────────────────▼──────────────────────┐
                         │ CAPABILITY 3 — TC-to-Automate (kit-aware)         │
                         │                                                    │
                         │  intake-agent → locator-agent → pom-agent         │
                         │         ↓              ↓             ↓            │
                         │  testgen-agent   bdd-agent    reporting-agent      │
                         └───────────────────────────────────────────────────┘
```

All automation agents communicate through **`intake.summary.json`** — written by intake-agent,
consumed by all downstream agents. No automation agent skips intake.

---

## 2. Three Capabilities

The kit is organized into three independent capability modules. Capabilities 1 and 2 are
kit-independent. Capability 3 is kit-aware.

```
Capability 1 — req-to-story/         : Requirement Doc  → User Stories
Capability 2 — story-to-testcase/    : User Story       → Manual Test Cases
Capability 3 — tc-to-automate/       : Test Cases       → Automation Artifacts
```

| Module | Input | Output | Kit-aware? |
|--------|-------|--------|-----------|
| `req-to-story/` | BRD, PRD, Feature Spec, meeting notes, images, spreadsheets | `US-NNN.md` user story files | No |
| `story-to-testcase/` | User story (any format) | Manual TC documents (BDD `.feature` or non-BDD `.md`) | No |
| `tc-to-automate/` | Test cases or manual TC output | POMs, spec files, BDD artifacts, CI pipeline | Yes |

**Input directories** (place files here before running commands):

| Directory | Used by |
|-----------|---------|
| `inputs/req-docs/` | Capability 1 — `/gen-user-stories` |
| `inputs/user-stories/` | Capability 2 — `/gen-manual-tests`, `/e2e` |
| `inputs/snapshots/` | Capability 3 — DOM snapshots for Tier 2 locator extraction |
| `inputs/test-cases/` | Capability 3 — manually authored test case files |

---

## 3. CLAUDE.md — The Orchestrator

**Location**: `CLAUDE.md` (project root)

CLAUDE.md is the master instruction file loaded by Claude Code at the start of every session. It
defines all behaviour, routing, and quality rules for the kit.

### 3.1 Startup Protocol

Every session begins with:
1. Read `kit.config.json` — note `id`, `mode`, `tech`, `testStyle`, `locatorStrategies`, `outputDir`
2. Read `tc-to-automate/kits/<kit-id>/KIT.md` — kit-specific naming conventions and generation rules
3. Read `tc-to-automate/configs/integrations.yml` — check active plugins (jira, github, allureServer)
4. Confirm active kit: `Active kit: <id> | <tech.testRunner> | <testStyle>`

### 3.2 Input Detection Protocol

### Input Type 0 — Requirement Document

| Signal | Classification |
|--------|---------------|
| `--req` flag present | `req_doc = true` |
| File ending in `.docx`, `.brd`, `.prd` | `req_doc = true` |
| Path under `requirements/`, `specs/`, or `inputs/req-docs/` | `req_doc = true` |

### Input Type 1 — User Story

| Input Signal | Classification |
|---|---|
| Text containing `"As a <role>"`, `"I want"`, `"So that"` | `user_story = true` |
| File ending in `.story.md`, `.us.md`, or path under `inputs/user-stories/` | `user_story = true` |
| Text with `"Feature:"` heading + acceptance criteria section | `user_story = true` |

### Input Type 2 — Test Cases / Scenarios

| Input Signal | Classification |
|---|---|
| URL starting with `http://` or `https://` | `app_url = true` |
| HTML content with `<html`, `data-testid`, `aria-`, `id=` | `dom_snapshot = true` |
| Numbered list or plain-English steps | `test_cases = true` |
| JSON with `"info"` + `"paths"` keys | `openapi_spec = true` |
| JSON with `"item"` arrays + request objects | `postman_collection = true` |

### 3.3 Agent Dispatch Table

**Input Type 0 — Requirement Doc → User Stories → (optional) Manual TCs → (optional) Automation:**

| Slash Command | Agents / Skills Invoked |
|---|---|
| `/gen-user-stories` | req-ingestor → req-analyzer → feature-splitter → user-story-writer |
| `/gen-user-stories --then-manual-tests` | [same as above] → for each US: user-story-parser → quality-master-orchestrator → tc-formatter |
| `/gen-user-stories --then-e2e` | [same as above] → for each US: full manual TC pipeline → full automation pipeline |

**Input Type 1 — User Story → Manual TCs → (optional) Automation:**

| Slash Command | Agents / Skills Invoked |
|---|---|
| `/gen-manual-tests` | user-story-parser skill → quality-master-orchestrator → quality-orchestrator → 8 specialist agents → quality-evaluator → gap-filler → tc-formatter skill |
| `/e2e` | [same as above] → intake-agent → locator-agent → pom-agent → testgen/bdd-agent → reporting-agent |
| `/generate-automation-tests --story` | user-story-parser → quality-orchestrator (mode 2, no eval) → tc-formatter → intake-agent → locator-agent → pom-agent → testgen/bdd-agent |

**Input Type 2 — Test Cases → Automation (original behavior, unchanged):**

| Slash Command | Agents Spawned (in order) |
|---|---|
| `/setup-kit` | scaffolder-agent |
| `/ingest` | intake-agent only |
| `/generate-automation-tests --cases` | intake-agent → locator-agent → pom-agent → testgen-agent or bdd-agent → reporting-agent |
| `/generate-locators` | locator-agent only |
| `/generate-page-objects` | pom-agent (reads existing selectors.json) |
| `/gen-bdd` | intake-agent → bdd-agent |
| `/gen-data` | datagen-agent |
| `/add-test` | testgen-agent or bdd-agent (spec-only mode) |
| `/setup-reporting` | reporting-agent |
| `/setup-ci` | ci-agent |
| `/run-smoke` | No agent — shell commands directly |
| `/run-test` | No agent — shell commands directly |
| `/validate-kit` | No agent — validation commands directly |

### 3.4 testStyle Routing Rule

After intake completes, `testStyle` is read from `intake.summary.json`:

| `testStyle` | Agents spawned |
|---|---|
| `"nonbdd"` | testgen-agent only |
| `"bdd"` | bdd-agent only |
| `"both"` | testgen-agent + bdd-agent |

### 3.5 Quality Rules (Non-Negotiable)

1. Never generate test code without confirmed locators
2. Always use Page Object Model — locators live in POM files only
3. BDD artifacts are always paired (.feature + .steps.ts in same run)
4. Validate after generation: `npx tsc --noEmit`
5. AI-guided locators are always flagged — scores capped at 0.75
6. One Page Object per UI component/page
7. No `waitForTimeout` — use `expect(locator).toBeVisible()` or `waitFor`
8. Selector file name is always `selectors.json` (never `locators.json`)
9. All generation commands support `--dry-run`

---

## 4. Slash Commands

All commands live in `.claude/commands/`. They are invoked as `/command-name [args]`.

### `/gen-user-stories`
**File**: `.claude/commands/gen-user-stories.md`
**Purpose**: Input Type 0 entry point — extract user stories from requirement documents (BRD, PRD, Feature Spec, meeting notes, images, etc.).

| Argument | Description |
|---|---|
| `--req <path>` | File path, comma-separated list, or directory |
| `--type <doctype>` | Doc type hint: `brd`, `prd`, `feature`, `epic`, `workflow`, `data`, `api`, `ux`, `meeting`, `email`, `ticket`, `other` |
| `--limit <n>` | Max user stories to generate (default: 20) |
| `--dry-run` | Preview features without writing |
| `--then-manual-tests` | Auto-feed each story into `/gen-manual-tests` |
| `--then-e2e` | Auto-feed each story into `/e2e` |
| `--url`, `--dom`, `--style` | Forwarded to `/e2e` when `--then-e2e` is set |
| `--project <name>` | Override project name |

**Pipeline**: req-ingestor → req-analyzer → feature-splitter → user-story-writer → quality lint → summary

**Supported formats**: `.pdf`, `.md`, `.txt`, `.docx` (via pandoc), `.xlsx` (via pandas), `.yaml`, `.json`, `.xml`, `.bpmn`, `.csv`, `.eml`, `.png`, `.jpg`

**Output**: `outputs/<project>/user-stories/US-NNN-<slug>.md` files + `US-INDEX.md`

---

### `/gen-manual-tests`
**File**: `.claude/commands/gen-manual-tests.md`
**Purpose**: User story → manual test case documents (BDD `.feature` or non-BDD `.md`). Does NOT touch the automation pipeline.

| Argument | Description |
|---|---|
| `--story <path\|inline>` | User story file (`.md`, `.txt`, `.pdf`, `.yml`) or inline text |
| `--style bdd\|nonbdd\|both` | Output format (default: from kit.config.json) |
| `--types <list>` | Comma-separated specialist types (default: auto from analyzer) |
| `--mode 1\|2\|3\|4` | Pipeline mode (1=full with eval, 2=generation only, 3=re-evaluate existing, 4=single phase) |
| `--project <name>` | Override project name |
| `--dry-run` | Preview without writing |

**Pipeline**: user-story-parser → quality-master-orchestrator (Phases 0–3) → tc-formatter → quality lint → summary

**Output**: `outputs/<project>/manual-tests/manual-tcs/bdd/` and/or `outputs/<project>/manual-tests/manual-tcs/nonbdd/`

---

### `/e2e`
**File**: `.claude/commands/e2e.md`
**Purpose**: Full end-to-end: user story → manual TCs → automation artifacts.

| Argument | Description |
|---|---|
| `--story <path\|inline>` | User story file (`.md`, `.txt`, `.pdf`, `.yml`) or inline text |
| `--style bdd\|nonbdd\|both` | TC format + automation test style |
| `--url <url>` | Live app URL for Tier-1 locator extraction |
| `--dom <path>` | DOM snapshot directory/file for Tier-2 locator extraction |
| `--types <list>` | Specialist types to run |
| `--mode 1\|2\|3\|4` | Orchestrator mode |
| `--skip-automation` | Stop after manual TCs — skip automation pipeline |
| `--project <name>` | Override project name |
| `--dry-run` | Preview without writing |

**Pipeline**: user-story-parser → quality-master-orchestrator → tc-formatter → [intake-agent → locator-agent → pom-agent → testgen/bdd-agent → reporting-agent] → validate → summary

---

### `/generate-automation-tests`
**File**: `.claude/commands/generate-automation-tests.md`
**Purpose**: Full automation pipeline — from inputs to runnable test artifacts. Accepts `--cases` (Input Type 2) or `--story` (Input Type 1, quick mode).

| Argument | Description |
|---|---|
| `--story <path\|inline>` | Input Type 1: user story (runs manual TC pipeline first, mode 2) |
| `--cases <path\|inline>` | Input Type 2: test case text or file path |
| `--url <url>` | Live app URL (Playwright MCP strategy) |
| `--dom <path>` | Directory of `.html` snapshots OR single file |
| `--bdd` | Force BDD output (overrides strategy default) |
| `--story-mode 1\|2` | When `--story` used: orchestrator mode (default: 2) |
| `--style bdd\|nonbdd\|both` | When `--story` used: manual TC format + test style |
| `--project <name>` | Override project name |
| `--dry-run` | Preview without writing |
| `--tags <tag,...>` | Only generate for matching tags |

**`--story` and `--cases`** are mutually exclusive. `--story` takes precedence.

**Pipeline**: Load context → [user-story-parser → tc-formatter if `--story`] → Resolve domFiles → intake-agent → locator-agent → pom-agent → testgen/bdd-agent → reporting-agent → tsc validate → summary

**`--dom` directory scanning**: When a directory path is passed, all `*.html` files are auto-discovered and sorted alphabetically. Page names are inferred from filenames using `inferPageName()` (kebab/snake → PascalCase).

### `/ingest`
**File**: `.claude/commands/ingest.md`
**Purpose**: Run intake-agent standalone to preview how inputs will be classified.
Produces `intake.summary.json` without generating any test code.

### `/generate-locators`
**File**: `.claude/commands/generate-locators.md`
**Purpose**: Run locator-agent only. Produces or updates `selectors.json`.

### `/generate-page-objects`
**File**: `.claude/commands/generate-page-objects.md`
**Purpose**: Run pom-agent only. Reads existing `selectors.json` and generates POM files.

### `/gen-bdd`
**File**: `.claude/commands/gen-bdd.md`
**Purpose**: Generate BDD feature files + step definitions only.

### `/gen-data`
**File**: `.claude/commands/gen-data.md`
**Purpose**: Generate test data fixtures using the datagen-agent.

### `/add-test`
**File**: `.claude/commands/add-test.md`
**Purpose**: Add a single new test to an existing project. Reuses existing POMs; creates new POMs only for missing components.

| Argument | Description |
|---|---|
| `--case <text\|file>` | Test case description (inline or file path) |
| `--page <PageName>` | Existing POM to reuse — skips locator extraction entirely |
| `--url <url>` | Live app URL — triggers Tier 1 (Playwright MCP) locator extraction for missing POMs |
| `--dom <path>` | HTML snapshot file or directory — triggers Tier 2 locator extraction for missing POMs |
| `--bdd` | Generate BDD format (feature + step def) instead of plain spec |

**Locator strategy (resolved automatically when a POM is missing):**

| Condition | Tier | Strategy |
|-----------|------|----------|
| `--page` provided and POM exists | — | Skip locator extraction entirely |
| `--url` provided | 1 | Playwright MCP (highest confidence) |
| `--dom` provided, no `--url` | 2 | DOM snapshot extraction |
| Neither `--url` nor `--dom` | 3 | AI-guided (scores capped at 0.75 — verify against app) |

**Key behaviour**: Never modifies existing spec files. Each call generates a NEW spec file.

### `/setup-reporting`
**File**: `.claude/commands/setup-reporting.md`
**Purpose**: Configure Allure + Playwright HTML reporting in `playwright.config.ts`.

### `/setup-ci`
**File**: `.claude/commands/setup-ci.md`
**Purpose**: Scaffold `.github/workflows/ui.yml` CI pipeline.

### `/run-smoke`
**File**: `.claude/commands/run-smoke.md`
**Purpose**: Execute `@smoke`-tagged tests across both non-BDD (Playwright) and BDD (Cucumber) suites, generate reports, surface pass/fail summary.

| Argument | Description |
|---|---|
| `--tags <expr>` | Override tag filter (default: `@smoke`) |
| `--browser <name>` | Browser (default: chromium) |
| `--headed` | Run headed mode |
| `--report-only` | Skip run, regenerate reports only |
| `--nonbdd-only` | Run Playwright specs only (skip Cucumber) |
| `--bdd-only` | Run Cucumber feature files only (skip specs) |

**Run order**: Playwright specs first (`--grep "@smoke"`), then `npx cucumber-js --tags @smoke`. Results aggregated into one summary.

### `/run-test`
**File**: `.claude/commands/run-test.md`
**Purpose**: Execute tests across both non-BDD (Playwright) and BDD (Cucumber) suites with optional tag filtering.

| Argument | Description |
|---|---|
| `--tags <expr>` | Tag filter (default: all tests) |
| `--browser <name>` | Browser (default: chromium) |
| `--headed` | Run headed mode |
| `--report-only` | Skip run, regenerate reports only |
| `--nonbdd-only` | Run Playwright specs only |
| `--bdd-only` | Run Cucumber feature files only |

### `/validate-kit`
**File**: `.claude/commands/validate-kit.md`
**Purpose**: Run `tsc --noEmit`, ESLint, `playwright test --list`, and BDD dry-run.

### `/setup-kit`
**File**: `.claude/commands/setup-kit.md`
**Purpose**: Scaffold a new kit project (installs dependencies, creates directory structure, generates `playwright.config.ts`).

---

## 5. Code Generation Rules

Rule files in `tc-to-automate/rules/` are automatically applied when generating or editing matching files.

### playwright-ts.md
**Applied to**: `*.spec.ts`, `*.test.ts`, `*.page.ts`

Key rules:
- Import only from `@playwright/test`
- Page classes are plain TypeScript — no base class extension
- All locators are `readonly` class properties
- Locator priority: `getByTestId` > `getByRole` > `getByLabel` > `getByPlaceholder` > CSS `data-*` > CSS `id` > CSS `class`
- Never use XPath, `page.$`, `page.$$`, or positional selectors
- All action methods return `Promise<void>`
- No assertions inside Page Objects
- Never use `waitForTimeout()` — use `expect(locator).toBeVisible()`
- Config must include both `html` and `allure-playwright` reporters

### page-objects.md
**Applied to**: `*Page.ts`, `*PO.ts`, `*Page.java`, `*PO.java`

Key rules:
- One POM per page/component
- All locators defined as `readonly` class-level properties
- Action methods include only user-facing interactions
- No assertions, no conditional logic, no sleeps, no test data generation in POMs
- TypeScript actions return `Promise<void>`; Java returns `void`

### bdd-gherkin.md
**Applied to**: `*.feature`

Key rules:
- One `Feature:` per file; unique `Scenario:` titles
- Steps use only: `Given`, `When`, `Then`, `And`, `But`
- `Then` steps contain ONE assertion each
- **Mandatory tags**: `@ui`/`@api` on Feature; `@TC-<id>` on every Scenario
- Tag taxonomy: `@smoke`, `@regression`, `@module(<name>)`, `@wip`, `@negative`, `@boundary`
- Strict mode: no UI implementation details in step text (no "click the button", use "submit the form")

### selenium-java.md
**Applied to**: `*Test.java`, `*Page.java`, `*Steps.java`

Key rules:
- Use `WebDriverManager` — never manually manage drivers
- Locator priority: `By.id` > `By.name` > `By.cssSelector([data-testid])` > `By.cssSelector([aria-label])` > `By.xpath`
- Mandatory `WebDriverWait` — no `Thread.sleep()`
- TestNG `@BeforeClass`/`@AfterClass` for driver lifecycle

### python-test.md
**Applied to**: `test_*.py`, `*_test.py`, `conftest.py`

Key rules:
- Test functions named `test_<tc_id>_<description>`
- All shared setup in `conftest.py` fixtures
- API: always assert status code first
- Selenium: `WebDriverWait` only — no `time.sleep()`

---

## 6. settings.json — MCP Servers, Permissions & Hooks

**Location**: `.claude/settings.json`

### MCP Servers

| Server | Command | Purpose |
|---|---|---|
| `playwright` | `npx @playwright/mcp@latest --headless` | Tier-1 locator extraction — live browser automation |
| `filesystem` | `npx @modelcontextprotocol/server-filesystem .` | Read/write project files |
| `git` | `npx @modelcontextprotocol/server-git --repository .` | Commit generated artifacts |
| `shell` | `npx @modelcontextprotocol/server-shell` | Run npm/playwright/tsc commands |

### Permissions

**Allowed**:
- All `mcp__playwright__browser_*` operations (navigate, snapshot, click, fill, close, etc.)
- All `mcp__filesystem__*` operations (read, write, list, create directory, move)
- All `mcp__git__*` operations (init, add, commit, status, log)
- `Bash(npm install *)`, `Bash(npm run *)`, `Bash(npm ci)`
- `Bash(npx playwright install*)`, `Bash(npx playwright test*)`
- `Bash(npx tsc*)`, `Bash(npx cucumber-js*)`, `Bash(npx allure*)`
- `Bash(mvn *)`, `Bash(pytest *)`

**Denied** (safety guardrails):
- `Bash(rm -rf *)` — no destructive deletes
- `Bash(git push --force*)` — no force pushes
- `Bash(git push -f *)` — no force pushes

### Hooks

| Event | Matcher | Action |
|---|---|---|
| `PostToolUse` | `Write\|Edit` | Logs: `[Kit] File written: <path>` |

---

## 7. Tools Used

### Claude Code Built-in Tools

| Tool | Used For |
|---|---|
| `Read` | Reading kit config, agent files, DOM snapshots, existing test files |
| `Write` | Creating new POM files, spec files, JSON artifacts |
| `Edit` | Patching existing files (e.g., adding tags, fixing imports) |
| `Grep` | Searching for patterns in source files (hooks, selectors, imports) |
| `Glob` | Discovering files by pattern (e.g., `inputs/snapshots/*.html`, `**/*.spec.ts`) |
| `Bash` | Running npm/playwright/tsc/allure commands |
| `Agent` | Spawning specialised sub-agents (Explore for DOM analysis, Plan for architecture) |
| `Skill` | Executing slash commands |
| `TodoWrite` | Tracking multi-step task progress |
| `WebSearch` | Researching locator patterns or framework APIs |

### MCP Tools (via settings.json)

| Tool | Used For |
|---|---|
| `mcp__playwright__browser_navigate` | Open page in headless browser (Tier-1 locators) |
| `mcp__playwright__browser_snapshot` | Capture accessibility tree + DOM |
| `mcp__playwright__browser_click` | Interact with elements to reveal hidden DOM |
| `mcp__playwright__browser_fill` | Fill form fields during MCP-based extraction |
| `mcp__playwright__browser_close` | Close browser after extraction |
| `mcp__playwright__browser_screenshot` | Capture visual evidence |
| `mcp__filesystem__read_file` | Read files via filesystem MCP |
| `mcp__filesystem__write_file` | Write files via filesystem MCP |
| `mcp__git__git_add` / `git_commit` | Commit generated artifacts (GitHub plugin) |

---

## 10. Capability 3 — TC-to-Automate Agents

All automation agents live in `tc-to-automate/kits/_base/agents/`. They are kit-aware and invoked by reading
their `.md` file. Agents never run themselves — CLAUDE.md dispatches them via the Agent Dispatch Table.

### intake-agent.md
**Role**: Step 0 — normalise all inputs, produce `intake.summary.json`.

Key responsibilities:
- Normalise `domFiles` input: single path, comma-separated, array of `{pageName, file}`, or directory
- Infer page names from filenames using `inferPageName()` (e.g., `amazonHome-page.html` → `AmazonHomePage`)
- Determine strategy: `playwright-mcp` > `dom-based` > `ai-guided`
- Parse test cases using `test-case-parser` skill
- Detect page transitions per step (populates `step.pageContext` + `step.transitionTo`)
- Build `pageTransitions[]` array per test case
- Warn on: missing DOM files, unmatched page contexts, unreachable URLs, no test cases
- Write `intake.summary.json` to `<outputDir>/<projectName>/`

**Output**: `intake.summary.json`

### locator-agent.md
**Role**: Extract UI element locators and produce `selectors.json`.

Tiers:
- **Tier 1 (playwright-mcp)**: Live browser — highest precision
- **Tier 2 (dom-based)**: Static HTML files — iterates over each `domFiles` entry, merges all page blocks into single `selectors.json`
- **Tier 3 (ai-guided)**: No source — inferred from test case text, all scores capped at 0.75

For multi-page (Tier 2): filters test case steps by `step.pageContext` to focus extraction on elements needed for that page. Identifies navigation triggers via `step.transitionTo`.

**Output**: `selectors.json`, `selectors-lint.html`

### pom-agent.md
**Role**: Generate Page Object TypeScript class files from `selectors.json`.

- One POM class file per top-level key in `selectors.json`
- Action method inference from locator alias names (`*Input` → `fill<X>()`, `*Button` → `click<X>()`)
- `pageRoute` inferred from class name (`LoginPage` → `/login`)
- Skips existing files unless `overwrite = true`

**Output**: `src/pages/<ComponentName>.ts`

### testgen-agent.md
**Role**: Generate non-BDD Playwright spec files from `intake.summary.json` + existing POMs.

- One spec file per test case
- Uses `step.pageContext` and `pageTransitions` to import correct POM classes and route steps
- Inserts `await page.waitForURL(...)` at every `transitionTo` step
- All steps wrapped in `test.step()` for Allure hierarchy

**Output**: `tests/nonbdd/<TC-ID>-<slug>.spec.ts`

### bdd-agent.md
**Role**: Generate BDD feature files + step definitions.

- Applies `bdd-gherkin.md` rules strictly
- Mandatory tag taxonomy enforced
- Shared POMs are referenced — no duplicate locators in step definitions

**Output**: `tests/bdd/<feature>.feature` + `tests/bdd/<feature>.steps.ts`

### datagen-agent.md(TBD)
**Role**: Generate test data fixtures using `data-factory` skill.

**Output**: `src/fixtures/<name>.fixtures.ts`

### reporting-agent.md
**Role**: Configure `playwright.config.ts` with Allure + HTML reporters.

- Only patches config if allure reporter is not already present
- Sets `screenshot: 'only-on-failure'`, `video: 'retain-on-failure'`, `trace: 'on-first-retry'`

### ci-agent.md(TBD)
**Role**: Scaffold `.github/workflows/ui.yml` CI pipeline.

**Output**: `configs/.github/workflows/ui.yml`

### framework-setup-agent.md (kit-u1 specific)
**Location**: `tc-to-automate/kits/kit-u1/agents/framework-setup-agent.md`
**Role**: Full project scaffold — installs dependencies, creates directory structure, generates `playwright.config.ts`, `tsconfig.json`, `package.json`.

---

## 8. Capability 1 — Req-to-Story Module

The `req-to-story/` directory handles Input Type 0 (requirement document → user stories).
Kit-independent — does not depend on any kit files.

```
req-to-story/
├── agents/
│   ├── req-ingestor.agent.md        ← format detection + text extraction (all 27 req types)
│   ├── req-analyzer.agent.md        ← doc type classification, actors, NFRs, scope
│   ├── feature-splitter.agent.md    ← splits doc into discrete features (feature-map.json)
│   └── user-story-writer.agent.md   ← generates US-NNN.md per feature
├── skills/
│   └── req-text-extractor.md        ← extraction rules per format tier
└── rules/
    └── user-story-quality.md        ← AC specificity, role clarity, traceability rules
```

### Agents

| Agent | Role |
|-------|------|
| `req-ingestor.agent.md` | Format detection + text extraction (Tier 1/2/3) — always runs first |
| `req-analyzer.agent.md` | Doc type classification + actors + NFRs + scope |
| `feature-splitter.agent.md` | Splits doc into discrete features → `feature-map.json` |
| `user-story-writer.agent.md` | Generates `US-NNN.md` per feature with acceptance criteria |

### `/gen-user-stories` Arguments

| Argument | Description |
|---|---|
| `--req <path>` | File, comma-separated list, or directory of requirement docs |
| `--type <doctype>` | Hint: `brd`, `prd`, `feature`, `epic`, `workflow`, `data`, `api`, `ux`, `meeting`, `email`, `ticket`, `other` |
| `--limit <n>` | Max user stories per run (default: 20) |
| `--dry-run` | Preview features without writing |
| `--then-manual-tests` | Auto-feed each generated story into `/gen-manual-tests` |
| `--then-e2e` | Auto-feed each generated story into `/e2e` |

### File Format Extraction Tiers

| Tier | Formats | Method |
|------|---------|--------|
| 1 — Native | `.md`, `.txt`, `.eml`, `.pdf`, `.yaml`, `.json`, `.xml`, `.bpmn`, `.csv`, `.png`, `.jpg` | `Read` tool (multimodal for images) |
| 2 — Shell | `.docx` | `pandoc` → markdown (fallback: `docx2txt`) |
| 2 — Shell | `.xlsx` | `pandas` → CSV (fallback: save as CSV) |
| 3 — Unsupported | `.fig` | User prompted to export as PDF/PNG |

### Output Structure

```
outputs/<projectName>/user-stories/
├── 00-analysis/
│   ├── extraction-manifest.json
│   └── req-analysis.json
├── 01-features/
│   └── feature-map.json
├── US-001-<slug>.md               ← ready for /gen-manual-tests --story
├── US-002-<slug>.md
└── US-INDEX.md
```

---

## 9. Capability 2 — Story-to-Testcase Module

The `story-to-testcase/` directory is a self-contained, kit-independent module. It handles all of
Input Type 1 (user story → manual TCs). It does not depend on any kit files.

```
story-to-testcase/
├── agents/
│   ├── orchestration/
│   │   ├── quality-master-orchestrator.agent.md  ← 5-phase coordinator; bridge to automation
│   │   ├── quality-orchestrator.agent.md          ← coordinates 8 specialist agents
│   │   └── quality-evaluator.agent.md             ← quality scoring + gap analysis
│   ├── analysis/
│   │   ├── user-story-analyzer.agent.md          ← test type applicability + confidence scoring
│   │   └── gap-filler.agent.md                   ← fills auto-fillable coverage gaps
│   └── specialists/
│       ├── positive-scenarios.agent.md           ← happy path (3–7 scenarios)
│       ├── negative-scenarios.agent.md           ← validation failures, error handling
│       ├── edge-cases.agent.md                   ← boundary conditions, min/max values
│       ├── integration-scenarios.agent.md        ← cross-component, names 2+ systems
│       ├── security-scenarios.agent.md           ← OWASP Top 10, attack payloads
│       ├── performance-scenarios.agent.md        ← only when explicit SLAs exist
│       ├── api-scenarios.agent.md                ← REST contract, endpoints, status codes
│       └── ui-scenarios.agent.md                 ← domain language, no raw selectors
├── skills/
│   ├── user-story-parser.md     ← parses 4 story formats → structured JSON
│   ├── tc-formatter-bdd.md      ← raw Gherkin → tagged .feature files (kit rules applied)
│   └── tc-formatter-nonbdd.md   ← raw Gherkin → TC-ID / Steps / Expected .md
└── rules/
    └── manual-tc-quality.md     ← universal and format-specific quality rules
```

### 5-Phase Workflow

| Phase | Agent | Output dir | Mode |
|-------|-------|-----------|------|
| 0 | user-story-analyzer | `00-analysis/` | always |
| 1 | quality-orchestrator + 8 specialists | `01-scenarios/` | always |
| 2 | quality-evaluator | `02-evaluation/` | mode 1 only |
| 3 | gap-filler | `03-gap-filled/` | mode 1, if gaps exist |
| 4 | tc-formatter skill | `manual-tcs/` | always |

### User Story Parser

Parses 4 formats:
- **Agile Standard**: `As a <role> / I want <goal> / So that <benefit>`
- **Plain English**: `# Feature: X` + description paragraphs
- **Structured Spec**: `## Feature` with `**User Story**`, `**Acceptance Criteria**` keys
- **Inline / Mixed**: free-form text with roles, goals, criteria in any order

Supported file types: `.md`, `.txt`, `.pdf`, `.yml`/`.yaml`.

### TC Formatter Skills (Bridge to Automation)

`tc-formatter-bdd.md`: applies kit tagging rules from `bdd-gherkin.md` — assigns `@TC-NNN`,
`@ui`/`@api`, `@smoke`/`@regression`, `@module()`, `@negative`, `@boundary`. Merges Phase 1 + Phase 3 output.

`tc-formatter-nonbdd.md`: converts Gherkin Given/When/Then → TC-ID tables with Preconditions,
Steps, Expected Results. Groups TCs by type with reserved ID ranges (TC-001–019 positive, TC-020–039 negative, etc.).

The formatted output in `manual-tcs/` is passed to `intake-agent` as `testCasesRaw` when
`handoffToAutomation = true` (triggered by `/e2e` without `--skip-automation`).

---

## 11. Skills

Skills are reusable logic modules in `tc-to-automate/kits/_base/skills/`. Agents invoke them by reading the skill
file and applying its rules.

### test-case-parser.md
Parses free-form test case text into structured JSON objects.

Supported formats: numbered list, Gherkin-like, plain paragraph, table.

Key outputs per test case:
- `testCaseId`, `title`, `tags`, `preconditions`, `steps[]`, `expectedResults[]`, `pages[]`
- `step.pageContext` — which page each step executes on
- `step.transitionTo` — destination page for navigation-triggering steps
- `pageTransitions[]` — ordered list of page hops (used by pom-agent + testgen-agent)

**Page transition detection**: Tracks `currentPage` as it parses steps. Detects transition signals:
- `action == "navigate"` with new URL/page
- `action == "click"` targeting CTAs (search icon, login button, submit, checkout, etc.)
- Expected result mentions a different page

### selector-scorer.md
Assigns a numeric confidence score (0.0–1.0) to each candidate selector.

| Attribute | Score |
|---|---|
| `data-testid` | 0.99 |
| `aria-label` + role | 0.95 |
| `placeholder` | 0.90 |
| `role` | 0.90 |
| `id` (stable) | 0.85 |
| `data-*` attribute | 0.85 |
| Stable CSS class | 0.70 |
| Dynamic class / XPath | < 0.70 |

AI-guided cap: all scores ≤ 0.75 regardless of attribute type.

### locator-strategies.md
Defines the selector priority ladder used by locator-agent:
`getByTestId` > `getByRole` > `getByLabel` > `getByPlaceholder` > CSS `data-*` > CSS `id` > CSS `class`

### template-renderer.md
Rendering rules applied when populating `.tmpl` files for POM and spec generation.

### gherkin-transformer.md
Transforms structured test case objects (from test-case-parser) into Gherkin syntax.

### data-factory.md(TBD)
Generates typed TypeScript fixture files with realistic test data using `@faker-js/faker`.

### ci-scaffolder.md(TBD)
Generates GitHub Actions YAML workflow files for Playwright test execution in CI.

---

## 12. Kit Configuration (kit.config.json)

**Location**: `kit.config.json` (project root)

```json
{
  "id": "kit-u1",
  "extends": null,
  "displayName": "UI Greenfield — Playwright TypeScript",
  "mode": "greenfield",
  "tech": {
    "language": "typescript",
    "testRunner": "playwright",
    "apiClient": null,
    "bddRunner": null,
    "bddRunnerBdd": "@cucumber/cucumber"
  },
  "testStyle": "both",
  "locatorStrategies": ["playwright-mcp", "dom-based", "ai-guided"],
  "reporting": ["playwright-html", "allure"],
  "outputDir": "./outputs",
  "templates": "./tc-to-automate/kits/kit-u1/templates",
  "agents": { ... },
  "namingConventions": { ... },
  "strategyDefaults": { ... },
  "dependencies": { ... },
  "projectName": "playwright-greenfieldproject"
}
```

### Key Fields

| Field | Purpose |
|---|---|
| `id` | Identifies which kit is active — used to load `tc-to-automate/kits/<id>/KIT.md` |
| `mode` | `"greenfield"` (new project) or `"integration"` (existing codebase) |
| `testStyle` | Top-level default: `"nonbdd"`, `"bdd"`, or `"both"` |
| `locatorStrategies` | Ordered list of supported strategies (Tier 1 → 3) |
| `outputDir` | Root directory for all generated artifacts |
| `templates` | Path to kit-specific Handlebars templates |
| `projectName` | Subdirectory under `outputDir` for this project's files |

### strategyDefaults

Controls which test style is generated based on locator strategy:

| Strategy | Default testStyle | Reason |
|---|---|---|
| `playwright-mcp` | `"both"` | Live browser → reliable locators → full artifact set |
| `dom-based` | `"nonbdd"` | Static DOM → spec files; BDD deferred until locators confirmed |
| `ai-guided` | `"nonbdd"` | Inferred locators → spec only; BDD deferred |

**Override**: `--bdd` flag always forces BDD output regardless of strategy.

### namingConventions

| Convention | Value |
|---|---|
| Page Object suffix | `Page` (e.g., `LoginPage`) |
| Spec file suffix | `.spec.ts` |
| Feature file dir | `tests/bdd` |
| Steps file dir | `tests/bdd` |
| Step file suffix | `.steps.ts` |
| POM dir | `src/pages` |
| Test dir (nonbdd) | `tests/nonbdd` |
| Fixtures dir | `src/fixtures` |

---

## 13. Output Structure

```
outputs/
└── <projectName>/
    ├── manual-tests/                ← Input Type 1 output (user story → manual TCs)
    │   ├── 00-analysis/
    │   │   └── test-type-analysis.json   ← Applicability scores for 8 test types
    │   ├── 01-scenarios/            ← Raw Gherkin from specialist agents
    │   ├── 02-evaluation/           ← Quality report + completeness-analysis.json (mode 1)
    │   ├── 03-gap-filled/           ← Gap-filled scenarios (mode 1, if gaps found)
    │   ├── manual-tcs/
    │   │   ├── bdd/
    │   │   │   ├── <featureSlug>.feature  ← Final tagged BDD manual TCs
    │   │   │   └── TC-INDEX.md
    │   │   └── nonbdd/
    │   │       ├── <featureSlug>.md       ← TC-ID / Steps / Expected format
    │   │       └── TC-INDEX.md
    │   └── MASTER-REPORT.md
    ├── intake.summary.json          ← Step 0 output — input contract for all agents
    ├── selectors.json               ← Locator map (one top-level key per page)
    ├── selectors-lint.html          ← Selector quality report (flags score < 0.70)
    ├── playwright.config.ts         ← Playwright configuration
    ├── tsconfig.json                ← TypeScript configuration with @pages/* alias
    ├── package.json                 ← npm dependencies
    ├── node_modules/                ← Installed dependencies
    ├── src/
    │   ├── pages/                   ← Page Object Model classes
    │   │   ├── LoginPage.ts
    │   │   └── SecureAreaPage.ts
    │   └── fixtures/                ← Test data factories
    │       └── auth.fixtures.ts
    ├── tests/
    │   ├── nonbdd/                  ← Playwright spec files (automation)
    │   │   └── TC-01-successful-login.spec.ts
    │   └── bdd/                     ← Feature files + step definitions (automation)
    │       ├── login.feature
    │       └── login.steps.ts
    └── configs/
        └── .github/
            └── workflows/
                └── ui.yml           ← CI pipeline (generated by /setup-ci)
```

### Key intermediate files

| File | Description | Preserved between runs? |
|---|---|---|
| `intake.summary.json` | Normalised input + page transitions | Yes — cached if inputs unchanged |
| `selectors.json` | Locator map for all pages | Yes — prompt before overwrite |
| `selectors-lint.html` | Selector quality report | Overwritten each generation |

---

## 14. Plugins & Integrations

**Location**: `tc-to-automate/configs/integrations.yml`

All plugins are **disabled by default**. Enable by setting `enabled: true` and providing environment variables.

### Jira Plugin

```yaml
jira:
  enabled: false
  baseUrl: ${JIRA_URL}
  projectKey: QA
  apiToken: ${JIRA_API_TOKEN}
  importTestCases: false    # pull test cases from Jira during /ingest
  pushResults: false        # push execution results after /run-smoke
```

When enabled: `/ingest` can fetch test cases from Jira; `/run-smoke` pushes results back.

### GitHub Plugin

```yaml
github:
  enabled: false
  repoOwner: ${GITHUB_OWNER}
  repoName: ${GITHUB_REPO}
  token: ${GITHUB_TOKEN}
  createPr: true
  prBaseBranch: main
  commitMessage: "feat: generated kit-u1 test artifacts"
```

When enabled: after any generation command, artifacts are committed and a PR is created automatically.

### Allure Server Plugin

```yaml
allureServer:
  enabled: false
  serverUrl: ${ALLURE_SERVER_URL}
  projectId: ${ALLURE_PROJECT_ID}
  apiToken: ${ALLURE_TOKEN}
  publishAfterRun: true     # auto-publish after /run-smoke
```

When enabled: after `/run-smoke`, Allure results are published to a remote Allure Server instance.

---

## 15. Guardrails

Defined in `tc-to-automate/configs/integrations.yml`:

| Guardrail | Value | Effect |
|---|---|---|
| `selectorScoreThreshold` | `0.70` | Selectors below this flagged in `selectors-lint.html` |
| `forbidXpathDepthGreaterThan` | `3` | XPath with > 3 levels flagged |
| `forbidSleep` | `true` | `sleep()` / `waitForTimeout()` flagged in generated code |
| `forbidDynamicIds` | `true` | `id=ember*`, `id=react-*`, `id=css-*` patterns flagged |
| `requireTagOnEveryScenario` | `true` | All BDD scenarios must have `@ui` or `@api` + `@TC-*` |
| `idempotentGeneration` | `true` | Generation commands never overwrite files without confirmation |

---

## 16. Locator Strategy Tiers

| Tier | Strategy | Trigger | Precision | Score cap |
|---|---|---|---|---|
| 1 | `playwright-mcp` | `--url` provided | Highest — live DOM | None |
| 2 | `dom-based` | `--dom` provided | High — static DOM | None |
| 3 | `ai-guided` | Neither provided | Moderate — inferred | 0.75 |

**Tier 2 multi-page flow**: `--dom <directory>` scans all `.html` files, processes each independently, and merges all page blocks into a single `selectors.json`.

---

## 17. Generated Project Config

### playwright.config.ts (generated)

```typescript
{
  testDir: './tests/nonbdd',
  fullyParallel: false,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  reporter: [
    ['html', { outputFolder: 'playwright-report', open: 'never' }],
    ['allure-playwright', { detail: true, outputFolder: 'allure-results' }],
    ['list']
  ],
  use: {
    baseURL: process.env.BASE_URL || 'https://example.com/',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry',
  }
}
```

### tsconfig.json (generated)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "paths": {
      "@pages/*": ["src/pages/*"],
      "@fixtures/*": ["src/fixtures/*"],
      "@tests/*": ["tests/*"]
    }
  }
}
```

The `@pages/*` path alias means spec files import POMs as:
```typescript
import { LoginPage } from '@pages/LoginPage';
```

---

*Last updated: 2026-03-13 | Kit version: 2.0 | Three capabilities: req-to-story / story-to-testcase / tc-to-automate*
