# ADR-0006: Automatic reordering with a min/max inventory-position policy

**Status:** Proposed  
**Date:** 2026-10-28  
**Deciders:** Warehouse project team

## Context

Feature 5 requires the system to order stock automatically when an item runs low, without creating duplicate orders. Two questions decide the design:

1. **What counts as "low"?** If we compare only quantity on hand to the minimum, every pick below the minimum would create another order while the first one is still on the truck.
2. **When does the check run?** A nightly job would let stock fall far below the minimum during a busy day.

## Decision

Use a **min/max policy on inventory position**, checked **every time quantity on hand goes down**.

```text
position = qtyOnHand + qtyOnOrder
if minQty and maxQty are set and position < minQty:
    cases = ceil((maxQty - position) / unitsPerCase)
    add a line to today's Open automatic order for the item's supplier (create it if needed)
    qtyOnOrder += cases * unitsPerCase
```

- `checkReorder(itemId, transaction)` lives in `app/services/reorderService.js`. Feature 2 (adjustment), Feature 5 (closing a short order), and Feature 6 (recorded pick) call it inside their own transaction.
- Orders are in whole cases, so the position can end slightly above `maxQty`.
- One automatic order per supplier per day, with lines added as items run low.

## Consequences

### Positive
- Duplicate orders are impossible: once an order is placed, the position is back at or above the minimum.
- Stock is replenished as soon as it runs low, not the next morning.
- One order per supplier per day matches how the office faxes orders.

### Negative / tradeoffs
- Reorders depend on accurate `qtyOnOrder`; a short delivery must be **closed** (Feature 5, US-5.4) or the shortfall stays "on order" forever.
- There is no forecasting or lead-time logic; min/max are set by hand by managers.

## Alternatives considered

| Option | Why not |
|--------|---------|
| Compare only on-hand to minimum | Creates repeat orders while stock is in transit |
| Nightly batch job | Stock can run out during the day; needs a scheduler |
| Manager approves every suggested order | Slower; requirements say "automatically" |

## Related artifacts

- Features: [Feature 2](../../features/feature-2-item-management.md), [Feature 5](../../features/feature-5-supplier-ordering-receiving.md), [Feature 6](../../features/feature-6-customer-ordering-picking.md)
- ADRs: [ADR-0008](0008-inventory-stock-lots-fifo.md)
