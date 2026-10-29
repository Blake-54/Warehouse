# ADR-0005: Node.js + Express REST API with one error contract

**Status:** Proposed  
**Date:** 2026-10-28  
**Deciders:** Warehouse project team

## Context

Seven features add about 50 endpoints. The requirements review pointed out that error codes and messages were not consistent across features (for example, 400 in one place and 409 in another for similar cases). Implementers and tests need one predictable contract.

## Decision

Use **Node.js + Express** (ES modules) with a **REST + flat JSON** API mounted at **`/api`**.

**Resource style:**
- Nouns for resources: `/api/items`, `/api/suppliers`, `/api/customer-orders`.
- State changes that are not simple edits use action sub-resources with `POST`: `/receipts`, `/close`, `/cancel`, `/start-picking`, `/finish-picking`, `/ship`. Each one maps to one allowed transition in [ADR-0007](0007-order-status-state-machines.md).

**One error contract for all features:**

| Status | Meaning |
|--------|---------|
| `400` | Validation failed, duplicate value, or a status change that is not allowed |
| `401` | No session or expired session |
| `403` | Signed in, wrong role |
| `404` | Not found **or** outside the caller's store/route scope |
| `409` | Delete blocked by related data (open orders, stock, assigned customers) |

Body is always `{ "message": "Human-readable sentence." }`. Success bodies are flat objects or arrays, with no `{ success, data }` envelope.

**Layers:** routes → controllers → services (`inventoryService`, `reorderService`) → Sequelize models. Controllers never change stock quantities directly.

## Consequences

### Positive
- Every feature spec's error table follows the same rules, so Gherkin and tests are easy to write.
- Action endpoints make status rules explicit and testable.
- Same stack and logging (winston + morgan) as the course kit.

### Negative / tradeoffs
- Action endpoints are less "pure REST" than `PATCH { status }`, but they prevent clients from jumping to any status.
- A single Node process; long reports could block requests (reports are out of scope).

## Alternatives considered

| Option | Why not |
|--------|---------|
| `PATCH /orders/:id { status }` | Clients could request illegal transitions; rules scattered |
| GraphQL | Not in the course stack; harder to test with supertest |
| 422 for validation | The kit and all existing Gherkin use 400 |

## Related artifacts

- Features: the `API Requirements` section in every feature; the contract table in [Feature 1](../../features/feature-1-user-login.md#api-requirements)
- Rules: [api-conventions.mdc](../../.cursor/rules/api-conventions.mdc)
