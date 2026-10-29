# Feature: Customer Ordering and Picking

**Feature ID:** 6
**Branch pattern:** `feature/6-customer-ordering-picking`
**Status:** Draft
**Created:** 2026-09-29
**Updated:** 2026-10-28 (design draft)
**Input:** Stores place orders in the app instead of on paper, and pickers fill each order from a location-sorted pick list
**Depends on:** [Feature 2 — Item Management](feature-2-item-management.md), [Feature 4 — Customer Management](feature-4-customer-management.md), [Feature 5 — Supplier Ordering and Receiving](feature-5-supplier-ordering-receiving.md)

---

## User Stories

### US-6.1: Place an order
**As a** customer  
**I want to** place my weekly order in the app  
**So that** I no longer use the paper order book

**Priority:** P1  
**Independent test:** Submitting an order creates it with status New  
**Acceptance scenarios:** see ### US-6.1 under Acceptance Criteria

### US-6.2: See when an item is out of stock
**As a** customer  
**I want to** see an out of stock message for items we cannot supply  
**So that** I do not order something that cannot ship

**Priority:** P1  
**Independent test:** Adding an out-of-stock item is blocked with a message  
**Acceptance scenarios:** see ### US-6.2 under Acceptance Criteria

### US-6.3: Get a pick list
**As a** order picker  
**I want to** get a pick list sorted by aisle and side with the oldest stock first  
**So that** I fill orders quickly and rotate stock

**Priority:** P1  
**Independent test:** Starting an order creates a sorted pick list  
**Acceptance scenarios:** see ### US-6.3 under Acceptance Criteria

### US-6.4: Record what was picked
**As a** order picker  
**I want to** record the quantity I actually picked  
**So that** shortages are recorded

**Priority:** P1  
**Independent test:** Picking less than ordered shows a shortage  
**Acceptance scenarios:** see ### US-6.4 under Acceptance Criteria

### US-6.5: Cancel an order before picking
**As a** customer  
**I want to** cancel an order that has not started picking  
**So that** I can fix a mistake

**Priority:** P2  
**Independent test:** A New order can be cancelled; a Picking order cannot  
**Acceptance scenarios:** see ### US-6.5 under Acceptance Criteria

---

## Requirements

### Functional Requirements

- **FR-001**: A customer MUST only create and view orders for their own store.
- **FR-002**: An item that is out of stock MUST NOT be added to an order and MUST show the message "Out of stock".
- **FR-003**: A new order MUST start with status New; other statuses are Picking, Picked, Shipped, and Cancelled. Status changes MUST follow [ADR-0007](../docs/adr/0007-order-status-state-machines.md).
- **FR-004**: A pick list MUST be sorted by aisle and side and MUST use the oldest stock first.
- **FR-005**: Recording a pick MUST reduce quantity on hand by the quantity picked.
- **FR-006**: The system MUST record the quantity picked on each order line.
- **FR-007**: The shortage on an order line MUST equal quantity ordered − quantity picked, and MUST be shown to the picker and the customer.
- **FR-008**: Starting to pick MUST require a bin number.
- **FR-009**: An order MUST only become Picked after every pick-list line has a recorded quantity.
- **FR-010**: Only `New` orders MAY be cancelled, by the customer who owns them, an Office Worker, or a Manager.
- **FR-011**: After each recorded pick, the Feature 5 reorder check MUST run for that item.

---

## Assumptions

- **Out of stock** means `qtyOnHand = 0`. Customers see only an in-stock or out-of-stock flag, never exact quantities.
- Placing an order does **not** reserve stock. If stock runs out between ordering and picking, the line is picked short. Reservations are out of scope.
- The cart (items chosen before **Submit Order**) is kept only in the browser until it is submitted.
- A submitted order cannot be edited; the customer cancels it (while `New`) and places a new one.
- Pick-list allocation and put-away locations follow [ADR-0008](../docs/adr/0008-inventory-stock-lots-fifo.md).

## Edge Cases

