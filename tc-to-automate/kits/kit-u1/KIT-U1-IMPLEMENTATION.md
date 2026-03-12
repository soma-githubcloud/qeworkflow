# Kit-U1 — High-Level Implementation

> **Kit**: UI Greenfield — Playwright TypeScript
> **Mode**: Greenfield (creates complete project from scratch)
> **Stack**: Playwright + TypeScript + Allure + optional Cucumber BDD

---

## 1. What Kit-U1 Is

Kit-U1 creates a complete, production-ready Playwright + TypeScript test automation project from
scratch. It is the primary kit and the reference implementation that all other kits extend or fork
from.

---

## 2. Component Map

```
Kit-U1 = CLAUDE.md orchestrator
       + kit.config.json         (active kit selector)
       + KIT.md                  (TS naming rules + code shape)
       + 12 slash commands       (user entry points)
       + 9 agents                (execution workers)
       + 7 skills                (reusable knowledge modules)
       + 9 templates             (code generation stencils)
       + 4 path-scoped rules     (auto-applied code quality guards)
       + 3 plugins               (optional integrations)
       + 4 MCP tools             (browser, filesystem, git, shell)
```

---

## 3. End-to-End Flow

The primary workflow is `/generate-automation-tests`. Every other command is a focused subset of it.

```
User Input (URL / DOM snapshot / test cases)
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│  STEP 0 — intake-agent                                           │
│  • Classifies inputs: app_url / dom_snapshot / test_cases        │
│  • Determines strategy: playwright-mcp | dom-based | ai-guided   │
│  • Parses test cases → structured TC objects                     │
│  • Writes: outputs/<project>/intake.summary.json                 │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│  STEP 1 — locator-agent (3-tier pipeline)                        │
│  Tier 1 (app_url):   Playwright MCP → browser_navigate +         │
│                      browser_snapshot → real DOM inspection       │
│  Tier 2 (dom file):  Parse HTML → extract attributes             │
│  Tier 3 (neither):   Infer selectors from test case semantics    │
│  All tiers: selector-scorer skill → numeric score (0.99 → 0.50)  │
│  • Writes: outputs/<project>/selectors.json                      │
│  • Writes: outputs/<project>/selectors-lint.html (scores < 0.70) │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│  STEP 2 — pom-agent                                              │
│  • Reads: selectors.json                                         │
│  • Renders: page-object.ts.tmpl per component                    │
│  • Writes: outputs/<project>/src/pages/<Name>Page.ts             │
│  • Idempotent — safe to re-run, won't duplicate locators         │
└──────────────────────────────────────────────────────────────────┘
    │
    ├── (--bdd flag) ──► bdd-agent
    │                       • gherkin-transformer skill
    │                       • Writes: tests/bdd/<name>.feature
    │                       • Writes: tests/bdd/<name>.steps.ts
    │
    └── (default) ─────► testgen-agent
                            • Writes: tests/nonbdd/<TC-ID>-<slug>.spec.ts
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│  STEP 4 — reporting-agent                                        │
│  • Wires allure-playwright + HTML reporter in playwright.config   │
│  • Idempotent — no-op if already configured                      │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
  tsc --noEmit  →  0 errors required before delivery
```

---

## 4. Agents and Their Single Responsibilities

| Agent | File | Single Responsibility |
|---|---|---|
| `intake-agent` | `tc-to-automate/kits/_base/agents/intake-agent.md` | Normalize inputs → `intake.summary.json` |
| `locator-agent` | `tc-to-automate/kits/_base/agents/locator-agent.md` | Produce `selectors.json` via 3-tier pipeline |
| `pom-agent` | `tc-to-automate/kits/_base/agents/pom-agent.md` | Generate `*Page.ts` POM files from `selectors.json` |
| `testgen-agent` | `tc-to-automate/kits/_base/agents/testgen-agent.md` | Generate non-BDD `.spec.ts` files |
| `bdd-agent` | `tc-to-automate/kits/_base/agents/bdd-agent.md` | Generate `.feature` + `.steps.ts` together |
| `datagen-agent` | `tc-to-automate/kits/_base/agents/datagen-agent.md` | Generate faker-js factories + data fixtures |
| `ci-agent` | `tc-to-automate/kits/_base/agents/ci-agent.md` | Generate GitHub Actions / AzDO / GitLab YAML |
| `scaffolder-agent` | `tc-to-automate/kits/kit-u1/agents/framework-setup-agent.md` | Create project structure + config files |
| `reporting-agent` | `tc-to-automate/kits/_base/agents/reporting-agent.md` | Wire Allure + HTML reporters |

---

