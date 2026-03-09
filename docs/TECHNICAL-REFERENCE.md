# Automation Kit — Technical Reference
## Kit U1: UI Greenfield — Playwright TypeScript

**Version**: 1.0 | **Kit ID**: `kit-u1` | **Date**: 2026-03-09

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [CLAUDE.md — The Orchestrator](#2-claudemd--the-orchestrator)
3. [Slash Commands](#3-slash-commands)
4. [Code Generation Rules](#4-code-generation-rules)
5. [settings.json — MCP Servers, Permissions & Hooks](#5-settingsjson--mcp-servers-permissions--hooks)
6. [Tools Used](#6-tools-used)
7. [Base Agents](#7-base-agents)
8. [Skills](#8-skills)
9. [Kit Configuration (kit.config.json)](#9-kit-configuration-kitconfigjson)
10. [Output Structure](#10-output-structure)
11. [Plugins & Integrations](#11-plugins--integrations)
12. [Guardrails](#12-guardrails)
13. [Locator Strategy Tiers](#13-locator-strategy-tiers)
14. [Generated Project Config](#14-generated-project-config)

---

## 1. Architecture Overview

The Automation Kit is an AI-driven test generation system built on top of **Claude Code**. It uses
a layered architecture:

```
┌─────────────────────────────────────────────────────────┐
│  User Input (URL / DOM snapshots / Test case text)       │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│  CLAUDE.md  —  Orchestrator                              │
│  Reads kit config, routes commands to agents             │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   intake-agent    locator-agent      pom-agent
   (Step 0,        (Tier 1/2/3)       (POM classes)
    always first)
        │                                 │
        └────────────────┬────────────────┘
                         ▼
              testgen-agent / bdd-agent
              (spec files / feature+steps)
                         │
                         ▼
                  reporting-agent
                  (playwright.config.ts)
```

All agents communicate through **`intake.summary.json`** — a structured contract written by the
intake-agent and consumed by all downstream agents. No agent skips intake.

---

## 2. CLAUDE.md — The Orchestrator

**Location**: `CLAUDE.md` (project root)

CLAUDE.md is the master instruction file loaded by Claude Code at the start of every session. It
defines all behaviour, routing, and quality rules for the kit.

### 2.1 Startup Protocol

Every session begins with:
1. Read `kit.config.json` — note `id`, `mode`, `tech`, `testStyle`, `locatorStrategies`, `outputDir`
2. Read `kits/<kit-id>/KIT.md` — kit-specific naming conventions and generation rules
3. Read `configs/integrations.yml` — check active plugins (jira, github, allureServer)
4. Confirm active kit: `Active kit: <id> | <tech.testRunner> | <testStyle>`

### 2.2 Input Detection Protocol

| Input Signal | Classification |
|---|---|
| URL starting with `http://` or `https://` | `app_url = true` |
| HTML content with `<html`, `data-testid`, `aria-`, `id=` | `dom_snapshot = true` |
| Numbered list or plain-English steps | `test_cases = true` |
| JSON with `"info"` + `"paths"` keys | `openapi_spec = true` |
| JSON with `"item"` arrays + request objects | `postman_collection = true` |

### 2.3 Agent Dispatch Table

| Slash Command | Agents Spawned (in order) |
|---|---|
| `/setup-kit` | scaffolder-agent |
| `/ingest` | intake-agent only |
| `/generate-tests` | intake-agent → locator-agent → pom-agent → testgen-agent or bdd-agent → reporting-agent |
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

### 2.4 testStyle Routing Rule

After intake completes, `testStyle` is read from `intake.summary.json`:

| `testStyle` | Agents spawned |
|---|---|
| `"nonbdd"` | testgen-agent only |
| `"bdd"` | bdd-agent only |
| `"both"` | testgen-agent + bdd-agent |

### 2.5 Quality Rules (Non-Negotiable)

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

## 3. Slash Commands

All commands live in `.claude/commands/`. They are invoked as `/command-name [args]`.

### `/generate-tests`
**File**: `.claude/commands/generate-tests.md`
**Purpose**: Full pipeline — from inputs to runnable test artifacts.

| Argument | Description |
|---|---|
| `--url <url>` | Live app URL (Playwright MCP strategy) |
| `--dom <path>` | Directory of `.html` snapshots OR single file |
| `--cases <path\|inline>` | Test case text or file path |
| `--bdd` | Force BDD output (overrides strategy default) |
| `--project <name>` | Override project name |
| `--dry-run` | Preview without writing |
| `--tags <tag,...>` | Only generate for matching tags |

**Pipeline**: Load context → Resolve domFiles → intake-agent → locator-agent → pom-agent → testgen/bdd-agent → reporting-agent → tsc validate → summary

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
**Purpose**: Add a single new test to an existing project. Reuses existing POMs.

| Argument | Description |
|---|---|
| `--case <text\|file>` | Test case description |
| `--page <PageName>` | Existing POM to reuse |
| `--bdd` | Generate BDD format |

### `/setup-reporting`
**File**: `.claude/commands/setup-reporting.md`
**Purpose**: Configure Allure + Playwright HTML reporting in `playwright.config.ts`.

### `/setup-ci`
**File**: `.claude/commands/setup-ci.md`
**Purpose**: Scaffold `.github/workflows/ui.yml` CI pipeline.

### `/run-smoke`
**File**: `.claude/commands/run-smoke.md`
**Purpose**: Execute `@smoke`-tagged tests, generate reports, surface pass/fail summary.

| Argument | Description |
|---|---|
| `--tags <expr>` | Override tag filter (default: `@smoke`) |
| `--browser <name>` | Browser (default: chromium) |
| `--headed` | Run headed mode |
| `--report-only` | Skip run, regenerate reports only |

### `/run-test`
**File**: `.claude/commands/run-test.md`
**Purpose**: Execute a specific test or test file.

### `/validate-kit`
**File**: `.claude/commands/validate-kit.md`
**Purpose**: Run `tsc --noEmit`, ESLint, `playwright test --list`, and BDD dry-run.

### `/setup-kit`
**File**: `.claude/commands/setup-kit.md`
**Purpose**: Scaffold a new kit project (installs dependencies, creates directory structure, generates `playwright.config.ts`).

---

## 4. Code Generation Rules

Rule files in `.claude/rules/` are automatically applied when generating or editing matching files.

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

## 5. settings.json — MCP Servers, Permissions & Hooks

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

## 6. Tools Used

### Claude Code Built-in Tools

| Tool | Used For |
|---|---|
| `Read` | Reading kit config, agent files, DOM snapshots, existing test files |
| `Write` | Creating new POM files, spec files, JSON artifacts |
| `Edit` | Patching existing files (e.g., adding tags, fixing imports) |
| `Grep` | Searching for patterns in source files (hooks, selectors, imports) |
| `Glob` | Discovering files by pattern (e.g., `snapshots/*.html`, `**/*.spec.ts`) |
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

## 7. Base Agents

All agents live in `kits/_base/agents/`. They are invoked by reading their `.md` file and following
the instructions within. Agents never run themselves — CLAUDE.md dispatches them.

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
**Location**: `kits/kit-u1/agents/framework-setup-agent.md`
**Role**: Full project scaffold — installs dependencies, creates directory structure, generates `playwright.config.ts`, `tsconfig.json`, `package.json`.

---

## 8. Skills

Skills are reusable logic modules in `kits/_base/skills/`. Agents invoke them by reading the skill
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

## 9. Kit Configuration (kit.config.json)

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
  "templates": "./kits/kit-u1/templates",
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
| `id` | Identifies which kit is active — used to load `kits/<id>/KIT.md` |
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

## 10. Output Structure

```
outputs/
└── <projectName>/
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
    │   ├── nonbdd/                  ← Playwright spec files
    │   │   └── TC-01-successful-login.spec.ts
    │   └── bdd/                     ← Feature files + step definitions
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

## 11. Plugins & Integrations

**Location**: `configs/integrations.yml`

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

## 12. Guardrails

Defined in `configs/integrations.yml`:

| Guardrail | Value | Effect |
|---|---|---|
| `selectorScoreThreshold` | `0.70` | Selectors below this flagged in `selectors-lint.html` |
| `forbidXpathDepthGreaterThan` | `3` | XPath with > 3 levels flagged |
| `forbidSleep` | `true` | `sleep()` / `waitForTimeout()` flagged in generated code |
| `forbidDynamicIds` | `true` | `id=ember*`, `id=react-*`, `id=css-*` patterns flagged |
| `requireTagOnEveryScenario` | `true` | All BDD scenarios must have `@ui` or `@api` + `@TC-*` |
| `idempotentGeneration` | `true` | Generation commands never overwrite files without confirmation |

---

## 13. Locator Strategy Tiers

| Tier | Strategy | Trigger | Precision | Score cap |
|---|---|---|---|---|
| 1 | `playwright-mcp` | `--url` provided | Highest — live DOM | None |
| 2 | `dom-based` | `--dom` provided | High — static DOM | None |
| 3 | `ai-guided` | Neither provided | Moderate — inferred | 0.75 |

**Tier 2 multi-page flow**: `--dom <directory>` scans all `.html` files, processes each independently, and merges all page blocks into a single `selectors.json`.

---

## 14. Generated Project Config

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
