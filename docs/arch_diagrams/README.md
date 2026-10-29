# Architecture diagrams (C4)

C4 views of the **Warehouse Management System**. Each diagram has three files:

| File type | Purpose |
|-----------|---------|
| `*.drawio` | Editable source. Open it at [app.diagrams.net](https://app.diagrams.net) (**File → Open from → Device**) or with the draw.io VS Code/Cursor extension. |
| `*.png` | Image exported from draw.io (**File → Export as → PNG**, 200% zoom, white background). Shown in the `.md` page. |
| `*.md` | Explanation, with a Mermaid copy of the same diagram so it also renders in Cursor or GitHub. |

| Diagram | C4 level | Shows |
|---------|----------|-------|
| [c4-context.md](./c4-context.md) | 1 — Context | Users, the system, and the supplier |
| [c4-container.md](./c4-container.md) | 2 — Container | Web app, API, database |
| [c4-deployment.md](./c4-deployment.md) | Deployment | User device and web server |

Why the system has this shape: [ADR-0001](../adr/0001-client-server-web-architecture.md). Security model: [ADR-0002](../adr/0002-role-based-security.md).
