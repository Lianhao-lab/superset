### Explore Tests Migration Notes

This directory contains Playwright migrations of legacy Cypress tests for the Explore UI.

#### General Patterns
- Page object: `ExplorePage` consolidates selectors and actions (no assertions inside methods).
- Network waits: Cypress `intercept` + alias replaced with `waitForResponse` predicates or optional wait helper.
- Skipped tests: Complex dashboard cross-referencing and annotation rendering kept skipped pending fixture strategy and stable visual assertions.
- Data creation helpers from Cypress (e.g., `createSampleDashboards`) omitted; future Playwright fixtures may reintroduce.

#### File Status
- `AdhocMetrics.spec.ts`: Active – validates metric creation via adhoc editor.
- `advanced_analytics.spec.ts`: Active – tests time compare values persistence.
- `annotations.spec.ts`: Skipped – adds formula annotation using page object methods; visual layer assertion deferred.
- `chart.spec.ts`: Partially migrated – "No Results" test active; cross-referenced dashboards skipped.
- `link.spec.ts`: Active – covers view query modal, share/embed code, save-as, overwrite, and save to new dashboard flows (backend verification requests omitted).

#### Deviations From Cypress
- API response count verifications (`cy.request`) removed to reduce external coupling; can be re-added via direct API calls if needed.
- Dashboard/chart fixture creation not replicated; skipped tests clearly marked.
- Rison encoding used directly for "No Results" form data navigation; assumes default test dataset presence.

#### Future Enhancements
- Introduce Playwright test fixtures for sample dashboards/charts enabling activation of skipped tests.
- Add visual regression or DOM layer checks for annotations rendering.
- Expand `ExplorePage` with chart parameter navigation helper to remove raw encoding in spec.

#### Stability Considerations
- Optional response waits swallow timeouts to avoid brittle tests where caching removes network calls.
- Randomized names (`nanoid`) mitigate collisions when running tests repeatedly.