## 5. Skills (Reusable Knowledge Modules)

| Skill | File | Purpose |
|---|---|---|
| `selector-scorer` | `tc-to-automate/kits/_base/skills/selector-scorer.md` | Numeric 0.99→0.50 scoring; lint threshold at 0.70 |
| `locator-strategies` | `tc-to-automate/kits/_base/skills/locator-strategies.md` | Playwright selector priority rules; what to prefer and avoid |
| `gherkin-transformer` | `tc-to-automate/kits/_base/skills/gherkin-transformer.md` | Maps TC steps → Given/When/Then; enforces domain language guard |
| `data-factory` | `tc-to-automate/kits/_base/skills/data-factory.md` | faker-js field patterns; positive/boundary/negative data sets |
| `ci-scaffolder` | `tc-to-automate/kits/_base/skills/ci-scaffolder.md` | GitHub Actions YAML patterns; secrets injection; matrix strategy |
| `test-case-parser` | `tc-to-automate/kits/_base/skills/test-case-parser.md` | Parses 4 input formats into structured TC objects |
| `template-renderer` | `tc-to-automate/kits/_base/skills/template-renderer.md` | Handlebars `{{var}}` / `{{#if}}` / `{{#each}}` rendering rules |

---

## 6. Templates → Generated Files

| Template | Rendered To |
|---|---|
| `page-object.ts.tmpl` | `src/pages/<Name>Page.ts` |
| `spec-nonbdd.ts.tmpl` | `tests/nonbdd/<TC-ID>-<slug>.spec.ts` |
| `spec-bdd.feature.tmpl` | `tests/bdd/<name>.feature` |
| `step-definitions.ts.tmpl` | `tests/bdd/<name>.steps.ts` |
| `playwright.config.ts.tmpl` | `configs/playwright.config.ts` |
| `package.json.tmpl` | `package.json` (pinned deps + conditional BDD extras) |
| `allure.config.ts.tmpl` | `configs/allure.config.ts` |
| `factories.ts.tmpl` | `src/fixtures/factories.ts` |
| `ci-github.yml.tmpl` | `configs/.github/workflows/ui.yml` |

---

## 7. Slash Commands (User Entry Points)

| Command | Flags | Role |
|---|---|---|
| `/setup-kit` | — | Bootstrap project; write configs, install deps, verify compile |
| `/ingest` | `--url --dom --cases` | Standalone input classification — preview `intake.summary.json` |
| `/generate-automation-tests` | `--url --cases --dom --bdd --tags --dry-run` | **Primary**: full pipeline (Steps 0–4) |
| `/generate-locators` | `--mode mcp\|dom\|ai` | Extract selectors only → `selectors.json` + lint report |
| `/generate-page-objects` | — | POMs from existing `selectors.json` (skips locator step) |
| `/gen-bdd` | `--cases --style strict\|relaxed --tags --dry-run` | BDD-only: feature + steps from test cases |
| `/gen-data` | `--schema --count --boundary --negative` | Faker fixtures: factories + static data files |
| `/add-test` | `--case --page --bdd --tags --dry-run` | Incremental: one new test using existing POMs |
| `/setup-ci` | `--provider github\|azdo\|gitlab --matrix --schedule` | CI YAML generation |
| `/setup-reporting` | — | Wire Allure + HTML reporters; idempotent |
| `/run-smoke` | — | Execute `@smoke` tests → generate + attach Allure + HTML reports |
| `/validate-kit` | — | `tsc --noEmit` + selector lint + `playwright test --list` + Cucumber dry-run |

---

## 8. Path-Scoped Rules (Auto-Applied by File Pattern)

| Rule File | Applied To | Enforces |
|---|---|---|
| `playwright-ts.md` | `*.spec.ts`, `*.page.ts` | `@playwright/test` imports only; `readonly` locators; `test.step()` required; no `waitForTimeout` |
| `bdd-gherkin.md` | `*.feature` | Tag taxonomy (`@smoke`, `@regression`, `@module()`, `@TC-*`, `@wip`); domain language guard; one assertion per `Then` |
| `page-objects.md` | `*Page.ts` | No assertions in POMs; `Promise<void>` returns; fluent naming conventions |
| `python-test.md` | `*_test.py` | Not used by U1 — inherited by A2/H3 kits |

---

## 9. MCP Tools Used by Kit-U1

| MCP Server | Used For | Key Tools |
|---|---|---|
| `playwright` | Tier-1 locator extraction; `/run-smoke` execution | `browser_navigate`, `browser_snapshot`, `browser_screenshot` |
| `filesystem` | Read DOM snapshot files; write all generated artifacts | `read_file`, `write_file`, `create_directory` |
| `git` | GitHub plugin: commit artifacts, prepare PRs | `git_add`, `git_commit`, `git_status` |
| `shell` | Run `npm ci`, `tsc`, `playwright test`, `allure` commands | `execute_command` |

