# /setup-kit — Initialize Automation Kit

Initialize a new automation project by selecting a kit and scaffolding the full project structure.

## Steps

1. **Read current state**
   - Check if `kit.config.json` already exists. If it does, show current kit and ask the user if they want to switch or re-scaffold.
   - If no config exists, proceed to kit selection.

2. **Kit selection** — Ask the user:
   - Which kit? (kit-u1, kit-u2, kit-u3, kit-u4, kit-a1, kit-a2, kit-h1, kit-h2, kit-h3)
   - Test style? (non-bdd | bdd)
   - Output directory? (default: `./generated`)
   - Project name? (used as the subdirectory under outputDir)

3. **Write kit.config.json**
   - Set `id`, `mode`, `tech`, `testStyle`, `outputDir`, `projectName`
   - Set `templates` and `agents` paths based on selected kit
   - If the selected kit has `extends`, merge from parent kit config first

4. **Load the kit's scaffolder agent**
   - Read the agent file at `kits/<kit-id>/agents/framework-setup-agent.md` (greenfield kits)
   - OR `kits/<kit-id>/agents/framework-analyzer-agent.md` (existing framework kits)
   - Follow the agent instructions precisely

5. **Verify scaffold**
   - For TypeScript kits: run `npx tsc --noEmit` in the output directory
   - For Java kits: run `mvn compile -q`
   - For Python kits: run `pytest --collect-only -q`
   - Report success or any errors to the user

6. **Confirm completion**
   - List all created files and directories
   - Show the user the next suggested command: `/generate-tests`

## Notes
- Never overwrite existing source files without explicit user confirmation
- Always create `.gitignore` with `node_modules/`, `allure-results/`, `playwright-report/`
- Pin dependency versions; do not use `latest` tags in package.json
