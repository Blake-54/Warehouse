# ADR-0001: Client–server web architecture for the warehouse system

**Status:** Proposed  
**Date:** 2026-10-28  
**Deciders:** Warehouse project team

## Context

The warehouse is replacing paper order books and manual stock cards. Six kinds of users work in the system at the same time:

- office workers at desks
- receivers and pickers on the warehouse floor
- managers
- drivers on the road
- customers (convenience stores) placing orders from their own stores

Everyone must see the **same** stock numbers in real time. For example, a pick recorded on the floor must immediately change what the reorder check and the customer catalog see. Customers are outside the company network.

We had to decide where the data lives, how each kind of user reaches it, and how the system stays consistent when many people change stock at once.

## Decision

Build a **browser-based single-page app (SPA)** that talks to **one central REST API** backed by **one relational database**. The API is the only component that reads or writes the database and the only place business rules are enforced.

| Layer | Choice |
|-------|--------|
| Client | Vue 3 SPA with Vuetify, running in any modern browser (desktop, tablet, or phone) |
| Server | Node.js + Express REST API under `/api` ([ADR-0005](0005-node-express-rest-api.md)) |
| Database | MySQL ([ADR-0003](0003-mysql-relational-database.md)) |
| Auth | Session token + role on every request ([ADR-0002](0002-role-based-security.md)) |

```text
Browsers (office PC, floor tablet, driver phone, store PC)
        │  HTTPS + JSON, Bearer token
        ▼
Express API  ── role guards, ownership scopes, stock rules, reorder check
        │  SQL in transactions
        ▼
MySQL (single source of truth)
```

## Consequences

### Positive
- Every role uses the same app; there is nothing to install on store PCs or driver phones.
- Stock rules (FIFO, reorder, quantity on hand) live in one place, so every screen sees the same numbers.
- Matches the OC Speckit stack, so the course rules, tests, and deploy workflow apply without changes.

### Negative / tradeoffs
- The floor and drivers need network access; there is no offline mode (out of scope in [NFR](../nfr/quality-attributes.md)).
- One API server is a single point of failure. That is acceptable for a single warehouse.
- Customers reach the system over the internet, so security has to be strong ([ADR-0002](0002-role-based-security.md)).

## Alternatives considered

| Option | Why not |
|--------|---------|
| Desktop app per PC with a shared database file | Customers and drivers could not use it; risky concurrent writes |
| Separate apps for customers and staff | Twice the code and two places for stock rules |
| Native mobile app for pickers and drivers | Extra platform; a responsive web page meets the need |

## Related artifacts

- Diagrams: [C4 Context](../arch_diagrams/c4-context.md), [C4 Container](../arch_diagrams/c4-container.md), [C4 Deployment](../arch_diagrams/c4-deployment.md)
- Rules: [project-structure.mdc](../../.cursor/rules/project-structure.mdc)
