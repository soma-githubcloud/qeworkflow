# Plugin: Jira / Zephyr / Xray

**Activation**: Set `plugins.jira.enabled: true` in `tc-to-automate/configs/integrations.yml`.

## Capabilities

### Import Test Cases from Jira (`/ingest` integration)
When `importTestCases: true`:
1. Connect to Jira REST API: `GET /rest/api/3/search?jql=project=<projectKey> AND issuetype=Test`
2. Extract: issue key (→ TC-ID), summary (→ title), description (→ steps), acceptance criteria (→ expectedResults)
3. Map to the `intake.summary.json` test case schema
4. Feed into the normal `/ingest` → `/generate-automation-tests` pipeline

### Push Execution Results (`/run-smoke` integration)
When `pushResults: true` and `/run-smoke` completes:
1. Read Playwright JSON output or Allure result XMLs
2. Map each test result to its `@TC-<id>` tag
3. Update Jira issue test status via Zephyr/Xray API

## Configuration (in integrations.yml)
```yaml
jira:
  enabled: true
  baseUrl: ${JIRA_URL}
  projectKey: QA
  apiToken: ${JIRA_API_TOKEN}
  importTestCases: true
  pushResults: false
```

## Required Environment Variables
- `JIRA_URL` — Jira base URL (e.g., `https://your-org.atlassian.net`)
- `JIRA_API_TOKEN` — API token (generate at id.atlassian.com → Security → API tokens)

## Security Note
Never commit `JIRA_API_TOKEN`. Add to `.env` (local) or CI secrets. The `.env` file must be in `.gitignore`.
