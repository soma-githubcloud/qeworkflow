# Automation Kit — User Guide

> **Who this guide is for**: Anyone cloning this repository and using the kit for the first time,
> including users who have never installed Node.js, Playwright, or Claude Code.

---

## Table of Contents

1. [What Is This Kit?](#1-what-is-this-kit)
2. [Prerequisites](#2-prerequisites)
3. [Clone & First-Time Setup](#3-clone--first-time-setup)
4. [Configure Claude Code](#4-configure-claude-code)
5. [Project Structure at a Glance](#5-project-structure-at-a-glance)
6. [Quick-Start: Your First Test in 5 Minutes](#6-quick-start-your-first-test-in-5-minutes)
7. [Command Reference](#7-command-reference)
8. [Working with DOM Snapshots](#8-working-with-dom-snapshots)
9. [Working with Test Cases](#9-working-with-test-cases)
10. [Locator Strategy — Which to Choose](#10-locator-strategy--which-to-choose)
11. [Generating BDD Tests](#11-generating-bdd-tests)
12. [Running Tests & Viewing Reports](#12-running-tests--viewing-reports)
13. [Configuring Plugins & Pushing to GitHub](#13-configuring-plugins)
14. [Switching Kits](#14-switching-kits)
15. [Troubleshooting & FAQ](#15-troubleshooting--faq)

---

## 1. What Is This Kit?

The Automation Kit is an AI-powered test automation scaffold that turns your test cases and
application pages into production-ready Playwright TypeScript tests — including Page Object
Models, spec files, BDD feature files, and CI pipelines(TBD).

You describe **what** to test. The kit figures out **how** to test it.

**What gets generated for you:**
- `src/pages/` — Page Object Model (POM) classes with correct locators
- `tests/nonbdd/` — Playwright `test()` spec files
- `tests/bdd/` — Gherkin `.feature` files + step definitions (optional)
- `playwright.config.ts` — Ready-to-run config with Allure reporting
- `.github/workflows/ui.yml` — GitHub Actions CI pipeline (optional)

---

## 2. Prerequisites

Install these tools **before** cloning. The table below includes direct download links and the
minimum versions required.

| Tool | Minimum Version | Why It's Needed | Install |
|---|---|---|---|
| **Node.js** | 18.x LTS | Runs Playwright and all TypeScript tooling | [nodejs.org](https://nodejs.org) — download "LTS" |
| **npm** | 9.x (bundled with Node.js) | Package manager | Comes with Node.js |
| **Git** | 2.x | Clone the repository | [git-scm.com](https://git-scm.com) |
| **Claude Code CLI** | Latest | AI-powered generation engine | See §4 below |
| **VS Code** (recommended) | Any recent | Editor with Claude Code extension | [code.visualstudio.com](https://code.visualstudio.com) |

### Verify your installs

Open a terminal and run:

```bash
node --version    # should print v18.x or higher
npm --version     # should print 9.x or higher
git --version     # should print 2.x or higher
```

If any command fails, install the missing tool before continuing.

---

## 3. Clone & First-Time Setup

### Step 1 — Clone the repository

```bash
git clone <repository-url> automation-kit
cd automation-kit
```

> Replace `<repository-url>` with the actual URL provided by your team.

### Step 2 — Install root-level dependencies (if any)

The kit itself has no npm dependencies at the root level. Dependencies are installed **inside**
each generated project. Skip this step unless a `package.json` exists at the root.

### Step 3 — Verify the kit configuration

```bash
cat kit.config.json
```

You should see `"id": "kit-u1"` and `"projectName": "playwright-greenfieldproject"`. This is the
active kit. If you need a different kit, see §14.

### Step 4 — Run `/setup-kit` ⬅ REQUIRED before first use

> **This step is mandatory.** `/setup-kit` scaffolds the output project folder, installs
> Playwright and all dependencies, and validates the kit configuration. Nothing else works
> until this has been run at least once on a new machine or a fresh clone.

Open Claude Code (see §4) and type in the chat:

```
/setup-kit
```

What it does:
- Creates `outputs/playwright-greenfieldproject/` with the full folder structure
- Writes `playwright.config.ts`, `package.json`, `tsconfig.json`
- Runs `npm install` and `npx playwright install chromium`
- Confirms the kit is ready: `"Kit kit-u1 scaffolded successfully"`

You only need to run `/setup-kit` **once per machine / per fresh clone**. After that, go
straight to `/generate-tests` for all subsequent work.

### Step 5 — Check the snapshots directory (optional)

If your team has provided HTML snapshots of the application:

```bash
ls snapshots/
```

You should see `.html` files named after the pages they represent (e.g., `loginPage.html`,
`homePage.html`). If this directory is empty or does not exist, you will use AI-guided mode
instead (see §10).

---

## 4. Configure Claude Code

Claude Code is the AI engine that powers all generation commands.

### Install Claude Code CLI

```bash
npm install -g @anthropic/claude-code
```

### Authenticate

```bash
claude login
```

Follow the browser prompt to authenticate with your Anthropic account. A valid API key is
required. If your team uses a shared API key, set it as an environment variable instead:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### Open the project in Claude Code

```bash
# Option A — VS Code extension (recommended)
code .
# Then open the Claude Code panel from the sidebar or press Ctrl+Shift+P → "Claude Code"

# Option B — CLI mode
claude
```

### Confirm the kit is active

At the start of any Claude Code session, type anything and look for the startup confirmation line:

```
Active kit: kit-u1 | playwright | nonbdd
```

If you do not see this line, ask Claude: _"What is the active kit?"_

---

## 5. Project Structure at a Glance

```
automation-kit/
├── kit.config.json               ← Active kit configuration (change this to switch kits)
├── CLAUDE.md                     ← Orchestrator rules (do not edit unless extending the kit)
├── kits/
│   ├── _base/
│   │   └── agents/               ← Shared AI agents (intake, locator, pom, testgen, bdd, etc.)
│   └── kit-u1/                   ← Playwright TypeScript kit
│       ├── KIT.md                ← Kit-specific naming rules
│       └── templates/            ← Code templates
├── snapshots/                    ← PUT YOUR HTML SNAPSHOTS HERE
├── configs/
│   └── integrations.yml          ← Plugin configuration (Jira, GitHub, Allure Server)
├── outputs/
│   └── playwright-greenfieldproject/   ← All generated files go here
│       ├── intake.summary.json   ← Input classification (auto-generated)
│       ├── selectors.json        ← Selector map (auto-generated)
│       ├── src/pages/            ← Page Object Model classes
│       ├── tests/nonbdd/         ← Playwright spec files
│       ├── tests/bdd/            ← Feature files + step definitions
│       └── playwright.config.ts  ← Test runner config
└── docs/
    ├── TECHNICAL-REFERENCE.md    ← Deep technical reference for kit developers
    └── USER-GUIDE.md             ← This file
```

---

## 6. Quick-Start: Your First Test in 5 Minutes

This walkthrough generates a login test for `practice.expandtesting.com` — no DOM snapshot
required (AI-guided mode).

> **Before you begin**: If this is a fresh clone or a new machine, you must run `/setup-kit`
> first (§3 Step 4). If you have already done that, skip straight to Step 2 below.

### Step 1 — Run `/setup-kit` (first time only)

In the Claude Code chat:

```
/setup-kit
```

Wait for: `"Kit kit-u1 scaffolded successfully"` — then continue.

### Step 2 — Open Claude Code and confirm the kit

```
Active kit: kit-u1 | playwright | nonbdd
```

### Step 3 — Run the generate command

In the Claude Code chat, type:

```
/generate-tests --cases "TC-01: Successful Login (Happy Path)
Area: Authentication
Preconditions: User account is available at practice.expandtesting.com
Steps:
  1. Navigate to the login page
  2. Enter valid username: practice
  3. Enter valid password: SuperSecretPassword!
  4. Click Login
  Assertions:
  - URL changes to /secure
  - A message containing 'Logged in as' is visible
  - No error messages are displayed"
```

### Step 4 — Watch the pipeline run

Claude Code will:
1. Parse your test case (intake-agent)
2. Infer locators from the step text (AI-guided, scores ≤ 0.75)
3. Generate `LoginPage.ts` and `SecureAreaPage.ts`
4. Generate `TC-01-successful-login.spec.ts`
5. Configure reporting

You will see a summary table at the end listing all created files.

### Step 5 — Install generated project dependencies

```bash
cd outputs/playwright-greenfieldproject
npm install
npx playwright install chromium
```

### Step 6 — Run the tests

```bash
npx playwright test
```

### Step 7 — View the report

```bash
npx playwright show-report
```

A browser window opens showing the Playwright HTML report.

---

## 7. Command Reference

All commands are typed in the Claude Code chat panel. Commands starting with `/` are **slash
commands** that trigger automated generation pipelines.

### `/generate-tests` — Main generation command

Generates the full artifact set: locators → POMs → specs → (optionally) BDD.

```bash
# AI-guided (no DOM, no live URL) — test cases only
/generate-tests --cases "TC-01: ..."

# DOM snapshot(s) — pass a directory, all .html files are discovered
/generate-tests --dom snapshots/ --cases "TC-01: ..."

# Single DOM file
/generate-tests --dom snapshots/loginPage.html --cases "TC-01: ..."

# Live URL (most accurate locators — requires browser MCP)
/generate-tests --url https://myapp.com/login --cases "TC-01: ..."

# Include BDD feature + step files in addition to specs
/generate-tests --dom snapshots/ --cases "TC-01: ..." --bdd

# Preview files without writing anything
/generate-tests --dom snapshots/ --cases "TC-01: ..." --dry-run

# Only generate smoke-tagged tests
/generate-tests --cases path/to/testcases.md --tags smoke
```

**What gets generated:**

| Input | Locator Quality | Output |
|---|---|---|
| `--url` only | Highest (live DOM, scores up to 1.0) | POMs + specs + BDD |
| `--dom` + `--cases` | High (real HTML, scores up to 1.0) | POMs + specs only |
| `--dom` + `--cases` + `--bdd` | High | POMs + specs + BDD |
| `--cases` only | Inferred (AI, scores ≤ 0.75) | POMs + specs only |

---

### `/run-smoke` — Run smoke tests

```bash
/run-smoke
/run-smoke --headed          # Run with browser visible
/run-smoke --browser firefox # Use Firefox instead of Chromium
/run-smoke --report-only     # Regenerate reports without re-running tests
```

---

### `/gen-bdd` — Generate BDD artifacts only

Use this when you already have POMs and specs but want to add BDD coverage.

```bash
/gen-bdd --cases "TC-01: ..."
```

---

### `/add-test` — Add a test to an existing project

Adds new specs without regenerating the whole project.

```bash
/add-test --cases "TC-02: Failed Login with wrong password"
```

---

### `/generate-locators` — Extract locators only

Produces or updates `selectors.json` without touching POMs or specs.

```bash
/generate-locators --dom snapshots/
/generate-locators --url https://myapp.com/login
```

---

### `/generate-page-objects` — Regenerate POMs from existing selectors

Use after manually editing `selectors.json`.

```bash
/generate-page-objects
```

---

### `/setup-reporting` — Configure Allure reporting

Adds Allure to `playwright.config.ts` if not already present.

```bash
/setup-reporting
```

---

### `/setup-ci` — Generate CI pipeline

Creates `.github/workflows/ui.yml` for GitHub Actions.

```bash
/setup-ci
```

---

### `/validate-kit` — Validate the generated project

Runs TypeScript compile check + Playwright test discovery.

```bash
/validate-kit
```

---

### `/ingest` — Normalize inputs only

Runs intake-agent and produces `intake.summary.json` without generating any code.

```bash
/ingest --cases "TC-01: ..."
/ingest --dom snapshots/ --cases "TC-01: ..."
```

---

### `/setup-kit` — Initialize a new kit

Only needed when switching to a different tech stack (see §14).

```bash
/setup-kit
```

---

## 8. Working with DOM Snapshots

DOM snapshots are `.html` files saved from the browser. They give the AI real element IDs,
aria labels, and data attributes — producing much more reliable locators than AI inference.

### How to save a DOM snapshot

**Option A — Chrome DevTools:**
1. Open the page in Chrome
2. Press `F12` → Elements tab
3. Right-click `<html>` → Copy → Copy outerHTML
4. Paste into a file in `snapshots/` (e.g., `snapshots/loginPage.html`)

**Option B — Command line:**
```bash
curl -s "https://myapp.com/login" -o snapshots/loginPage.html
```

**Option C — Playwright script:**
```javascript
const { chromium } = require('@playwright/test');
const fs = require('fs');
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto('https://myapp.com/login');
fs.writeFileSync('snapshots/loginPage.html', await page.content());
await browser.close();
```

### File naming convention

Name files so the kit can infer page names automatically:

| File name | Inferred page name |
|---|---|
| `loginPage.html` | `LoginPage` |
| `login-page.html` | `LoginPage` |
| `amazon-home-page.html` | `AmazonHomePage` |
| `search_results.html` | `SearchResults` |
| `checkout.html` | `Checkout` |

The rule: strip the extension, convert kebab/snake to PascalCase.

### Multi-page flows

Put all snapshot files in `snapshots/` and pass the directory:

```bash
/generate-tests --dom snapshots/ --cases "TC-01: ..."
```

The kit discovers all `.html` files, generates one POM per page, and wires page transitions
between them based on your test case steps.

Example directory structure for a login → dashboard flow:

```
snapshots/
├── loginPage.html         → LoginPage.ts
└── dashboardPage.html     → DashboardPage.ts
```

In your test case, mention both pages in the steps:

```
Steps:
1. Navigate to the login page
2. Enter username and password
3. Click Login                     ← transition happens here
4. Assert dashboard URL is /dashboard
5. Assert welcome message is visible
```

The kit detects the transition at step 3 and generates `waitForURL()` at that point.

---

## 9. Working with Test Cases

### Inline test cases

Pass test cases directly in the command:

```bash
/generate-tests --cases "TC-01: Login with valid credentials
Steps:
  1. Navigate to /login
  2. Enter valid username
  3. Enter valid password
  4. Click Submit
  Assertions:
  - URL is /dashboard
  - Welcome message is visible"
```

### File-based test cases

Save test cases in a markdown file and pass the path:

```bash
/generate-tests --cases tests/test-cases.md
```

**File format (`tests/test-cases.md`):**

```markdown
# TC-01: Successful Login (Happy Path)
Tags: smoke
Preconditions: User account exists at https://myapp.com

## Steps
1. Navigate to the login page at /login
2. Fill username with valid credentials
3. Fill password with valid credentials
4. Click the Login button

## Expected Results
- URL changes to /dashboard
- A success banner displays "Welcome back"
- No error messages are shown

---

# TC-02: Login with Wrong Password (Negative)
Tags: regression, negative
Preconditions: User account exists

## Steps
1. Navigate to the login page
2. Fill username with valid credentials
3. Fill password with an incorrect value
4. Click Login

## Expected Results
- URL remains /login
- An error message "Invalid credentials" is displayed
```

### Multiple test cases in one run

List all test cases in the same markdown file — the kit generates one spec per test case.

```bash
/generate-tests --dom snapshots/ --cases tests/test-cases.md --tags smoke
```

---

## 10. Locator Strategy — Which to Choose

The kit uses three strategies in priority order. Choose the highest tier available.

### Tier 1 — Live URL (most accurate)

**When to use**: You have a running application accessible via URL.

```bash
/generate-tests --url https://staging.myapp.com/login --cases "TC-01: ..."
```

- Playwright MCP opens a real browser, extracts actual DOM elements
- Locator scores up to 1.0
- Generates both spec and BDD by default

**Requirement**: Browser MCP must be enabled (it is by default in `.claude/settings.json`).

### Tier 2 — DOM Snapshot (high accuracy)

**When to use**: You have `.html` files from the application but no live URL.

```bash
/generate-tests --dom snapshots/ --cases "TC-01: ..."
```

- Kit parses real HTML to extract `data-testid`, `aria-label`, `id`, `name`, `role`
- Locator scores up to 1.0 for definitive matches
- Generates spec files only by default (add `--bdd` for BDD)

### Tier 3 — AI-Guided (inferred, lowest accuracy)

**When to use**: No URL and no snapshots — test cases only.

```bash
/generate-tests --cases "TC-01: ..."
```

- AI infers selectors from step wording (e.g., "Enter username" → `getByLabel('Username')`)
- Locator scores capped at 0.75 — **always verify before running in CI**
- A warning is added to `selectors.json` and shown in the summary
- Run `npx playwright test --headed` to visually confirm locators work

**Tip**: After running tests once in AI-guided mode, save the real HTML from the browser and
regenerate with `--dom snapshots/` to get confirmed locators.

---

## 11. Generating BDD Tests

BDD (Behavior-Driven Development) artifacts consist of:
- A **Gherkin `.feature` file** with `Given/When/Then` scenarios
- A **step definitions file** (`.steps.ts`) that maps steps to POM actions

### When BDD is generated automatically

| Strategy | Default output | Override |
|---|---|---|
| `--url` (live) | spec + BDD | n/a |
| `--dom` (snapshot) | spec only | add `--bdd` flag |
| `--cases` only (AI) | spec only | add `--bdd` flag |

### Force BDD generation

```bash
/generate-tests --dom snapshots/ --cases "TC-01: ..." --bdd
```

### BDD only (no spec)

```bash
/gen-bdd --cases "TC-01: ..."
```

### BDD file output locations

```
outputs/playwright-greenfieldproject/
├── tests/bdd/
│   ├── login.feature           ← Gherkin scenarios
│   └── login.steps.ts          ← Step definitions (imports from POMs)
```

### Running BDD tests

```bash
cd outputs/playwright-greenfieldproject
npx cucumber-js
```

---

## 12. Running Tests & Viewing Reports

### Install dependencies (first time only)

```bash
cd outputs/playwright-greenfieldproject
npm install
npx playwright install chromium    # installs Chromium browser
```

### Run all tests

```bash
npx playwright test
```

### Run smoke tests only

```bash
npx playwright test --grep "@smoke"
# or use the kit command:
/run-smoke
```

### Run a specific spec file

```bash
npx playwright test tests/nonbdd/TC-01-successful-login.spec.ts
```

### Run in headed mode (browser visible)

```bash
npx playwright test --headed
```

### Run in a specific browser

```bash
npx playwright test --project=firefox
npx playwright test --project=webkit
```

### View Playwright HTML report

```bash
npx playwright show-report
```

Opens a browser at `http://localhost:9323` showing pass/fail results with screenshots and traces.

### View Allure report

```bash
# Generate the report from results
npx allure generate --output allure-report allure-results

# Open in browser
npx allure open allure-report
```

> **Note**: You need Java 8+ installed to run Allure CLI. Install from
> [allure.qatools.ru](https://allure.qatools.ru/docs/getting-started/installation/).
> Alternatively, install the npm wrapper: `npm install -g allure-commandline`

### Re-run failed tests only

```bash
npx playwright test --last-failed
```

---

## 13. Configuring Plugins

Plugins extend the kit with integrations for Jira, GitHub, and Allure Server.

### Enable a plugin

Edit `configs/integrations.yml`:

```yaml
plugins:
  github:
    enabled: true
    repo: "org/repo-name"
    branch: "main"

  jira:
    enabled: true
    host: "https://yourorg.atlassian.net"
    project: "QA"

  allureServer:
    enabled: false
    host: "http://localhost:5050"
```

### What each plugin does

| Plugin | Trigger | What it does |
|---|---|---|
| `github` | After any generation command | Commits generated files + optionally opens a PR |
| `jira` | Before generation + after `/run-smoke` | Fetches test cases from Jira; pushes results back |
| `allureServer` | After `/run-smoke` | Publishes Allure results to a shared Allure server |

### GitHub plugin — push generated project to a repo

Use this to push the generated `outputs/<project>/` folder to a GitHub repository without
running any git commands manually.

#### Step 1 — Generate a GitHub Personal Access Token (PAT)

1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token**
3. Select scope: **repo** (full control of private repositories)
4. Copy the token — you will not see it again

#### Step 2 — Set environment variables

Open a **Command Prompt or PowerShell** window and set these before launching Claude Code:

**Windows Command Prompt:**
```cmd
set GITHUB_OWNER=your-github-username
set GITHUB_REPO=your-repo-name
set GITHUB_TOKEN=ghp_your_token_here
```

**Windows PowerShell:**
```powershell
$env:GITHUB_OWNER = "your-github-username"
$env:GITHUB_REPO  = "your-repo-name"
$env:GITHUB_TOKEN = "ghp_your_token_here"
```

**macOS / Linux:**
```bash
export GITHUB_OWNER="your-github-username"
export GITHUB_REPO="your-repo-name"
export GITHUB_TOKEN="ghp_your_token_here"
```

> Set these in the **same terminal session** you will use to launch Claude Code.
> Environment variables set in one window are NOT inherited by other windows.

#### Step 3 — Enable the plugin in `configs/integrations.yml`

```yaml
github:
  enabled: true                          # change false → true
  repoOwner: ${GITHUB_OWNER}             # do NOT hardcode — uses env var
  repoName:  ${GITHUB_REPO}              # do NOT hardcode — uses env var
  token:     ${GITHUB_TOKEN}             # do NOT hardcode — uses env var
  createPr:  true                        # set false if you just want a push, no PR
  prBaseBranch: main
  commitMessage: "feat: generated kit-u1 test artifacts"
```

#### Step 4 — Launch Claude Code from the same terminal

```cmd
cd c:\Users\<you>\automation-kit
claude
```

#### Step 5 — Trigger the push in the Claude Code chat

```
Please follow plugins/github.md to push the playwright-greenfieldproject to GitHub.
```

The plugin will:
1. `git init` inside `outputs/playwright-greenfieldproject/`
2. Stage all generated files (`git add .`)
3. Commit with the configured commit message
4. Add your GitHub repo as remote origin
5. Push to `main`
6. Create a Pull Request if `createPr: true`

#### Avoid permission prompts

By default Claude Code asks for approval on every git and shell command. To pre-approve
all GitHub plugin operations, the following entries must be present in `.claude/settings.json`
under `permissions.allow` (they are already added if you are using this kit):

```json
"Bash(git init*)",
"Bash(git add *)",
"Bash(git commit *)",
"Bash(git remote *)",
"Bash(git branch *)",
"Bash(git push *)",
"Bash(gh pr create*)",
"mcp__git__git_push",
"mcp__git__git_remote",
"mcp__git__git_branch"
```

> Force-push commands (`git push --force`, `git push -f`) remain in the **deny** list and
> will always prompt — this is intentional for safety.

#### What gets pushed / excluded

| Pushed | Excluded (add to `.gitignore`) |
|---|---|
| `src/pages/*.ts` | `node_modules/` |
| `tests/nonbdd/*.spec.ts` | `allure-results/`, `allure-report/` |
| `tests/bdd/` (if generated) | `playwright-report/`, `test-results/` |
| `playwright.config.ts` | `cucumber.json` |
| `package.json`, `tsconfig.json` | |
| `selectors.json`, `intake.summary.json` | |

#### After cloning the pushed repo

Anyone who clones the repo runs:

```bash
npm install
npx playwright install chromium
npx playwright test
```

---

## 14. Switching Kits

The active kit is set in `kit.config.json` at the root. Kit-u1 (Playwright TypeScript) is
the default. Other kits target different tech stacks.

| Kit ID | Stack |
|---|---|
| `kit-u1` | Playwright + TypeScript (current) |
| `kit-u2` | Playwright + TypeScript + Cucumber (BDD-first) |
| `kit-u3` | Selenium + Java + TestNG |
| `kit-u4` | Selenium + Java + Cucumber |
| `kit-a2` | pytest + requests (API) |
| `kit-h1` | Playwright + TypeScript + Hybrid (API + UI) |
| `kit-h3` | Selenium + Python + Hybrid |

### Switch to a different kit

1. Edit `kit.config.json` — change `"id"` to the target kit
2. Run `/setup-kit` to scaffold the new kit's project structure
3. Start a new Claude Code session (the startup protocol reads the new kit config)

```bash
# Example: switch to Selenium Java
# In kit.config.json:
{
  "id": "kit-u3",
  ...
}
```

Then in Claude Code:

```
/setup-kit
```

---

## 15. Troubleshooting & FAQ

### "Active kit" line does not appear at startup

**Cause**: Claude Code did not read `kit.config.json` on startup.
**Fix**: Ask Claude: _"Please read kit.config.json and confirm the active kit."_

---

### `npx playwright test` finds 0 tests

**Cause**: The generated project directory has no tests, or `playwright.config.ts` points to
the wrong `testDir`.

**Fix**:
```bash
cd outputs/playwright-greenfieldproject
npx playwright test --list    # lists all discovered tests
```

If the list is empty, check `testDir` in `playwright.config.ts` and ensure spec files exist in
`tests/nonbdd/`.

---

### Tests fail with "locator not found" or timeout errors

**Cause**: AI-guided selectors were inferred and may not match the real DOM.

**Fix**:
1. Run in headed mode: `npx playwright test --headed`
2. Take DOM snapshots of the failing pages (see §8)
3. Re-run: `/generate-tests --dom snapshots/ --cases "TC-01: ..."`
4. The kit will regenerate POMs with confirmed selectors

---

### `npx playwright install` fails behind a corporate proxy

**Fix**:
```bash
HTTPS_PROXY=http://proxy.corp.com:8080 npx playwright install chromium
```

---

### Allure command not found

**Fix** — install the npm allure wrapper:
```bash
npm install -g allure-commandline
```

Or use Java allure CLI:
```bash
# macOS
brew install allure

# Windows (scoop)
scoop install allure
```

---

### Generated selectors have score 0.75 — what does this mean?

Score 0.75 means the locator was **AI-inferred** from step text, not extracted from a real DOM.
It will work in many cases but is not guaranteed. Always:
1. Run tests in `--headed` mode first to visually confirm
2. Replace with DOM-based selectors when possible (see §8 and §10)

The score is visible in `selectors.json` and flagged in `selectors-lint.html`.

---

### I edited a POM file manually — how do I regenerate without losing my changes?

The kit asks before overwriting existing files. If you want to regenerate cleanly:
1. Delete the specific `.ts` file
2. Re-run `/generate-page-objects`

If you want to keep manual changes: **do not delete** the file. Run `/generate-tests --dry-run`
first to preview what would change.

---

### How do I add a test to an existing project without regenerating everything?

```bash
/add-test --cases "TC-05: Password reset flow
Steps:
  1. Navigate to /forgot-password
  2. Enter email address
  3. Click Send Reset Link
  Assertions:
  - A confirmation message appears"
```

This adds a new spec file using existing POMs. No existing files are touched.

---

### Can I use the kit without VS Code?

Yes. Run Claude Code in terminal mode:

```bash
cd automation-kit
claude
```

All slash commands work identically in CLI mode.

---

### Where are the generated files?

All output goes to:
```
outputs/<projectName>/
```

The project name is set in `kit.config.json` under `"projectName"`. Default is
`playwright-greenfieldproject`.

---

### How do I reset the project and start fresh?

```bash
# Delete all generated files for the current project
rm -rf outputs/playwright-greenfieldproject/

# Then re-run any /generate-tests command
```

The kit will regenerate everything from scratch.

---

*Last updated: 2026-03-09 | Kit version: kit-u1 | Playwright ^1.44.0*
