# /setup-ci — Scaffold CI/CD Pipeline

Generates a CI/CD pipeline YAML that runs the automation suite, publishes the Allure report,
and uploads artifacts. Idempotent — safe to re-run; updates existing file if configuration changes.

## Arguments

| Argument | Description |
|---|---|
| `--provider github\|azdo\|gitlab` | CI provider (default: github) |
| `--env staging\|prod` | Target environment (sets BASE_URL env var source) |
| `--matrix` | Add multi-browser matrix (chromium + firefox + webkit) |
| `--schedule` | Add nightly cron schedule (`0 2 * * *`) |

## Execution Steps

1. **Read kit.config.json** — get `tech.testRunner`, `reporting`, `projectName`
2. **Read `tc-to-automate/configs/integrations.yml`** — check if `allureServer.enabled = true` (add publish step if so)

3. **Spawn ci-agent** (read `tc-to-automate/kits/_base/agents/ci-agent.md`)
   - Pass: `provider`, `env`, `matrix`, `schedule`, `kitConfig`, `integrations`
   - Agent uses `ci-scaffolder` skill for YAML generation

4. **Write output** based on `--provider`:
   - GitHub: `outputs/<project>/configs/.github/workflows/ui.yml`
   - Azure DevOps: `outputs/<project>/configs/azure-pipelines.yml`
   - GitLab: `outputs/<project>/configs/.gitlab-ci.yml`

5. **Validate**: Check YAML syntax (run `npx js-yaml <file>` to parse)

6. **Show generated pipeline steps** as a summary list

## Generated GitHub Actions Pipeline (Kit-U1 default)

```yaml
name: UI-Playwright
on:
  push:
    branches: [main]
  workflow_dispatch:
  schedule:                           # only if --schedule
    - cron: '0 2 * * *'

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        browser: [chromium]           # expands to [chromium, firefox, webkit] if --matrix
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx playwright install --with-deps ${{ matrix.browser }}
      - run: npx playwright test --grep "@smoke" --project=${{ matrix.browser }}
        env:
          BASE_URL: ${{ secrets.BASE_URL }}          # staging or prod from --env
      - run: npx allure generate allure-results --clean -o allure-report
        if: always()
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-report-${{ matrix.browser }}
          path: allure-report
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report-${{ matrix.browser }}
          path: playwright-report
```
