# Feature: Supplier Ordering and Receiving

**Feature ID:** 5
**Branch pattern:** `feature/5-supplier-ordering-receiving`
**Status:** Draft
**Created:** 2026-09-29
**Updated:** 2026-10-28 (design draft)
**Input:** The system orders more stock when it runs low, and receivers check deliveries and record where items are put away
**Depends on:** [Feature 2 — Item Management](feature-2-item-management.md), [Feature 3 — Supplier Management](feature-3-supplier-management.md)

---

## User Stories

### US-5.1: Order automatically when stock is low
**As a** office worker  
**I want to** have the system order more automatically when an item is low  
**So that** we do not run out of stock

**Priority:** P1  
**Independent test:** Dropping below the minimum creates a supplier order  
**Acceptance scenarios:** see ### US-5.1 under Acceptance Criteria

### US-5.2: Check a delivery
**As a** receiver  
**I want to** compare what arrived with what was ordered  
**So that** shortages and extras are caught

**Priority:** P1  
**Independent test:** Receiving fewer cases than ordered shows a shortage  
**Acceptance scenarios:** see ### US-5.2 under Acceptance Criteria

### US-5.3: Put items away
**As a** receiver (stocker)  
**I want to** record the warehouse location for received items  
**So that** stock is easy to find later

**Priority:** P1  
**Independent test:** Put-away saves a location and received date  
**Acceptance scenarios:** see ### US-5.3 under Acceptance Criteria

### US-5.4: Close a short order
**As a** office worker  
**I want to** close a supplier order that will not be filled completely  
**So that** quantity on order is accurate and the item can be reordered

**Priority:** P2  
**Independent test:** Closing a Partial order releases the missing quantity from quantity on order  
**Acceptance scenarios:** see ### US-5.4 under Acceptance Criteria

---

## Requirements

### Functional Requirements

- **FR-001**: When an item's **inventory position** (quantity on hand + quantity on order) falls below its minimum, the system MUST create a supplier order automatically.
- **FR-002**: An automatic order MUST bring the inventory position up to the item's maximum, rounded **up** to whole cases.
- **FR-003**: The system MUST NOT create another automatic order when quantity on hand + quantity on order is already at or above the minimum.
- **FR-004**: Receiving MUST show ordered versus received quantity for each line, including the number of cases short or over.
- **FR-005**: Received items MUST be saved with an aisle (A-T), a side (North or South), and a received date.
- **FR-006**: The cases received on a line MAY be split across more than one location; each location becomes its own stock lot.
- **FR-007**: Receiving MUST increase quantity on hand by the units received and decrease quantity on order by the units received, but never below 0.
- **FR-008**: Supplier order status MUST follow the allowed transitions in [ADR-0007](../docs/adr/0007-order-status-state-machines.md): `Open → Partial → Received`, `Open → Received`, and `Open/Partial → Closed`.
- **FR-009**: Only Receivers and Managers MUST be able to record receipts; only Office Workers and Managers MUST be able to close an order.

---

## Assumptions

- The reorder check runs whenever an item's quantity on hand goes **down**: when a pick is recorded (Feature 6) or after a manager adjustment (Feature 2). There is no nightly batch job.
- Items with no `minQty`/`maxQty` are never auto-ordered.
- If the item's supplier already has an `Open` automatic order created **today**, the new line is added to that order instead of starting a new one. This gives one order per supplier per day, which is easier to fax.
- The office faxes or phones the order using the **Print Order** view. The system does not send it.
- Units received = cases received × the item's `unitsPerCase`.
- An inventory quantity is defined once in [ADR-0008](../docs/adr/0008-inventory-stock-lots-fifo.md): `qtyOnHand` = the sum of the item's stock lots.

## Edge Cases

