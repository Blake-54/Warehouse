# C4 Level 2 — Containers

The Warehouse System is a web app that calls one API, which stores everything in one database.

![C4 Container diagram](./c4-container.png)

Source: [c4-container.drawio](./c4-container.drawio)

```mermaid
C4Container
title Container Diagram — Warehouse System

Person(usr, "User", "Staff, driver, or customer")

System_Boundary(wms, "Warehouse System") {
  Container(spa, "Web App", "Vue 3")
  Container(api, "API", "Node.js / Express")
  ContainerDb(db, "Database", "MySQL")
}

Rel(usr, spa, "Uses")
Rel(spa, api, "JSON/HTTPS")
Rel(api, db, "SQL")
```

## Notes

- The API enforces all security and stock rules ([ADR-0002](../adr/0002-role-based-security.md), [ADR-0008](../adr/0008-inventory-stock-lots-fifo.md)).

**Related:** [ADR-0003](../adr/0003-mysql-relational-database.md) · [ADR-0004](../adr/0004-vue-vuetify-spa.md) · [ADR-0005](../adr/0005-node-express-rest-api.md)
