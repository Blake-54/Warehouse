# ADR-0008: Inventory as stock lots with FIFO picking

**Status:** Proposed  
**Date:** 2026-10-28  
**Deciders:** Warehouse project team

## Context

The warehouse must rotate stock (oldest first) and tell pickers exactly where to go (aisle A–T, North or South side). The requirements review asked how quantity on hand, quantity on order, quantity picked, cases received, and stock lots relate, especially for partial receipts and stock split across locations.

## Decision

**Stock lots are the source of truth for physical stock.** The item's totals are kept in step with the lots in the same transaction.

| Quantity | Definition | Changed by |
|----------|------------|-----------|
| `stock_lots.qty` | Units of one item at one aisle/side, from one receipt or adjustment | Put-away (+), pick (−), adjustment (±) |
| `items.qtyOnHand` | **Always** = sum of the item's `stock_lots.qty` | The same transaction as any lot change |
| `items.qtyOnOrder` | Units ordered from suppliers and not yet received or released | Reorder (+), receipt (−, not below 0), close order (−) |
| `supplier_order_lines.casesReceived` | Running total of cases delivered for the line | Receipt |
| `customer_order_lines.qtyPicked` | Sum of `pick_list_lines.qtyPicked` for the line | Recorded pick |

**Rules:**

1. **Receiving:** each put-away location creates a new lot with `qty = cases × unitsPerCase` and `receivedDate = today`. One line may be split across several lots.
2. **Building a pick list (FIFO allocation):** for each order line, take lots with `qty > 0` ordered by `receivedDate` (oldest first), allocating until the ordered quantity is covered or lots run out. Any remainder is shown as a shortage right away.
3. **Sorting the pick list:** after allocation, sort the lines by **aisle (A→T), then side (North, South), then receivedDate**. FIFO decides *which* lots; location decides the *walking order*.
4. **Recording a pick:** subtract `qtyPicked` from the lot and from `qtyOnHand`, set `qtyPicked` on the lines, then run `checkReorder` ([ADR-0006](0006-automatic-reorder-policy.md)).
5. **No reservations:** placing an order does not hold stock; competing orders are resolved at pick time (a short pick).
6. **Adjustments:** decreases remove units from the oldest lots first; increases create a new lot at the location the manager gives.
7. Lots at 0 are kept for history and hidden from pick lists.

## Consequences

### Positive
- Answers "where is it?" and "which one is oldest?" with one query.
- `qtyOnHand` can be checked against the lots in tests (Feature 5 SC-002, Feature 6 SC-004).
- Split put-away and partial receipts are handled naturally.

### Negative / tradeoffs
- Every stock change touches at least two tables, so transactions are required ([ADR-0003](0003-mysql-relational-database.md)).
- Without reservations, a customer can order stock that another order picks first.
- Moving stock between locations needs a future feature.

## Alternatives considered

| Option | Why not |
|--------|---------|
| Only `items.qtyOnHand`, no lots | Cannot do FIFO or tell pickers a location |
| Sort the pick list purely by date | Pickers walk back and forth across aisles |
| Reserve stock when the order is placed | More states and release rules than this release needs |

## Related artifacts

- Features: [Feature 2](../../features/feature-2-item-management.md), [Feature 5](../../features/feature-5-supplier-ordering-receiving.md), [Feature 6](../../features/feature-6-customer-ordering-picking.md)
