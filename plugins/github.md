# Plugin: GitHub

**Activation**: Set `plugins.github.enabled: true` in `tc-to-automate/configs/integrations.yml`.

## Capabilities

### Auto-Commit Generated Artifacts
After any generation command completes (`/generate-automation-tests`, `/gen-bdd`, etc.), if this plugin
is enabled, use the **git MCP** to:
1. `git add outputs/<projectName>/` — stage all generated files
2. `git commit -m "<commitMessage from config>"` — commit with configured message
3. Do NOT push — user controls when to push

### Create Pull Request
After committing:
1. If `createPr: true`: use GitHub API to create a PR from the current branch to `prBaseBranch`
2. PR title: `"feat(<projectName>): generated automation artifacts [kit-u1]"`
3. PR body: generated list of files created, summary of test cases, locator strategy used

## Configuration (in integrations.yml)
```yaml
github:
  enabled: true
  repoOwner: ${GITHUB_OWNER}
  repoName: ${GITHUB_REPO}
  token: ${GITHUB_TOKEN}
  createPr: true
  prBaseBranch: main
  commitMessage: "feat: generated kit-u1 test artifacts"
```

## Required Environment Variables
- `GITHUB_OWNER` — GitHub org or username
- `GITHUB_REPO` — Repository name
- `GITHUB_TOKEN` — Personal access token with `repo` scope

## Security Note
Never commit `GITHUB_TOKEN`. Always reference as `${GITHUB_TOKEN}` in integrations.yml.

## MCP Used
Uses `mcp__git__git_add`, `mcp__git__git_commit`, `mcp__git__git_status`.
GitHub API calls use the shell MCP with `gh` CLI or direct REST API via curl.
