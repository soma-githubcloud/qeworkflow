# /add-test — Add a New Test to an Existing Project

Incrementally adds a single test case (or small set) to an already-scaffolded project.
Reuses existing Page Objects wherever possible; creates new POMs only for new components.

## Arguments

| Argument | Description |
|---|---|
| `--case <text\|file>` | Test case description (inline or file path) |
| `--page <PageName>` | Existing page object to use (skip locator extraction if provided) |
| `--url <url>` | Live app URL — triggers Tier 1 (Playwright MCP) locator extraction |
| `--dom <path>` | HTML snapshot file or directory — triggers Tier 2 locator extraction |
| `--bdd` | Generate as BDD (feature + step) instead of plain spec |

### Locator Strategy (resolved automatically)

| Condition | Tier | Strategy |
|-----------|------|----------|
| `--page` provided and POM exists | — | Skip locator extraction entirely |
| `--url` provided | 1 | Playwright MCP (highest confidence) |
| `--dom` provided, no `--url` | 2 | DOM snapshot extraction |
| Neither `--url` nor `--dom` | 3 | AI-guided (capped at 0.75 — must verify against app) |

## Execution Steps

1. **Read kit.config.json** — get `outputDir`, `projectName`, `testStyle`, `tech`

2. **Parse test case**
   - Use the `test-case-parser` skill (read `tc-to-automate/kits/_base/skills/test-case-parser.md`)
   - Extract: test ID, title, preconditions, steps, expected results

3. **Identify required pages/components**
   - From parsed test case, identify which UI pages/components are involved
   - Check `<outputDir>/<projectName>/pages/` for existing POM files matching those pages

4. **Handle missing POMs**
   - If a required POM doesn't exist:
     - If `--page` was specified but file not found: warn and stop, ask user to run `/generate-locators` first
     - Otherwise: resolve locator strategy from arguments (see table above), then spawn `locator-agent` in targeted mode:
       - `--url` → pass `strategy: "playwright-mcp"`, `url: <value>`
       - `--dom` → pass `strategy: "dom-snapshot"`, `domFiles: <resolved files>`
       - neither → pass `strategy: "ai-guided"` (warn user: scores capped at 0.75)
     - Then spawn `pom-agent` in `pom-only` mode for just that component

5. **Generate the new test artifact**
   - Spawn `test-generator-agent` in `spec-only` mode (or BDD mode if `--bdd`)
   - Pass: parsed test case, existing POM paths, `mode: "spec-only"`, `bdd: true/false`
   - The agent generates a NEW spec file (never modifies existing spec files)

6. **Validate**
   - Run `npx tsc --noEmit` / `mvn compile -q` / `pytest --collect-only`
   - Fix compile errors before completing

7. **Summary**
   - List newly created file(s)
   - If any existing POMs were reused, confirm which ones

## Examples

```bash
# Tier 1 — Playwright MCP (live URL, highest confidence)
/add-test --case "TC-012: Add item to wishlist" --url https://your-app.com

# Tier 2 — DOM snapshot
/add-test --case "TC-013: Filter by price range" --dom inputs/snapshots/product-list.html

# Tier 1 + reuse existing POM (no locator extraction at all)
/add-test --case "TC-014: Remove item from cart" --page CartPage --url https://your-app.com

# BDD output with live URL
/add-test --case "TC-015: Guest checkout flow" --url https://your-app.com --bdd

# Tier 3 fallback (no URL or DOM — AI-guided, verify locators manually)
/add-test --case "TC-016: Sort products by rating"
```

## Notes
- Each test case gets its own spec file (not appended to existing files)
- BDD: if a step definition already exists for a step, reuse it — don't create duplicates
- `--url` and `--dom` are ignored when `--page` is provided and the POM already exists
- Tier 3 (AI-guided) locators are always flagged with a warning and score ≤ 0.75
