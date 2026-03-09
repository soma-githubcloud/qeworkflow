# /add-test — Add a New Test to an Existing Project

Incrementally adds a single test case (or small set) to an already-scaffolded project.
Reuses existing Page Objects wherever possible; creates new POMs only for new components.

## Arguments

| Argument | Description |
|---|---|
| `--case <text\|file>` | Test case description (inline or file path) |
| `--page <PageName>` | Existing page object to use (skip locator extraction if provided) |
| `--bdd` | Generate as BDD (feature + step) instead of plain spec |

## Execution Steps

1. **Read kit.config.json** — get `outputDir`, `projectName`, `testStyle`, `tech`

2. **Parse test case**
   - Use the `test-case-parser` skill (read `kits/_base/skills/test-case-parser.md`)
   - Extract: test ID, title, preconditions, steps, expected results

3. **Identify required pages/components**
   - From parsed test case, identify which UI pages/components are involved
   - Check `<outputDir>/<projectName>/pages/` for existing POM files matching those pages

4. **Handle missing POMs**
   - If a required POM doesn't exist:
     - If `--page` was specified but file not found: warn and stop, ask user to run `/generate-locators` first
     - Otherwise: spawn `locator-agent` in targeted mode (only for the missing component)
     - Then spawn `test-generator-agent` in `pom-only` mode for just that component

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

## Notes
- Each test case gets its own spec file (not appended to existing files)
- BDD: if a step definition already exists for a step, reuse it — don't create duplicates
