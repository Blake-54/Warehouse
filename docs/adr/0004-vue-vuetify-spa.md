# ADR-0004: Vue 3 + Vuetify single-page app for every role

**Status:** Proposed  
**Date:** 2026-10-28  
**Deciders:** Warehouse project team

## Context

The same front end serves very different screens:

- dense data tables for office workers
- touch-friendly pick lists on tablets
- a one-column route page on a driver's phone
- a catalog and cart for store customers

The team already knows Vue from the course, and the kit's UI rules ([ui-style-system.mdc](../../.cursor/rules/ui-style-system.mdc)) assume Vuetify.

## Decision

Build one **Vue 3** SPA (Vite) using **Vuetify** components, **Vue Router** for screens, and **axios** service modules per feature.

- Each route has `meta.roles`; a global guard sends users to `home` if their role is not allowed. This is UX only; the API enforces access ([ADR-0002](0002-role-based-security.md)).
- Tables use `<v-data-table>`; forms and confirmations use `<v-dialog>`; errors use `<v-alert type="error">`; primary buttons use the `oc-cta` class.
- Floor and driver screens (Pick List, My Route, Receive Order) use large touch targets and single-column layouts below 600 px.

## Consequences

### Positive
- One codebase and one design language for all six roles.
- Vuetify's responsive grid covers desktop, tablet, and phone without a separate mobile app.
- Matches the course's Cursor rules, so agent output stays consistent.

### Negative / tradeoffs
- Vuetify adds bundle size; first load on a phone over cellular may be slower ([NFR](../nfr/quality-attributes.md) performance row).
- No offline support for drivers or pickers.

## Alternatives considered

| Option | Why not |
|--------|---------|
| React + MUI | The team and course rules are built on Vue |
| Server-rendered pages | Less responsive for scanning and picking; does not match the kit |
| Separate mobile app for drivers | Extra platform for one simple screen |

## Related artifacts

- UI: [docs/ui/README.md](../ui/README.md)
- Rules: [ui-style-system.mdc](../../.cursor/rules/ui-style-system.mdc), [frontend-services.mdc](../../.cursor/rules/frontend-services.mdc)
