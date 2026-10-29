# Quality attributes

App-wide non-functional targets for the **Warehouse** system (design draft, 2026-10-28).

**Teaching policy:** Specs say *what* to build. This table says *how good* the system should be. Only **Accepted** rows (and feature **Requirements (FR-00N)** / **Success Criteria (SC-00N)**) constrain implementation. **Deferred** is the quality backlog / example. **Out of scope** is what not to build. See [`.cursor/rules/quality-attributes.mdc`](../../.cursor/rules/quality-attributes.mdc).

## Column meanings

| Column | Meaning |
|--------|---------|
| **Target** | Measure or bar — prefer a **number**. Illustrative until Status is **Accepted** and tests enforce it. |
| **Approach** | How the target is realized or limited |
| **Status** | **Accepted** / **Accepted (minimal)** / **Deferred** / **Out of scope** (see below) |
| **Links** | ADR (why), Cursor rule (how), code path, or **—** |

## Status values

| Status | Meaning |
|--------|---------|
| **Accepted** | In force — do not regress |
| **Accepted (minimal)** | Thin bar in force — do not expand |
| **Deferred** | Documented example — not a coding requirement unless a feature spec says so |
| **Out of scope** | Do not invent |

## Links column

| Link type | When |
|-----------|------|
| ADR | Approach decision |
| Cursor rule | Day-to-day agent/code constraint |
| Code path | Concrete implementation |
| **—** | No artifact yet |

| Attribute | Target | Approach | How we verify | Status | Links |
|-----------|--------|----------|---------------|--------|-------|
| **Security** | **100%** of `/api` routes except login and health require a session; **100%** of write routes have a role guard; **0** cross-store or cross-route reads/writes in tests; out-of-scope rows → **404**, wrong role → **403** | `authenticate` → `requireRole` → scope helpers; bcrypt hashes; owner fields from `req.user` only | Jest + supertest "wrong role" and "other store / other route" scenarios in every feature | Accepted | [ADR-0002](../adr/0002-role-based-security.md), [security.mdc](../../.cursor/rules/security.mdc), [Feature 1](../../features/feature-1-user-login.md) |
| **Data integrity (inventory)** | After **100%** of receipts, picks, adjustments, and closes, `items.qtyOnHand` = sum of `stock_lots.qty`; `qtyOnOrder` never below **0**; **0** partial writes when a step fails | One Sequelize transaction per stock change; `inventoryService` and `reorderService` are the only writers | Jest checks totals after each stock scenario; a forced-failure test | Accepted | [ADR-0003](../adr/0003-mysql-relational-database.md), [ADR-0008](../adr/0008-inventory-stock-lots-fifo.md) |
| **Correctness (order flow)** | **0** status changes outside the allowed transitions; **0** duplicate automatic orders | Action endpoints per transition; inventory-position reorder rule | One Jest test per transition plus illegal-transition tests | Accepted | [ADR-0006](../adr/0006-automatic-reorder-policy.md), [ADR-0007](../adr/0007-order-status-state-machines.md) |
| **Reliability** | Happy-path write success ≥ **99%** in test runs; every failed request returns 4xx/5xx with `{ message }` (never an empty 200) | Single error contract; central Express error handler | Jest | Accepted (minimal) | [ADR-0005](../adr/0005-node-express-rest-api.md) |
| **Availability** | Up **≥ 99%** during warehouse hours (Mon–Sat, 6 am–8 pm Central); daily database backup | Single server; nightly `mysqldump` | Manual check of backups; uptime log | Deferred | [ADR-0001](../adr/0001-client-server-web-architecture.md) |
| **Performance** | p95 API < **300 ms** for list and pick endpoints with 2,000 items and 10,000 stock lots; item search < **2 s**; pick list built in < **1 s** for a 50-line order | Indexes on UPCs, `stock_lots(itemId, receivedDate)`, `customer_orders(customerId, status)` | Seeded load test (future k6); manual timing | Deferred | [ADR-0003](../adr/0003-mysql-relational-database.md) |
| **Scalability** | **≤ 50** concurrent users (staff + stores) at one warehouse; **≤ 300** customers | Single API process and one database | N/A for this release | Out of scope | [ADR-0001](../adr/0001-client-server-web-architecture.md) |
| **Usability** | Customer places a 20-line order in **≤ 5 min**; receiver records a 10-line delivery in **≤ 5 min**; picker sees the next location without scrolling on a tablet; **100%** of button labels match Screen Requirements | Vuetify; role-based home cards; large touch targets on floor and driver screens | Manual walkthrough with the Figma flow; Vitest checks labels | Deferred | [ADR-0004](../adr/0004-vue-vuetify-spa.md), [docs/ui](../ui/README.md) |
| **Accessibility (a11y)** | Primary flows keyboard reachable; **0** icon-only buttons without `aria-label`; aim **WCAG 2.2 AA** contrast | Vuetify semantic components | Manual audit; Vitest checks `aria-label` | Deferred | [ui-style-system.mdc](../../.cursor/rules/ui-style-system.mdc) |
| **Auditability** | **100%** of quantity adjustments, receipts, picks, and shipments record who and when | `userId` / `receivedBy` / `pickedBy` / `shippedBy` set from `req.user` | Jest asserts audit fields | Accepted | [Feature 2](../../features/feature-2-item-management.md), [Feature 7](../../features/feature-7-shipping-routes.md) |
| **Observability** | **100%** of unhandled errors logged at `error`; HTTP access log; logs kept **≥ 14 days** | winston + morgan, daily rotation | Logs present in local runs | Accepted (minimal) | `backend/app/config/logger.js` |
| **Portability (devices)** | Works on current Chrome, Edge, and Safari; floor and driver screens usable at **360 px** wide | Responsive Vuetify layout | Manual check at 360 px and 1280 px | Deferred | [ADR-0004](../adr/0004-vue-vuetify-spa.md) |
| **Offline use** | **0** offline capability | Online only | N/A | Out of scope | [ADR-0001](../adr/0001-client-server-web-architecture.md) |
| **Internationalization (i18n)** | **1** locale (en-US); dates in America/Chicago | No i18n framework | N/A | Out of scope | — |
| **Maintainability** | **100%** of Gherkin scenarios mapped before merge; `npm test` green; stock rules only in services | Cursor rules + feature specs as source of truth | Merge checklist; `npm test` | Accepted | [framework.md](../../features/framework.md), [quality-attributes.mdc](../../.cursor/rules/quality-attributes.mdc) |
---

## Feature-local NFRs

If only one feature needs a bar:

1. Put it in that feature’s **Requirements (FR-00N)** or **Success Criteria (SC-00N)**.
2. Add Gherkin when tests must prove it.
3. Optionally link from a row here (“see Feature N **FR-00N** / **SC-00N**”).

See [feature spec template](../../features/framework.md#feature-spec-template) in the teaching repo.

---

## Changing a bar

1. Edit **Target**, **Approach**, **Status**, **Links**. Prefer a number in **Target**.
2. **Approach** change → [ADR](../adr/README.md); add to **Links**.
3. Agent coding constraint → `.cursor/rules/`; add to **Links**.
4. API/schema change → `features/reference/` in the same feature PR ([Agent implementation request](../../features/framework.md#agent-implementation-request)).
5. **Deferred** → **Accepted** only with matching verification.