- Reorder rounding: min 50, max 200, 30 on hand, nothing on order, 24 per case → need 170 units → **8 cases** (192 units), and `qtyOnOrder` increases by 192.
- More cases arrive than were ordered (overage) → accepted; the screen shows **"2 cases over"**; `qtyOnOrder` only decreases by the units that were still on order.
- A second delivery for a `Partial` order → `casesReceived` adds up; the status becomes `Received` when every line is complete.
- Put-away cases do not add up to cases received on a line → `400` with `{ "message": "Put-away cases must equal cases received." }`.
- Aisle outside A–T or side not North/South → `400` with `{ "message": "Location must be aisle A–T and side North or South." }`.
- Receiving against a `Received` or `Closed` order → `400` with `{ "message": "Order is not open for receiving." }`.
- Closing an order releases the units still on order (`casesOrdered − casesReceived` × units per case) from `qtyOnOrder`, then runs the reorder check for those items.
- An item's supplier changes while it is on an open order → the open order stays with the original supplier.

## Success Criteria

- **SC-001**: Every Gherkin scenario has at least one automated test before merge.
- **SC-002**: After any receipt or close, every affected item has `qtyOnHand` equal to the sum of its stock lots (checked in tests).
- **SC-003**: **0** duplicate automatic orders are created when the inventory position is already at or above the minimum.
- **SC-004**: A receiver can record a 10-line delivery in **≤ 5 minutes**.

---

## Data Ownership & Isolation

Supplier orders and stock lots are shared warehouse data; access is controlled by **role**.

| Rule | Requirement |
|------|-------------|
| **Read scope** | Office, Receiver, and Manager can view supplier orders. Pickers read stock lots only through the pick list (Feature 6). |
| **Write scope** | Receiver and Manager record receipts and put-away. Office and Manager close orders. Automatic orders are created by the system, not by an endpoint. |
| **Audit** | Each receipt stores `receivedBy = req.user.id`. Automatic orders store `automatic = true` and `createdBy = null`. |
| **Wrong role** | `403`. |
| **Not found** | `404` with `{ "message": "Supplier order with id=<id> not found." }`. |
| **Consistency** | A receipt (lines, stock lots, item quantities, status) is saved in **one database transaction**; if any part fails, nothing is saved. |
| **Implementation** | `app/services/reorderService.js` exports `checkReorder(itemId, transaction)`; `app/services/inventoryService.js` owns stock-lot changes. Controllers call these services instead of updating quantities directly. |

---

## API Requirements

| Method | Endpoint | Auth | Roles | Purpose |
|--------|----------|------|-------|---------|
| `GET` | `/api/supplier-orders?status=` | Yes | office, receiver, manager | List orders, newest first; filter by status |
| `GET` | `/api/supplier-orders/:id` | Yes | office, receiver, manager | One order with supplier, lines, and items |
| `POST` | `/api/supplier-orders/:id/receipts` | Yes | receiver, manager | Record a delivery and put-away |
| `POST` | `/api/supplier-orders/:id/close` | Yes | office, manager | Close an `Open` or `Partial` order |
| `GET` | `/api/items/:id/stock-lots` | Yes | office, receiver, picker, manager | Stock lots for an item, oldest first |

**Supplier order response** (`200`):
```json
{
  "id": 31,
  "supplierId": 3,
  "supplierName": "Big Fizz Bottling",
  "orderDate": "2026-10-28",
  "status": "Partial",
  "automatic": true,
  "lines": [
    {
      "id": 77,
      "itemId": 11,
      "description": "Cola 12 oz can",
      "unitsPerCase": 24,
      "casesOrdered": 10,
      "casesReceived": 8,
      "casesShort": 2,
      "casesOver": 0
    }
  ]
}
```

**Receipt request body:**
```json
{
  "lines": [
    {
      "lineId": 77,
      "casesReceived": 8,
      "putAways": [
        { "aisle": "C", "side": "North", "cases": 5 },
        { "aisle": "D", "side": "South", "cases": 3 }
      ]
    }
  ]
}
```
`casesReceived` is the number of cases in **this** delivery. The server adds it to the line's running total.

**Receipt response** (`201`): the updated supplier order (same shape as above) plus `stockLots` created:
```json
{ "order": { "id": 31, "status": "Partial", "lines": [ "…" ] }, "stockLots": [ { "id": 140, "itemId": 11, "aisle": "C", "side": "North", "qty": 120, "receivedDate": "2026-10-28" } ] }
```

**Close response** (`200`): the order with `status: "Closed"`.

**Errors:**

