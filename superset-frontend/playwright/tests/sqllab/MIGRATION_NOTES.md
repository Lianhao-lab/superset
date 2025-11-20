<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# SQL Lab Tests Migration (Cypress → Playwright)

## Summary

The `sqllab` test suite (tabs & query panel) was migrated from Cypress to Playwright following the architecture rules in `playwright/README.md`.

| Area | Cypress Pattern | Playwright Replacement | Notes |
|------|-----------------|------------------------|-------|
| Navigation | `cy.visit('/sqllab')` | `sqlLab.goto()` | Centralized in page object |
| Selectors | `cy.get(...)` / `cy.contains` | `page.locator(...)` / `expect(...).toContainText()` | All selectors consolidated |
| Tabs | Manual DOM traversal | `SqlLabPage.getTabs()` + actions | Abstraction improves readability |
| Add Tab | `cy.get('[data-test="add-tab-icon"]')...` | `sqlLab.addTabViaButton()` / `addTabViaShortcut()` | Shortcut uses `keyboard.press('Control+T')` |
| Close Tab | Dropdown + remove icon | `closeLastTabViaMenu()` & `clickLastRemoveIcon()` | Mirrors original flows |
| Editor | Chained `.type()` / `.focus()` | `setEditorContent(query)` | Handles clearing & entry |
| Limit Dropdown | Cypress menu traversal | `configureRowLimit()` | Picks first item for deterministic state |
| Run Query | `cy.get(...).click()` + `cy.intercept` | `runQuery()` + `waitForQueryExecute()` | Response waiting avoids alias bookkeeping |
| Results | `selectResultsTab()` | `getResultsTables()` | Future: add richer query assertions |
| Network Intercepts | `cy.intercept(...).as('alias')` | `waitForResponse(predicate)` | No global alias state |
| Skipped Tests | `it.skip(...)` | `test.skip(...)` | Parity preserved (flaky / heavy flows) |

## Files Migrated

- `cypress/e2e/sqllab/tabs.test.ts` → `playwright/tests/sqllab/tabs.spec.ts`
- `cypress/e2e/sqllab/query.test.ts` → `playwright/tests/sqllab/query.spec.ts` (kept skipped)
- Helper logic embedded through new page object `playwright/pages/SqlLabPage.ts`

## New Page Object: `SqlLabPage`

Responsibilities:

- Navigation (`goto`)
- Tab management (`addTabViaButton`, `addTabViaShortcut`, `closeLastTabViaMenu`, `clickLastRemoveIcon`)
- Editor interactions (`setEditorContent`, `getEditorContent`, `configureRowLimit`)
- Query execution (`runQuery`, `waitForQueryExecute`)
- Result & chart flows (`getResultsTables`, `createChartFromQuery`, datasource & column locators)

Design Principles:

- Centralized immutable `SELECTORS`
- Actions & Queries only (no assertions)
- No premature abstraction for unsused skipped flows (YAGNI)

## Skipped Tests Rationale

Three original Cypress tests were already skipped due to flakiness / complexity:

1. Query execution timer assertions
2. Save query & compare results
3. Create chart from query

They are migrated as `test.skip` with core actions in place so enabling later requires only removing the `.skip` and refining assertions.

## Improvements vs Cypress

- Eliminated implicit command queue ordering for clarity
- Removed dependency on fragile `.eq()` index selectors; uses semantic locators and `.nth()`
- Encapsulated repeated selectors; reduces drift when UI changes
- Simplified network handling using `waitForResponse` predicates
- Prepared for future extension (e.g., robust result diffing) without over-engineering

## Example Usage

```typescript
const sqlLab = new SqlLabPage(page);
await sqlLab.goto();
await sqlLab.setEditorContent('SELECT 1');
await sqlLab.runQuery();
await expect(sqlLab.getResultsTables().first()).toBeVisible();
```

## Next Recommendations

1. Convert skipped tests to active once backend fixtures are deterministic.
2. Add a utility for comparing result tables (port Cypress DOM diff logic carefully if needed).
3. Introduce retries only where truly necessary (avoid arbitrary waits).
4. Consider splitting chart creation into its own spec when stabilized.

## Non-Goals

- No mocking layer added; uses real responses.
- No artificial sleep inserted; relies on built-in auto-waits.
- Did not migrate `_skip.sourcePanel.index.test.js` (still intentionally skipped upstream).

## Validation

Type checks: `SqlLabPage.ts`, `tabs.spec.ts`, `query.spec.ts` compile without errors.

Run selectively:
```bash
npx playwright test tests/sqllab/tabs.spec.ts
npx playwright test tests/sqllab/query.spec.ts --grep "skip" # (shows skipped)
```

---
Migration complete.