---

## 10. Output Directory Layout

```
outputs/<project-name>/
├── intake.summary.json               ← Step 0: normalized input classification
├── selectors.json                    ← Step 1: scored selector map
├── selectors-lint.html               ← Step 1: flags selectors with score < 0.70
│
├── src/
│   ├── pages/
│   │   └── LoginPage.ts              ← Step 2: Page Object per component
│   └── fixtures/
│       ├── factories.ts              ← /gen-data: faker factory functions
│       └── data/
│           └── user.json             ← /gen-data: static boundary/negative sets
│
├── tests/
│   ├── nonbdd/
│   │   └── TC001-valid-login.spec.ts ← Step 3 (default): Playwright spec
│   └── bdd/
│       ├── login.feature             ← Step 3 (--bdd): Gherkin feature
│       └── login.steps.ts            ← Step 3 (--bdd): Cucumber step definitions
│
└── configs/
    ├── playwright.config.ts
    ├── allure.config.ts
    └── .github/
        └── workflows/
            └── ui.yml                ← /setup-ci: GitHub Actions pipeline
```

---

## 11. Selector Quality Gates

All selectors pass through the `selector-scorer` skill before any POM or spec is generated:

| Attribute Type | Score | Status |
|---|---|---|
| `data-testid` | 0.99 | Always preferred |
| `aria-label` | 0.92 | Excellent |
| `placeholder` | 0.88 | Good |
| `role` + `name` | 0.85 | Good |
| `role` only | 0.78 | Acceptable |
| `id` | 0.72 | Acceptable |
| `text` | 0.65 | **Flagged in lint** |
| CSS class | 0.60 | **Flagged in lint** |
| XPath fallback | 0.50 | **Flagged in lint** |

> **Threshold**: `score < 0.70` → appears in `selectors-lint.html` for manual review.
> **AI-guided tier**: all scores capped at **0.75** (inferred, not confirmed against real DOM).

---

## 12. Optional Integrations (Plugins)

| Plugin | File | Activated By | What It Does |
|---|---|---|---|
| GitHub | `plugins/github.md` | `integrations.yml → github.enabled: true` | `git commit` artifacts via git MCP; optionally creates a PR |
| Jira / Xray | `plugins/jira.md` | `integrations.yml → jira.enabled: true` | Import test cases from Jira; push results after `/run-smoke` |
| Allure Server | `plugins/allure-server.md` | `integrations.yml → allureServer.enabled: true` | Publish allure-results to hosted Allure Server |

All plugins are **disabled by default**. Secrets are always referenced as `${ENV_VAR}` in
`tc-to-automate/configs/integrations.yml` — never hardcoded.

---

## 13. Key Design Constraints

| Constraint | Detail |
|---|---|
| **Greenfield only** | Kit-U1 creates new projects. Use Kit-U2 for existing Playwright projects. |
| **`--dry-run` on everything** | All generation commands preview the file list without writing unless the flag is omitted. |
| **Idempotent agents** | Every agent can be re-run safely. Existing files are patched, not overwritten. |
| **Compile gate** | `tsc --noEmit` must pass before artifacts are considered delivered. |
| **No inline locators** | Selectors live exclusively in `src/pages/*.ts`, never in spec files. |
| **Step 0 always runs** | `intake-agent` is mandatory as the first step in every generation command. |
| **Selectors file** | Always named `selectors.json` — never `locators.json`. |

---

## 14. Extending to Other Kits

Kit-U1 is the base. Other kits extend it with minimal overrides:

| Kit | Extends | Key Difference |
|---|---|---|
| **Kit-U2** | Kit-U1 | `mode: existing` — adds `framework-analyzer-agent` to detect existing project structure |
| **Kit-H1** | Kit-U1 | Adds `api-spec-parser-agent`; hybrid test generator combines UI + API in one spec |
| **Kit-U3/U4** | `_base` | Replaces TypeScript with Java; CDP-based locator tier instead of Playwright MCP |

**New kit checklist**:
1. Create `tc-to-automate/kits/<kit-id>/kit.config.json` with `"extends"` pointing to parent
2. Add `tc-to-automate/kits/<kit-id>/KIT.md` with tech-specific naming and code shape rules
3. Add `tc-to-automate/kits/<kit-id>/templates/` for any new file types
4. Override only the agents that differ in behavior — all others inherit from `_base`
