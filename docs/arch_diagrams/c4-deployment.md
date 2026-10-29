# C4 Deployment

The web app runs in the user's browser; the API and database run on one web server.

![C4 Deployment diagram](./c4-deployment.png)

Source: [c4-deployment.drawio](./c4-deployment.drawio)

```mermaid
C4Deployment
title Deployment Diagram — Warehouse System

Deployment_Node(dev, "User Device", "Web browser") {
  Container(spa, "Web App", "Vue 3")
}
Deployment_Node(srv, "Web Server", "Linux") {
  Container(api, "API", "Node.js / Express")
  ContainerDb(db, "Database", "MySQL")
}

Rel(spa, api, "HTTPS")
Rel(api, db, "SQL")
```

## Notes

- Locally (XAMPP), the user device and web server are the same laptop.

**Related:** [ADR-0001](../adr/0001-client-server-web-architecture.md)
