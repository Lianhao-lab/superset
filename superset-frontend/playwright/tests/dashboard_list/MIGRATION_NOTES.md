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

# Dashboard List Tests - Cypress to Playwright Migration

This document explains the migration of dashboard_list tests from Cypress to Playwright.

## Files Migrated

- `filter.test.ts` → `filter.spec.ts`
- `list.test.ts` → `list.spec.ts`

## New Files Created

- `../../pages/DashboardListPage.ts` - Page Object Model for dashboard list functionality

## Key Migration Patterns

### 1. Page Object Pattern

**Before (Cypress):**
```typescript
cy.visit(DASHBOARD_LIST);
cy.getBySel('listview-table').should('be.visible');
```

**After (Playwright):**
```typescript
const dashboardListPage = new DashboardListPage(page);
await dashboardListPage.goto();
await expect(dashboardListPage.getListViewTable()).toBeVisible();
```

**Why:** Encapsulates page interactions and selectors in a reusable class (DRY principle).

### 2. Helper Functions → Page Object Methods

**Before (Cypress):**
```typescript
import { setGridMode, clearAllInputs } from 'cypress/utils';

setGridMode('card');
clearAllInputs();
```

**After (Playwright):**
```typescript
await dashboardListPage.setGridMode('card');
await dashboardListPage.clearAllInputs();
```

**Why:** Page object methods provide better encapsulation and type safety.

### 3. API Interceptors → Response Waiting

**Before (Cypress):**
```typescript
function interceptFiltering() {
  cy.intercept('GET', `**/api/v1/dashboard/?q=*`).as('filtering');
}

setFilter('Owner', 'alpha user');
cy.wait('@filtering');
```

**After (Playwright):**
```typescript
async setFilter(filterName: string, optionValue: string): Promise<void> {
  const filteringPromise = this.page.waitForResponse(
    response =>
      response.url().includes('/api/v1/dashboard/?q=') &&
      response.request().method() === 'GET',
  );
  
  // ... UI interactions ...
  
  await filteringPromise;
}
```

**Why:** Playwright's `waitForResponse()` is more direct and doesn't require separate setup steps.

### 4. Selector Pattern

**Before (Cypress):**
```typescript
cy.getBySel('styled-card').first().click();
```

**After (Playwright):**
```typescript
await page.locator('[data-test="styled-card"]').first().click();
// Or via page object:
await dashboardListPage.getCard(0).click();
```

**Why:** Playwright uses standard CSS selectors; page object provides cleaner abstraction.

### 5. Assertions

**Before (Cypress):**
```typescript
cy.getBySel('bulk-select-copy').contains('5 Selected');
cy.getBySel('styled-card').should('have.length', 5);
```

**After (Playwright):**
```typescript
await expect(dashboardListPage.getBulkSelectCopy()).toContainText('5 Selected');
await expect(dashboardListPage.getCards()).toHaveCount(5);
```

**Why:** Playwright uses async/await with expect() - assertions stay in tests, not page objects.

### 6. Chaining → Sequential Awaits

**Before (Cypress):**
```typescript
cy.getBySel('delete-modal-input')
  .should('be.visible')
  .then($input => {
    cy.wrap($input).clear();
    cy.wrap($input).type('DELETE');
  });
```

**After (Playwright):**
```typescript
const deleteInput = page.locator('[data-test="delete-modal-input"]');
await deleteInput.waitFor({ state: 'visible' });
await deleteInput.clear();
await deleteInput.fill('DELETE');
```

**Why:** Playwright uses explicit async/await instead of command queue.

### 7. Custom Commands → API Fixtures

**Before (Cypress):**
```typescript
cy.createSampleDashboards([0, 1, 2, 3]);
```

**After (Playwright):**
```typescript
test.beforeEach(async ({ page, request, baseURL }) => {
  const apiBaseUrl = baseURL || 'http://localhost:8088';
  for (const id of [0, 1, 2, 3]) {
    await request.post(`${apiBaseUrl}/api/v1/dashboard/`, {
      data: {
        dashboard_title: `${id + 1} - Sample dashboard`,
      },
    });
  }
});
```

**Why:** Playwright's `request` fixture provides direct API access without custom commands.

### 8. Test Isolation

**Before (Cypress):**
```typescript
describe('list mode', () => {
  before(() => {
    cy.visit(DASHBOARD_LIST);
    setGridMode('list');
  });
  
  it('test 1', () => { /* uses shared state */ });
  it('test 2', () => { /* uses shared state */ });
});
```

**After (Playwright):**
```typescript
test.describe('list mode', () => {
  test.beforeEach(async ({ page }) => {
    dashboardListPage = new DashboardListPage(page);
    await dashboardListPage.goto();
    await dashboardListPage.setGridMode('list');
  });
  
  test('test 1', async () => { /* isolated */ });
  test('test 2', async () => { /* isolated */ });
});
```

**Why:** Using `beforeEach` instead of `before` provides better test isolation (Playwright best practice).

## Architecture Decisions

### DashboardListPage Design

The page object follows the architecture guidelines in `playwright/README.md`:

1. **Centralized Selectors** - All selectors in private static `SELECTORS` constant
2. **Actions and Queries** - Methods for what you can do and what you can get
3. **No Assertions** - Assertions stay in test files
4. **Reusable Components** - Uses core components (Modal, Input) where applicable
5. **YAGNI Principle** - Only implements what's needed for current tests

### Method Categories

- **Navigation**: `goto()`, `setGridMode()`
- **Actions**: `clickSortHeader()`, `toggleBulkSelect()`, `confirmDelete()`
- **Queries**: `getCards()`, `getTableRows()`, `getBulkSelectCopy()`
- **Complex Flows**: `setFilter()`, `editDashboardTitle()` (includes API waits)

## Known Limitations

1. **Sample Dashboard Creation**: The Cypress test uses `cy.createSampleDashboards()` which is a custom command not visible in the codebase. The Playwright version uses direct API calls, but the exact payload format may need adjustment based on the actual API.

2. **Skipped Test**: The "should delete correctly in list mode" test remains skipped as it was in the original Cypress version.

3. **Ant Design Interactions**: Some Ant Design component interactions (like Select dropdowns) may behave differently. The implementation uses `force: true` where needed to handle overlay issues.

## Running the Tests

```bash
# Run all dashboard_list tests
npx playwright test tests/dashboard_list

# Run specific test file
npx playwright test tests/dashboard_list/filter.spec.ts

# Run with UI mode for debugging
npx playwright test tests/dashboard_list --ui

# Run in headed mode to see browser
npx playwright test tests/dashboard_list --headed
```

## Next Steps

1. Verify sample dashboard creation works with actual API
2. Test in CI environment to ensure stability
3. Remove skipped test or fix underlying issue
4. Consider adding visual regression tests for dashboard cards/list items
