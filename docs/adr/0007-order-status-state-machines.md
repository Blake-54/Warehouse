# ADR-0007: Order status state machines

**Status:** Proposed  
**Date:** 2026-10-28  
**Deciders:** Warehouse project team

## Context

The requirements listed order statuses but not which changes are allowed, who can make them, or what "open" means for the delete rules in Features 3 and 4. Without one definition, each feature could drift, for example by letting a driver ship an order that was never picked.

## Decision

Statuses only change through the action endpoints below. Any other change → `400`.

### Customer orders (Features 6 and 7)

```text
          cancel (customer own / office / manager)
   ┌───────────────────────────────► Cancelled
   │
  New ──start-picking──► Picking ──finish-picking──► Picked ──ship──► Shipped
      (picker/manager,     (picker/manager, every      (driver on
       bin required)        line recorded)              today's route)
```

| From | To | Action | Roles | Extra rule |
|------|----|--------|-------|-----------|
| New | Picking | `POST /customer-orders/:id/start-picking` | picker, manager | Bin number required; pick list built |
| New | Cancelled | `POST /customer-orders/:id/cancel` | customer (own), office, manager | — |
| Picking | Picked | `POST /customer-orders/:id/finish-picking` | picker, manager | Every pick-list line recorded |
| Picked | Shipped | `POST /customer-orders/:id/ship` | driver | Order is on the driver's route today |

**Open customer order** = `New`, `Picking`, or `Picked` (blocks deleting the customer or its items).

### Supplier orders (Feature 5)

```text
Open ──receipt (some lines short)──► Partial ──receipt (all complete)──► Received
  │                                     │
  └──receipt (all complete)──► Received │
  └──────────close─────────► Closed ◄───┘
```

| From | To | Action | Roles | Extra rule |
|------|----|--------|-------|-----------|
| Open / Partial | Partial | `POST /supplier-orders/:id/receipts` | receiver, manager | Some lines still short |
| Open / Partial | Received | `POST /supplier-orders/:id/receipts` | receiver, manager | Every line received ≥ ordered |
| Open / Partial | Closed | `POST /supplier-orders/:id/close` | office, manager | Releases the remaining quantity on order |

**Open supplier order** = `Open` or `Partial` (blocks deleting the supplier or its items).

`Shipped`, `Cancelled`, `Received`, and `Closed` are final.

## Consequences

### Positive
- One place defines "open," used by every delete rule.
- Each transition has one endpoint and one test, so illegal jumps are impossible.
- Answers the requirements review's request for explicit transitions and roles.

### Negative / tradeoffs
- No undo (for example, Picked back to Picking). A mistake needs a manager adjustment.
- New statuses (for example, Delivered) need an ADR update and new endpoints.

## Alternatives considered

| Option | Why not |
|--------|---------|
| Free `status` field editable by staff | Allows Shipped without picking; breaks stock math |
| Single "Complete" status for supplier orders | Cannot tell a full delivery from a closed short one |

## Related artifacts

- Features: [Feature 3](../../features/feature-3-supplier-management.md), [Feature 4](../../features/feature-4-customer-management.md), [Feature 5](../../features/feature-5-supplier-ordering-receiving.md), [Feature 6](../../features/feature-6-customer-ordering-picking.md), [Feature 7](../../features/feature-7-shipping-routes.md)
- Reference: `features/reference/behavior.md` (to be updated when features ship)
