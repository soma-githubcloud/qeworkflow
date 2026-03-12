# Kit-U2: UI Existing Playwright Base

**Extends**: Kit-U1 (all templates, locator strategies, and rules are inherited)

## Key Difference from Kit-U1

This kit integrates into an EXISTING Playwright TypeScript project instead of creating a
greenfield scaffold. The framework-analyzer-agent must run first to map the existing structure.

## Framework Analyzer Behavior

Before generating any artifacts, the analyzer agent will:
1. Detect existing `playwright.config.ts` — extract `testDir`, `baseURL`, reporter config
2. Scan existing `pages/` or equivalent directory — catalog existing Page Object classes
3. Detect naming conventions in use (suffix, directory layout, import style)
4. Report findings to the user before proceeding
5. Adapt generated files to match the detected conventions (do NOT override them)

## What Is Never Overwritten
- Existing `playwright.config.ts` (only patched to add allure reporter if missing)
- Existing `package.json` (only new dependencies added, existing scripts preserved)
- Existing Page Object files (new POMs are created for NEW components only)
- Existing test spec files (new spec files always get unique names)

## Adapter Rules
- If existing POMs use a base class, generated POMs must extend it
- If existing project uses `import { Page } from 'playwright'` (not `@playwright/test`), match that
- If existing project uses absolute imports (`@pages/`), generate with same aliases
- Report any convention conflicts to the user for manual resolution
