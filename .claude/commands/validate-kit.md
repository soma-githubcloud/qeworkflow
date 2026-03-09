# /validate-kit — Smoke-Check Generated Project

Validates that all generated test artifacts compile, lint cleanly, and are structurally correct.
Run after any generation command or when diagnosing issues.

## Execution Steps

1. **Read kit.config.json** — get `tech.language`, `tech.testRunner`, `outputDir`, `projectName`

2. **Set project path**: `projectPath = <outputDir>/<projectName>`

3. **Run validation checks based on tech stack**

### TypeScript (Kit-U1, U2, H1)
```bash
cd <projectPath>
npx tsc --noEmit                    # Type-check all generated TS files
npx eslint "**/*.ts" --max-warnings 0  # Lint (if .eslintrc exists)
npx playwright test --list          # Confirm tests are discoverable
```

### Java (Kit-U3, U4, A1, H2)
```bash
cd <projectPath>
mvn compile -q                      # Compile all Java source
mvn test-compile -q                 # Compile test sources
```

### Python (Kit-A2, H3)
```bash
cd <projectPath>
pytest --collect-only -q            # Confirm tests discoverable
python -m py_compile tests/**/*.py  # Syntax check
```

4. **BDD validation** (if `testStyle = "bdd"`)

TypeScript BDD:
```bash
npx cucumber-js --dry-run           # Confirm all steps are defined
```

Java BDD (if Cucumber is configured):
```bash
mvn test -Dcucumber.options="--dry-run"
```

5. **Report results**
   - PASS: "All validation checks passed. Project is ready."
   - FAIL: Show exact error output, attempt to fix automatically if the fix is unambiguous
     (e.g., missing import, wrong relative path), then re-run validation
   - Never leave the user with failing validation without explaining what's wrong

## Notes
- This command is READ-ONLY except for auto-fixing obvious compile errors
- If auto-fix is needed, describe what was changed before making the edit
