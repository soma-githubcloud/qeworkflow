# /generate-locators — Extract or Generate Locators Only

Runs only the locator extraction phase and writes `locators.json`. Useful when you want to review
and approve locators before generating full test artifacts.

## Arguments

| Argument | Description |
|---|---|
| `--url <url>` | Live application URL (triggers Playwright MCP — highest accuracy) |
| `--dom <path>` | Path to HTML DOM snapshot file |
| `--components <names>` | Comma-separated component/page names to focus on (optional) |

## Execution Steps

1. **Read kit.config.json** — get `outputDir`, `projectName`, `locatorStrategies`

2. **Determine strategy**
   - `--url` provided → `"playwright-mcp"` (Tier 1)
   - `--dom` provided → `"dom-based"` (Tier 2)
   - Neither provided → `"ai-guided"` (Tier 3, will ask for test case context)

3. **Spawn locator-agent** (read `tc-to-automate/kits/_base/agents/locator-agent.md`)
   - Pass: `strategy`, `url`, `domFile`, `components` (optional focus list)
   - Agent writes `<outputDir>/<projectName>/locators.json`

4. **Display result**
   - Print the generated `locators.json` content to the user in formatted JSON
   - Show confidence level per locator (confirmed / inferred)
   - If any locators are `"confidence": "inferred"`, warn: "These locators were AI-generated and must be verified against your application."

5. **Offer next step**
   - "Run `/generate-page-objects` to convert these locators into Page Object files."

## Notes
- This command is idempotent. Re-running with the same URL replaces `locators.json`.
- Use `--components` to limit extraction to specific pages (e.g., `--components "LoginPage,DashboardPage"`)