| Case | Status | Message |
|------|--------|---------|
| Put-away does not match cases | `400` | `Put-away cases must equal cases received.` |
| Bad location | `400` | `Location must be aisle A–T and side North or South.` |
| Order not open | `400` | `Order is not open for receiving.` |
| Closing a Received/Closed order | `400` | `Only Open or Partial orders can be closed.` |

**Automatic ordering (no endpoint):** `checkReorder(itemId)` runs inside the same transaction as any decrease to `qtyOnHand` ([ADR-0006](../docs/adr/0006-automatic-reorder-policy.md)):

```text
position = qtyOnHand + qtyOnOrder
if minQty and maxQty are set and position < minQty:
    cases = ceil((maxQty - position) / unitsPerCase)
    add line to today's Open automatic order for the supplier (or create one)
    qtyOnOrder += cases * unitsPerCase
```

---

## Screen Requirements

UI images: [docs/ui/feature-5/](../docs/ui/feature-5/) (Supplier Order List, Receive Order, Print Order)

### [View: Supplier Orders] — route name `supplier-orders`, path `/supplier-orders`
*   Heading: **Supplier Orders**
*   Status filter chips: **Open**, **Partial**, **Received**, **Closed**, **All** (default: Open + Partial)
*   `<v-data-table>` columns: **Order #**, **Supplier**, **Order Date**, **Lines**, **Status** (colored chip), **Auto** (icon when automatic), actions
*   Row actions:
    *   **Receive** (receiver, manager; only `Open`/`Partial`) → `supplier-order-receive`
    *   **Print order** → `supplier-order-print`
    *   **Close order** (office, manager; only `Open`/`Partial`) → confirm dialog
*   Empty state: **"No supplier orders. Orders are created automatically when items run low."**

### [View: Receive Order] — route name `supplier-order-receive`, path `/supplier-orders/:id/receive`
*   Header: **Receive Order #{id} — {supplierName}**, with the status chip
*   Table, one row per line: **Item**, **Cases Ordered**, **Already Received**, **Cases Received Now** (number input), **Difference** (shows **"{n} cases short"** in orange or **"{n} cases over"** in blue), **Put Away**
*   **Put Away** cell: **Aisle** select (A–T) + **Side** select (North/South) + **Cases**; a **+ Split location** link adds another location row for the same line.
*   Footer actions: **Save Receipt** (`oc-cta`), **Cancel**
*   After saving: a success snackbar **"Receipt saved."** and the updated status chip.
*   Validation errors are shown inline on the row, with the API message in a `<v-alert type="error">`.

### [View: Print Order] — route name `supplier-order-print`, path `/supplier-orders/:id/print`
*   Print-friendly page (no MenuBar when printed): warehouse name, supplier name/address/fax, order #, date, and a table of **Item**, **Case UPC**, **Cases**
*   Action: **Print** (calls `window.print()`)

### [Dialog: Close Order]
*   Text: **"Close order #{id}? Cases not yet received will be removed from quantity on order."**
*   Actions: **Close Order**, **Cancel**

---

## Key Entities

- **Supplier Order**: an order placed with one supplier.
- **Supplier Order Line**: an item and number of cases on a supplier order.
- **Receipt**: one delivery recorded against an order (who and when).
- **Stock Lot**: quantity of an item at one location with a received date.

---

## Data Model Requirements

### `supplier_orders` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `supplierId` | INTEGER FK | Required, references `suppliers.id` |
| `supplierName` | STRING(100) | Copied from the supplier when the order is created (keeps history) |
| `orderDate` | DATEONLY | Required |
| `status` | ENUM | Open, Partial, Received, Closed; default Open |
| `automatic` | BOOLEAN | True when created by the system |
| `closedBy` | INTEGER FK | Nullable; references `users.id` |
| `createdAt` / `updatedAt` | DATE | Sequelize timestamps |

### `supplier_order_lines` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `orderId` | INTEGER FK | Required; references `supplier_orders.id`; cascade delete |
| `itemId` | INTEGER FK | Required; references `items.id` |
| `casesOrdered` | INTEGER | Required, > 0 |
| `casesReceived` | INTEGER | Default 0; running total |

### `receipts` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `orderId` | INTEGER FK | Required |
| `receivedBy` | INTEGER FK | Required; `req.user.id` |
| `createdAt` | DATE | Receipt date and time |

