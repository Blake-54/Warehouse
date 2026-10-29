# Feature: Customer Ordering and Picking

**Feature ID:** 6
**Branch pattern:** `feature/6-customer-ordering-picking`
**Status:** Draft
**Created:** 2026-09-29
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

---

## Requirements

### Functional Requirements

- **FR-001**: A customer MUST only create and view orders for their own store.
- **FR-002**: An item that is out of stock MUST NOT be added to an order and MUST show the message "Out of stock".
- **FR-003**: A new order MUST start with status New; other statuses are Picking, Picked, Shipped, and Cancelled.
- **FR-004**: A pick list MUST be sorted by aisle and side and MUST use the oldest stock first.
- **FR-005**: Recording a pick MUST reduce quantity on hand by the quantity picked.
- **FR-006**: The system MUST record the quantity picked on each order line.

---

## Key Entities

- **Customer Order**: an order from one store.
- **Order Line**: an item and quantity on an order.
- **Pick List**: the items and locations a picker collects for an order.

---

## Data Model Requirements

### `customer_orders` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `customerId` | INTEGER FK | Required; set from the signed-in customer |
| `orderDate` | DATE | Required |
| `status` | ENUM | New, Picking, Picked, Shipped, Cancelled |
| `binNumber` | STRING | Bin that holds the order while picking |


### `customer_order_lines` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `orderId` | INTEGER FK | Required |
| `itemId` | INTEGER FK | Required |
| `qty` | INTEGER | Required, greater than 0 |
| `qtyPicked` | INTEGER | Empty until picked |


---

## Acceptance Criteria (Gherkin)

### US-6.1 — Place an order

#### Scenario: Customer places an order
*   **Given** I am signed in as a customer
*   **When** I add an in-stock item and submit the order
*   **Then** the API returns `201` with status `New`
*   **And** the order belongs to my store

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

### US-6.4 — Record what was picked

#### Scenario: Picker records a short pick
*   **Given** an order line asks for 12 units
*   **When** I record 10 units picked
*   **Then** the screen shows a shortage of 2
*   **And** quantity on hand decreases by 10
