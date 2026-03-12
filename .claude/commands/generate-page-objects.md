# /generate-page-objects — Generate Page Object Model Files

Converts `locators.json` into typed Page Object Model files using the active kit's templates.
Run this after `/generate-locators` or when you have an existing `locators.json`.

## Arguments

| Argument | Description |
|---|---|
| `--locators <path>` | Override locators file path (default: `<outputDir>/<projectName>/locators.json`) |
| `--overwrite` | Overwrite existing POM files (default: skip existing) |

## Execution Steps

1. **Read kit.config.json** — get `outputDir`, `projectName`, `templates`, `tech`

2. **Load locators**
   - Read `<outputDir>/<projectName>/locators.json` (or `--locators` override)
   - If file doesn't exist: "locators.json not found. Run `/generate-locators` first."
   - Validate structure: must have at minimum `{ componentName: { locatorName: string } }`

3. **Check for existing POMs**
   - List files in `<outputDir>/<projectName>/pages/`
   - If `--overwrite` not set, skip components whose POM file already exists
   - Inform the user which files will be created vs skipped

4. **Spawn test-generator-agent in POM-only mode** (read `tc-to-automate/kits/_base/agents/test-generator-agent.md`)
   - Pass: `mode: "pom-only"`, `locatorsFile`, `kitConfig`, `overwrite`
   - Agent uses `tc-to-automate/kits/<kit-id>/templates/page-object.ts.tmpl` (or equivalent for the kit)

5. **Validate**
   - Run `npx tsc --noEmit` (TypeScript kits) or `mvn compile -q` (Java kits)
   - Fix any import or type errors before returning

6. **Summary**
   - List all created/updated POM files with line counts
   - Suggest: "Run `/generate-automation-tests` to create test specs using these page objects."
