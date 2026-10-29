# ADR-0002: Role-based access with store and route ownership scopes

**Status:** Proposed  
**Date:** 2026-10-28  
**Deciders:** Warehouse project team

## Context

The Todo example has one rule: every row belongs to one user. The warehouse is different:

- Most data (items, suppliers, stock) is **shared** by employees but limited by **job**. For example, only managers change quantities (Feature 1, US-1.3).
- Customer orders are **owned by a store**. A store must never see another store's orders (US-1.4).
- Drivers may see and ship only the orders on **their route for today** (Feature 7).
- Customers sign in from outside the building.

We need one model that covers all three cases and is easy to test.

## Decision

Every request goes through three checks, in this order:

1. **Authenticate:** `authenticate` middleware validates the Bearer token against the `sessions` table (24-hour expiry) and sets `req.user = { id, role, customerId }`. Failure → **401**.
2. **Authorize by role:** `requireRole(...roles)` on each route. A valid user with the wrong role → **403** `{ "message": "You do not have permission to do that." }`.
3. **Scope by ownership:** for store-owned and route-owned rows, helpers in `app/authorization/` load the row only if it is in scope:
   - `customerScope(req)` / `getAccessibleOrderOrNull(req, id)`: customers see only `customerId = req.user.customerId`.
   - `getDriverRouteOrderOrNull(req, id)`: drivers see only orders on today's assigned route.
   - A row that exists but is out of scope → **404**. This is the same response as a missing row, so the API never confirms that another store's order exists.

Other rules:
- Owner and audit fields (`customerId`, `userId`, `receivedBy`, `pickedBy`, `shippedBy`) are always set from `req.user`, never from the request body.
- Passwords are hashed with bcrypt (10 salt rounds); the hash is excluded from every response.
- The SPA hides menus and buttons by role for usability only. The API is the real enforcement.

## Consequences

### Positive
- One pattern for all seven features; every feature spec has a role column in its API table.
- 403 vs 404 is predictable: wrong job → 403; not yours → 404.
- Easy to test: each feature adds "wrong role" and "other store / other route" scenarios.

### Negative / tradeoffs
- Roles are fixed in code; a new job type needs a code change.
- Per-user permission overrides are not possible (out of scope).
- Route scope depends on today's date, so tests must control the clock.

## Alternatives considered

| Option | Why not |
|--------|---------|
| 403 for out-of-scope rows | Leaks the fact that another store's order exists |
| Permission table (role × action) in the database | More flexible, but more screens and testing than this release needs |
| Separate login systems for customers and staff | Two auth paths to secure and test |

## Related artifacts

- Features: [Feature 1](../../features/feature-1-user-login.md) (role table and error contract), [Feature 6](../../features/feature-6-customer-ordering-picking.md), [Feature 7](../../features/feature-7-shipping-routes.md)
- Rules: [security.mdc](../../.cursor/rules/security.mdc), [auth-patterns.mdc](../../.cursor/rules/auth-patterns.mdc)