- Order with no lines → `400` with `{ "message": "Order must have at least one item." }`.
- Quantity 0, negative, or not a whole number → `400` with `{ "message": "Quantity must be a whole number greater than 0." }`.
- The same item twice in one order → `400` with `{ "message": "Each item can appear only once." }`.
- An item goes out of stock while it is in the cart → submit returns `400` with `{ "message": "Out of stock: Cola 12 oz can." }`; the UI removes that line and shows **Out of stock**.
- An order line for 12 when only 8 units exist in all lots → the pick list shows 8 to pick and the line is already **4 short** before picking starts.
- An item with no stock at start of picking → the pick-list line shows **No stock** with 0 to pick; recording 0 is required to finish.
- A picker records more than the pick-list line asks for → `400` with `{ "message": "Picked quantity cannot be more than the quantity to pick." }`.
- The lot has less than expected when the picker arrives (another order took it) → the picker records the real amount; the difference shows as a shortage.
- Cancelling a `Picking`, `Picked`, or `Shipped` order → `400` with `{ "message": "Only New orders can be cancelled." }`.
- Finishing before every line is recorded → `400` with `{ "message": "Record every line before finishing." }`.
- Starting to pick an order that is not `New` → `400` with `{ "message": "Only New orders can start picking." }`.

## Success Criteria

