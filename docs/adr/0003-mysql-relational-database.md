# ADR-0003: MySQL relational database with Sequelize

**Status:** Proposed  
**Date:** 2026-10-28  
**Deciders:** Warehouse project team

## Context

Warehouse data is strongly related: items ↔ suppliers, orders ↔ order lines ↔ stock lots ↔ pick-list lines, and customers ↔ routes ↔ assignments. Several operations must change many rows **all or nothing**. For example, one recorded pick updates a stock lot, the item's quantity on hand, the order line, and possibly creates a supplier order. Delete rules ("supplier has open orders") depend on foreign keys.

## Decision

Use **MySQL** (MariaDB in XAMPP locally) as the single system of record, accessed through **Sequelize 6**.

- Foreign keys with `ON DELETE RESTRICT` on reference data (suppliers, customers, routes, items) back up the API's `409` delete rules.
- Unique indexes enforce UPCs, supplier names, store names, route names, and one assignment per route/truck/driver per date.
- Multi-row stock changes (receipt, pick, adjustment, reorder) run inside a Sequelize **transaction**.
- Quantities are integers in units; money is `DECIMAL(10,2)`.
- Models are registered in `models/index.js`; schema changes for this course use `sequelize.sync({ alter: true })` in development.

## Consequences

### Positive
- Transactions keep `qtyOnHand` equal to the sum of stock lots ([ADR-0008](0008-inventory-stock-lots-fifo.md)).
- The database also blocks bad data if a controller misses a check.
- Same database the class already runs (XAMPP), with phpMyAdmin for inspection.

### Negative / tradeoffs
- `sync({ alter: true })` is not safe for production data; real migrations would be needed later.
- Row locking under heavy picking could slow down. This is acceptable for one warehouse ([NFR](../nfr/quality-attributes.md) scalability row).

## Alternatives considered

| Option | Why not |
|--------|---------|
| MongoDB / document store | Weak multi-document transactions for the stock rules; joins needed everywhere |
| SQLite | Not suited to many users writing at once over a network |
| Raw SQL without an ORM | More code; the kit's patterns and tests assume Sequelize |

## Related artifacts

- Features: all `Data Model Requirements` sections
- Rules: [api-conventions.mdc](../../.cursor/rules/api-conventions.mdc)
