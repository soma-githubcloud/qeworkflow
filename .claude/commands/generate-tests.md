# /generate-tests — Generate Full Test Automation Artifacts

**Primary workflow command.** Generates locators, Page Objects, test specs, and (optionally) BDD feature files + step definitions from user-supplied inputs.

## Arguments (parse from $ARGUMENTS or user message)

| Argument | Description |
|---|---|
| `--url <url>` | Live application URL for Playwright MCP locator extraction |
| `--cases <path\|inline>` | Test case descriptions (file path or inline text) |
| `--dom <path>` | **Directory** containing `.html` snapshot files (all files auto-discovered), **or** a single `.html` file path. Directory is the preferred approach for multi-page flows. |
| `--bdd` | Generate BDD artifacts (feature + steps) in addition to plain spec |
| `--project <name>` | Override project name from kit.config.json |
| `--dry-run` | Preview planned files without writing any |
| `--tags <tag,...>` | Only generate tests whose tags match (e.g. `--tags smoke`) |

### `--dom` resolution rules
```
If --dom value ends with .html → single file
  domFiles = [{ pageName: inferPageName(path), file: path }]

If --dom value is a directory path → scan for all *.html files
  domFiles = glob("<path>/**/*.html").map(f => ({
    pageName: inferPageName(f),   // filename without ext, PascalCase
    file: f
  }))
  Files are sorted alphabetically. Log the discovered list to the user before proceeding.

inferPageName(filePath):
  strip directory + extension → convert kebab/snake to PascalCase
  e.g. "snapshots/amazonHome-page.html"   → "AmazonHomePage"
  e.g. "snapshots/amazonSearch-page.html" → "AmazonSearchPage"
  e.g. "snapshots/cart.html"              → "Cart"
```

### Examples
```bash
# Directory — discovers all .html files inside snapshots/
/generate-tests --dom snapshots/ --cases "TC_AMZ_001: Search for iPhone on Amazon"

# Single file (backward-compatible)
/generate-tests --dom snapshots/amazonHome-page.html --cases path/to/testcases.md

# URL-based (live browser, overrides --dom)
/generate-tests --url https://www.amazon.in --cases "TC_AMZ_001: ..."
```

---

## Execution Pipeline

### Step 0: Load context
- Read `kit.config.json` — get `id`, `tech`, `testStyle`, `outputDir`, `projectName`, `templates`
- Read `kits/<kit-id>/KIT.md` for naming conventions and kit-specific rules
- Confirm active kit to user: `Active kit: <id> | <tech.testRunner> | <testStyle>`

### Step 1: Resolve domFiles from --dom argument
Apply `--dom` resolution rules above. If a directory was given, list the discovered files to
the user: `"Found N DOM snapshots in <path>: [file1, file2, ...]"`. Pause if 0 files found.

### Step 2: Run intake-agent (MANDATORY — always runs first)
Read and execute `kits/_base/agents/intake-agent.md`. Pass:
- `appUrl`: value of `--url` (or null)
- `domFiles`: resolved array from Step 1 (or empty)
- `testCasesRaw`: value of `--cases` (inline text or file path)
- `kitConfig`: full kit.config.json contents

Output: `<outputDir>/<projectName>/intake.summary.json`

The intake-agent handles: DOM file validation, page-name inference, page transition detection
per step (`pageContext`, `transitionTo`), and coverage warnings for unmatched pages.

### Step 3: Check for existing selectors
- If `<outputDir>/<projectName>/selectors.json` exists and is non-empty, ask the user:
  `"Existing selectors.json found. Use existing selectors or regenerate?"`
- If regenerating or file missing: **run locator-agent** (`kits/_base/agents/locator-agent.md`)

  Pass from `intake.summary.json`:
  - `strategy`, `url` (if playwright-mcp), `domFiles` (if dom-based), `testCases`
  - `outputPath`: `<outputDir>/<projectName>/selectors.json`

  For a directory with N snapshots, locator-agent processes each file independently and
  merges all page blocks into one `selectors.json` (one top-level key per page).

### Step 4: Generate Page Objects
**Run pom-agent** (`kits/_base/agents/pom-agent.md`). Pass:
- `selectorsFile`: `<outputDir>/<projectName>/selectors.json`
- `intakeFile`: `<outputDir>/<projectName>/intake.summary.json`
- `kitConfig`: full kit.config.json

One POM class file is generated per page (per top-level key in `selectors.json`).

### Step 5: Generate test specs
Read `testStyle` from `intake.summary.json`:

| `testStyle` | Action |
|---|---|
| `"nonbdd"` | Run **testgen-agent** only → `kits/_base/agents/testgen-agent.md` |
| `"bdd"` | Run **bdd-agent** only → `kits/_base/agents/bdd-agent.md` |
| `"both"` | Run **both** testgen-agent and bdd-agent (sharing the same POMs) |

Each agent uses `step.pageContext` and `testCase.pageTransitions` from `intake.summary.json`
to import all required POM classes and route each step call to the correct POM instance.

### Step 6: Configure reporting
**Run reporting-agent** (`kits/_base/agents/reporting-agent.md`)
- Only if allure reporter is not already present in `playwright.config.ts`

### Step 7: Validate
- Run `npx tsc --noEmit` inside the generated project directory
- Run `npx playwright test --list` to confirm all specs are discoverable
- Fix unambiguous import path issues automatically; report anything that needs manual attention

### Step 8: Summary
Print a table of all generated files:

| File | Type | Pages covered |
|---|---|---|
| `src/pages/AmazonHomePage.ts` | POM | AmazonHomePage |
| `src/pages/AmazonSearchPage.ts` | POM | AmazonSearchPage |
| `tests/nonbdd/TC_AMZ_001-search-product.spec.ts` | Spec | AmazonHomePage → AmazonSearchPage |

---

## Important Rules
- Always run intake-agent first — it is the contract for all downstream agents
- Always generate POMs before specs — specs import from POMs
- Never put raw selectors directly in spec files
- The canonical selector file name is `selectors.json` — never `locators.json`
- If `--bdd` flag is used and `bddRunner` in kit.config.json is null, set it to `"@cucumber/cucumber"` automatically
- If `--cases` value is a file path, read the file before passing to intake-agent
- `--dry-run` prints planned file names and step counts; no files are written
