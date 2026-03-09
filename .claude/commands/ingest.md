# /ingest — Normalize and Validate Inputs

Classifies, validates, and structures all user-provided inputs before any generation begins.
Produces `intake.summary.json` — the single source of truth consumed by all other agents.

Automatically called as **Step 0** inside `/generate-tests`. Also run standalone to preview
how inputs will be classified before committing to full generation.

## Arguments

| Argument | Description |
|---|---|
| `--url <url>` | Application URL (triggers playwright-mcp strategy) |
| `--dom <path>` | Path to HTML DOM snapshot file |
| `--cases <path\|text>` | Test case descriptions — file path or inline text |

## Execution Steps

1. **Read kit.config.json** — get `outputDir`, `projectName`

2. **Classify each input**:
   - `--url` provided and starts with `http` → `appUrl = <url>`
   - `--dom` provided → read file, confirm it contains HTML markup → `domFile = <path>`
   - `--cases` provided → determine if file path or inline text; if file, read it
   - If neither `--url` nor `--dom` provided → warn: "No locator source. Strategy will be AI-guided — locators will be inferred and must be manually verified."

3. **Determine locator strategy** (priority order):
   ```
   appUrl present    → strategy = "playwright-mcp"
   domFile present   → strategy = "dom-based"
   neither           → strategy = "ai-guided"
   ```

4. **Spawn intake-agent** (read `kits/_base/agents/intake-agent.md`)
   - Pass: `appUrl`, `domFile`, `testCasesRaw`, `strategy`, `kitConfig`
   - Agent parses test cases using `test-case-parser` skill
   - Agent writes `intake.summary.json`

5. **Display summary to user**:
   ```
   Input Summary
   ─────────────────────────────────────
   App URL:        https://demo.app
   DOM file:       (none)
   Test cases:     3 parsed (TC-001, TC-002, TC-003)
   Strategy:       playwright-mcp (Tier 1)
   Warnings:       (none)
   Output:         outputs/my-project/intake.summary.json
   ```

6. If `warnings` array is non-empty, show each warning prominently and pause:
   - "Strategy is ai-guided — locators will be inferred. Proceed? (yes/no)"
   - If user says no: stop and guide them to provide a URL or DOM file

## Output: `intake.summary.json`
```json
{
  "appUrl": "https://demo.app",
  "domFile": null,
  "strategy": "playwright-mcp",
  "testCases": [
    {
      "id": "TC-001",
      "title": "Login with valid credentials",
      "tags": ["smoke"],
      "preconditions": ["User is not logged in"],
      "steps": [
        { "order": 1, "action": "navigate", "target": "login page", "value": null },
        { "order": 2, "action": "fill",     "target": "email field",    "value": "user@test.com" },
        { "order": 3, "action": "fill",     "target": "password field", "value": "Test@123" },
        { "order": 4, "action": "click",    "target": "login button",   "value": null }
      ],
      "expectedResults": ["User is redirected to dashboard"],
      "pages": ["LoginPage"]
    }
  ],
  "warnings": [],
  "generatedAt": "2026-03-03T00:00:00.000Z",
  "kitId": "kit-u1"
}
```