### `stock_lots` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `itemId` | INTEGER FK | Required |
| `receiptId` | INTEGER FK | Nullable (null for manager adjustments) |
| `aisle` | CHAR(1) | A through T |
| `side` | ENUM | North, South |
| `qty` | INTEGER | Units remaining, ≥ 0; a lot at 0 is kept for history and hidden from pick lists |
| `receivedDate` | DATEONLY | Oldest is picked first |

### Associations
*   `Supplier hasMany SupplierOrder`; `SupplierOrder hasMany SupplierOrderLine`; `SupplierOrderLine belongsTo Item`
*   `SupplierOrder hasMany Receipt`; `Receipt hasMany StockLot`; `Item hasMany StockLot`

---

## Acceptance Criteria (Gherkin)

### US-5.1 — Order automatically when stock is low

#### Scenario: Low stock triggers an automatic order
*   **Given** an item has a minimum of 50, a maximum of 200, 10 units per case, and 60 on hand with none on order
*   **When** the quantity on hand drops to 30
*   **Then** the system creates a supplier order for 17 cases (170 units)
*   **And** the item's quantity on order increases by 170

#### Scenario: Automatic order rounds up to whole cases
*   **Given** an item has a minimum of 50, a maximum of 200, 24 units per case, and 30 on hand with none on order
*   **When** the system checks stock levels
*   **Then** the supplier order line is for 8 cases
*   **And** the item's quantity on order increases by 192

#### Scenario: No duplicate automatic order
*   **Given** an item is below its minimum and already has 170 on order
*   **When** the system checks stock levels
*   **Then** no new supplier order is created

### US-5.2 — Check a delivery

#### Scenario: Delivery matches the order
*   **Given** a supplier order for 10 cases
*   **When** I receive 10 cases
*   **Then** the order status is `Received`
*   **And** quantity on hand increases and quantity on order decreases

#### Scenario: Delivery is short
*   **Given** a supplier order for 10 cases
*   **When** I receive 8 cases
*   **Then** the screen shows 2 cases short
*   **And** the order status is `Partial`

#### Scenario: Picker cannot record a receipt
*   **Given** I am signed in as a picker
*   **When** I send `POST /api/supplier-orders/31/receipts`
*   **Then** the API returns `403`

### US-5.3 — Put items away

#### Scenario: Receiver puts received items away
*   **Given** 8 cases were received
*   **When** I put them in aisle `C` on the `North` side
*   **Then** a stock lot is saved with that location and today's received date

#### Scenario: Receiver splits a line across two locations
*   **Given** 8 cases of an item with 24 units per case were received
*   **When** I put 5 cases in aisle `C` `North` and 3 cases in aisle `D` `South`
*   **Then** two stock lots are saved with 120 and 72 units

#### Scenario: Put-away must match cases received
*   **Given** I am receiving 8 cases on a line
*   **When** I enter put-away locations totaling 6 cases
*   **Then** the API returns `400` with `{ "message": "Put-away cases must equal cases received." }`

### US-5.4 — Close a short order

#### Scenario: Office worker closes a partial order
*   **Given** a `Partial` order line has 10 cases ordered and 8 received, with 24 units per case
*   **When** I close the order
*   **Then** the order status is `Closed`
*   **And** the item's quantity on order decreases by 48

---

## Definition of Done

*   [ ] Backend and frontend implemented per this spec (**FR-001 – FR-009** satisfied)
*   [ ] **Success Criteria (SC-001 – SC-004)** met
*   [ ] `reorderService` and `inventoryService` used by Features 2 and 6, with no direct quantity updates in controllers
*   [ ] Receipts run in one transaction (a test proves a failed put-away leaves no partial changes)
*   [ ] All Gherkin scenarios have tests, and `npm test` passes
*   [ ] `features/reference/api.md`, `data-model.md`, and `behavior.md` updated (status rules and reorder formula)
*   [ ] Merged to `dev` through a pull request

---

## Out of Scope

*   Manually created supplier orders. All orders are automatic in this release; a manual order is a candidate operational feature for the final requirements draft.
*   Sending orders to suppliers electronically (fax/EDI/email)
*   Supplier invoices, costs, and payment matching
*   Returning damaged goods to suppliers
*   Moving stock between locations after put-away (a candidate operational feature)
