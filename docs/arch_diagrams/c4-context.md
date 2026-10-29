# C4 Level 1 — System context

Who uses the Warehouse System and what sits outside it.

![C4 Context diagram](./c4-context.png)

Source: [c4-context.drawio](./c4-context.drawio)

```mermaid
C4Context
title System Context — Warehouse System

Person(staff, "Warehouse Staff", "Office, manager, receiver, picker")
Person(drv, "Delivery Driver")
Person_Ext(cst, "Store Customer")
System(wms, "Warehouse System", "Inventory, orders, picking, shipping")
System_Ext(sup, "Supplier")

Rel(staff, wms, "Uses")
Rel(drv, wms, "Uses")
Rel(cst, wms, "Places orders")
Rel(wms, sup, "Supplier orders (fax)")
```

## Notes

- Supplier orders are printed and faxed by the office; there is no automated link to suppliers ([Feature 5](../../features/feature-5-supplier-ordering-receiving.md)).

**Related:** [ADR-0001](../adr/0001-client-server-web-architecture.md) · [c4-container.md](./c4-container.md)
