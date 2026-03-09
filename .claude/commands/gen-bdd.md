# /gen-bdd — Generate BDD Artifacts Only

Generates `.feature` files and matching step definition files from test case inputs.
Does NOT re-run locator extraction or POM generation — runs the BDD pipeline only.
Use this when you already have POMs and selectors, or when you want to author Gherkin first.

## Arguments

| Argument | Description |
|---|---|
| `--cases <path\|text>` | Test case descriptions (required) |
| `--style strict\|relaxed` | Gherkin style (default: strict) |
| `--tags <list>` | Additional tags to apply (e.g., `@smoke @regression`) |
| `--dry-run` | Print planned output without writing files |

## Style Modes

| Mode | Description |
|---|---|
| `strict` | Domain language only — no UI verbs ("click", "fill"). Steps read as business language. |
| `relaxed` | Allows UI-style wording ("I click the login button"). Useful for developer-authored tests. |

## Execution Steps

1. **Read kit.config.json** — confirm `testStyle` includes BDD support (warn if `testStyle = "non-bdd"`)

2. **Normalize inputs** — if `intake.summary.json` already exists and is recent, offer to reuse it.
   Otherwise run intake-agent inline (without writing `intake.summary.json`) to parse test cases.

3. **Spawn bdd-agent** (read `kits/_base/agents/bdd-agent.md`)
   - Pass: `testCases`, `style`, `extraTags`, `kitConfig`, `dryRun`
   - Agent uses `gherkin-transformer` skill to convert steps to Given/When/Then
   - Agent checks `outputs/<project>/tests/bdd/` for existing step definitions — reuses matching steps

4. **If `--dry-run`**: print file list and Gherkin preview (first scenario of each feature) without writing

5. **Write artifacts**:
   - `outputs/<project>/tests/bdd/<feature>.feature`
   - `outputs/<project>/tests/bdd/<feature>.steps.ts`

6. **Validate** (skip if `--dry-run`):
   - Run `npx cucumber-js --dry-run` — confirm all steps are defined
   - Run `npx tsc --noEmit` — confirm step files compile

7. **Report**: List generated files + count of new vs reused steps

## Notes
- If `--tags @smoke` is passed, applies `@smoke` tag to ALL scenarios in this run
- Tags from parsed test case IDs are always applied: `@TC-001`, `@TC-002` etc.
- If a step definition already exists for a step text, it is REUSED — not duplicated