- **SC-001**: Every Gherkin scenario has at least one automated test before merge.
- **SC-002**: **0** cross-store order reads or writes in automated tests (customer A can never see customer B's orders).
- **SC-003**: Pick lists are always in aisle, then side, then oldest-received order (checked by API tests with 3+ lots).
- **SC-004**: After any recorded pick, `qtyOnHand` equals the sum of the item's stock lots.
- **SC-005**: A customer can place a 20-line order in **≤ 5 minutes**.

---

## Data Ownership & Isolation

Customer orders are **owned by a store**. This is the main multi-user boundary in the system.

| Rule | Requirement |
|------|-------------|
| **Read scope (customer)** | `GET /api/customer-orders` returns only rows where `customerId = req.user.customerId`. |
| **Read scope (staff)** | Office, Picker, and Manager can read all orders. Drivers read orders through Feature 7 only. |
| **Create scope** | `customerId` is set from `req.user.customerId`; a `customerId` in the body is ignored. |
| **Cross-store access** | A customer requesting another store's order by id → `404` (never `403`). |
| **Write scope** | A customer can cancel only their own `New` orders. Picking actions require role picker or manager. |
| **Wrong role** | `403` (for example, a customer calling `start-picking`). |
| **Catalog** | Customers use `GET /api/catalog`, which returns no costs or quantities. |
| **Implementation** | `getAccessibleOrderOrNull(req, orderId)` in `app/authorization/` applies the customer scope; picking uses `inventoryService` and `reorderService` from Feature 5 in one transaction per recorded pick. |

---

## API Requirements

| Method | Endpoint | Auth | Roles | Purpose |
|--------|----------|------|-------|---------|
| `GET` | `/api/catalog?search=` | Yes | customer | Items with description, unit UPC, price, `inStock` |
| `GET` | `/api/customer-orders?status=` | Yes | customer (own), office, picker, manager | List orders, newest first |
| `GET` | `/api/customer-orders/:id` | Yes | customer (own), office, picker, manager | One order with lines, picked quantities, and shortages |
| `POST` | `/api/customer-orders` | Yes | customer | Place an order (status `New`) |
| `POST` | `/api/customer-orders/:id/cancel` | Yes | customer (own), office, manager | Cancel a `New` order |
| `POST` | `/api/customer-orders/:id/start-picking` | Yes | picker, manager | Set bin, build the pick list, status → `Picking` |
| `GET` | `/api/customer-orders/:id/pick-list` | Yes | picker, manager | Sorted pick-list lines |
| `PUT` | `/api/pick-list-lines/:id` | Yes | picker, manager | Record the quantity picked on one line |
| `POST` | `/api/customer-orders/:id/finish-picking` | Yes | picker, manager | Status → `Picked` |

**Catalog item** (`200`, array):
```json
[{ "id": 11, "description": "Cola 12 oz can", "unitUpc": "012345678905", "price": "1.25", "inStock": true }]
```

**Place order request body:**
```json
{ "lines": [ { "itemId": 11, "qty": 24 }, { "itemId": 15, "qty": 12 } ] }
```

**Order response** (`200` / `201`):
```json
{
  "id": 42,
  "customerId": 12,
  "storeName": "Quick Stop #12",
  "orderDate": "2026-10-28",
  "status": "New",
  "binNumber": null,
  "lines": [
    { "id": 301, "itemId": 11, "description": "Cola 12 oz can", "qty": 24, "qtyPicked": null, "shortage": null }
  ]
}
```

**Start picking request body:**
```json
{ "binNumber": "B-07" }
```

**Pick list response** (`200`):
```json
{
  "orderId": 42,
  "binNumber": "B-07",
  "lines": [
    { "id": 901, "orderLineId": 301, "description": "Cola 12 oz can", "unitUpc": "012345678905", "aisle": "B", "side": "North", "receivedDate": "2026-10-01", "qtyToPick": 12, "qtyPicked": null },
    { "id": 902, "orderLineId": 301, "description": "Cola 12 oz can", "unitUpc": "012345678905", "aisle": "C", "side": "South", "receivedDate": "2026-10-15", "qtyToPick": 12, "qtyPicked": null }
  ]
}
```

**Record pick request body:**
```json
{ "qtyPicked": 10 }
```
Response (`200`): the pick-list line plus the order line's new `qtyPicked` and `shortage`.

**Errors:** see Edge Cases. All use `{ "message": "…" }` with `400`, `403`, or `404` per the [Feature 1 contract](feature-1-user-login.md#api-requirements).

---

## Screen Requirements

UI images: [docs/ui/feature-6/](../docs/ui/feature-6/) (My Orders, New Order, Order Detail, Pick Queue, Pick List)

### [View: My Orders] — route name `my-orders`, path `/my-orders` (customer)
*   Heading: **My Orders — {storeName}**
*   Primary action: **+ New Order** (`oc-cta`) → `order-new`
*   Table: **Order #**, **Date**, **Items**, **Status** chip; click a row → `order-detail`
*   Empty state: **"No orders yet. Place your first order."**

### [View: New Order] — route name `order-new`, path `/my-orders/new` (customer)
*   Two columns on desktop, stacked on mobile:
    *   **Catalog** (left): search box **Search items**; table **Description**, **UPC**, **Price**, **Qty** input, **Add** button. Out-of-stock rows show a red **Out of stock** chip, and **Add** is disabled.
    *   **Order** (right): cart lines with **Qty** (editable) and **Remove**; line count and estimated total
*   Actions: **Submit Order** (`oc-cta`, disabled when the cart is empty), **Cancel**
*   After submit: navigate to `order-detail` with snackbar **"Order #{id} placed."**

### [View: Order Detail] — route name `order-detail`, path `/orders/:id` (customer own, office, picker, manager)
*   Header: **Order #{id}**, store name, date, status chip, bin number (once picking)
*   Lines table: **Item**, **Ordered**, **Picked**, **Short** (shown in orange when > 0)
*   Action: **Cancel Order** (only when `New`, and only for the owner customer, office, or manager) → confirm dialog **"Cancel order #{id}?"**

### [View: Customer Orders] — route name `customer-orders`, path `/customer-orders` (office, manager)
*   Heading: **Customer Orders**; status filter chips; table **Order #**, **Store**, **Route**, **Date**, **Status**; click a row → `order-detail`

### [View: Pick Queue] — route name `pick-queue`, path `/picking` (picker, manager)
*   Two tabs: **New** (oldest first) and **In Progress** (status `Picking`)
*   Table: **Order #**, **Store**, **Route**, **Date**, **Lines**
*   **New** row action: **Start Picking** → dialog with a **Bin Number** field (required) → **Start** → navigate to `pick-list`
*   **In Progress** row action: **Continue** → `pick-list`

### [View: Pick List] — route name `pick-list`, path `/picking/:orderId`
*   Header: **Pick Order #{id} — {storeName}**, plus **Bin {binNumber}**
*   Table in pick order: **Aisle**, **Side**, **Item**, **UPC**, **Received**, **To Pick**, **Picked** (number input), **Short**, and a **Record** button per row
*   Recorded rows show a green check. Rows with **No stock** show a grey chip.
*   Footer: progress **"{n} of {total} lines recorded"** and **Finish Picking** (`oc-cta`, enabled only when every row is recorded)
*   After finishing: back to `pick-queue` with snackbar **"Order #{id} picked."**

---

## Key Entities

- **Customer Order**: an order from one store.
- **Customer Order Line**: an item and quantity on an order, with the quantity picked.
- **Pick List Line**: one stop for the picker: an order line, a stock lot (location), and the quantity to take from it.
- **Stock Lot**: the location and age of stock (Feature 5).

---

## Data Model Requirements

### `customer_orders` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `customerId` | INTEGER FK | Required; set from the signed-in customer |
| `orderDate` | DATEONLY | Required; set by the server |
| `status` | ENUM | New, Picking, Picked, Shipped, Cancelled; default New |
| `binNumber` | STRING(10) | Required when status becomes Picking |
| `pickedBy` | INTEGER FK | Nullable; the picker who finished; references `users.id` |
| `createdAt` / `updatedAt` | DATE | Sequelize timestamps |

### `customer_order_lines` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `orderId` | INTEGER FK | Required; cascade delete |
| `itemId` | INTEGER FK | Required; unique per order |
| `qty` | INTEGER | Required, greater than 0 |
| `qtyPicked` | INTEGER | Null until picked; sum of its pick-list lines |

### `pick_list_lines` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `orderLineId` | INTEGER FK | Required; references `customer_order_lines.id` |
| `stockLotId` | INTEGER FK | Nullable (null = **No stock**) |
| `aisle` | CHAR(1) | Copied from the lot when the list is built |
| `side` | ENUM | North, South |
| `receivedDate` | DATEONLY | Copied from the lot |
| `qtyToPick` | INTEGER | ≥ 0 |
| `qtyPicked` | INTEGER | Null until recorded; ≤ `qtyToPick` |
| `pickedBy` | INTEGER FK | Set from `req.user.id` when recorded |

### Associations
*   `Customer hasMany CustomerOrder`; `CustomerOrder hasMany CustomerOrderLine`; `CustomerOrderLine belongsTo Item`
*   `CustomerOrderLine hasMany PickListLine`; `PickListLine belongsTo StockLot`

---

## Acceptance Criteria (Gherkin)

### US-6.1 — Place an order

#### Scenario: Customer places an order
*   **Given** I am signed in as a customer
*   **When** I add an in-stock item and submit the order
*   **Then** the API returns `201` with status `New`
*   **And** the order belongs to my store

#### Scenario: Customer cannot place an order for another store
*   **Given** I am signed in as customer A
*   **When** I send `POST /api/customer-orders` with `"customerId"` set to customer B
*   **Then** the API returns `201`
*   **And** the order belongs to customer A

### US-6.2 — See when an item is out of stock

#### Scenario: Item is out of stock
*   **Given** an item has 0 on hand and more is on order from the supplier
*   **When** I try to add the item to my order
*   **Then** I see the message "Out of stock"
*   **And** the item is not added to my order

### US-6.3 — Get a pick list

#### Scenario: Pick list is sorted by location
*   **Given** an order has items in aisles `B` and `C`
*   **When** I start picking the order
*   **Then** the pick list shows aisle `B` items before aisle `C` items
*   **And** the order status is `Picking`

#### Scenario: Oldest stock is picked first
*   **Given** an item has a lot received `2026-10-01` in aisle `D` and a lot received `2026-10-15` in aisle `B`
*   **And** an order asks for fewer units than the older lot holds
*   **When** I start picking the order
*   **Then** the pick list takes the units from the `2026-10-01` lot in aisle `D`

#### Scenario: Bin number is required
*   **Given** a `New` order
*   **When** I start picking without a bin number
*   **Then** the API returns `400` with `{ "message": "Bin number is required." }`

### US-6.4 — Record what was picked

#### Scenario: Picker records a short pick
*   **Given** an order line asks for 12 units
*   **When** I record 10 units picked
*   **Then** the screen shows a shortage of 2
*   **And** quantity on hand decreases by 10

#### Scenario: Finish picking
*   **Given** every pick-list line for an order has been recorded
*   **When** I click **Finish Picking**
*   **Then** the order status is `Picked`

#### Scenario: Cannot finish with unrecorded lines
*   **Given** one pick-list line has not been recorded
*   **When** I finish picking
*   **Then** the API returns `400` with `{ "message": "Record every line before finishing." }`

### US-6.5 — Cancel an order before picking

#### Scenario: Customer cancels a new order
*   **Given** I own an order with status `New`
*   **When** I click **Cancel Order** and confirm
*   **Then** the order status is `Cancelled`

#### Scenario: Order being picked cannot be cancelled
*   **Given** an order has status `Picking`
*   **When** I try to cancel it
*   **Then** the API returns `400` with `{ "message": "Only New orders can be cancelled." }`

---

## Definition of Done

*   [ ] Backend and frontend implemented per this spec (**FR-001 – FR-011** satisfied)
*   [ ] **Success Criteria (SC-001 – SC-005)** met
*   [ ] Customer scoping uses `getAccessibleOrderOrNull`, with cross-store tests returning `404`
*   [ ] Each recorded pick updates lot, item, and order line in one transaction and calls `checkReorder`
*   [ ] All Gherkin scenarios have tests, and `npm test` passes
*   [ ] `features/reference/api.md`, `data-model.md`, and `behavior.md` updated
*   [ ] Merged to `dev` through a pull request

---

## Out of Scope

*   Editing an order after it is submitted
*   Stock reservations or back orders
*   Substituting a different item for a short line
*   Recurring "same as last week" orders (a candidate for the final requirements draft)
*   Invoices and payment
*   Handheld scanner integration for pickers
