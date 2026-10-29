# Feature: Supplier Ordering and Receiving

**Feature ID:** 5
**Branch pattern:** `feature/5-supplier-ordering-receiving`
**Status:** Draft
**Created:** 2026-09-29
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
**As a** stocker  
**I want to** record the warehouse location for received items  
**So that** stock is easy to find later

**Priority:** P1  
**Independent test:** Put-away saves a location and received date  
**Acceptance scenarios:** see ### US-5.3 under Acceptance Criteria

---

## Requirements

### Functional Requirements

- **FR-001**: When an item's quantity on hand falls below its minimum, the system MUST create a supplier order automatically.
- **FR-002**: An automatic order MUST bring the item up to its maximum, counting what is already on order.
- **FR-003**: The system MUST NOT create another automatic order when enough is already on order.
- **FR-004**: Receiving MUST show ordered versus received quantity for each line.
- **FR-005**: Received items MUST be saved with an aisle (A-T), a side (North or South), and a received date.

---

## Key Entities

- **Supplier Order**: an order placed with one supplier.
- **Order Line**: an item and quantity on a supplier order.
- **Stock Lot**: quantity of an item at one location with a received date.

---

## Data Model Requirements

### `supplier_orders` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `supplierId` | INTEGER FK | Required, references `suppliers.id` |
| `orderDate` | DATE | Required |
| `status` | ENUM | Open, Partial, Received |
| `automatic` | BOOLEAN | True when created by the system |


### `supplier_order_lines` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `orderId` | INTEGER FK | Required |
| `itemId` | INTEGER FK | Required |
| `casesOrdered` | INTEGER | Required |
| `casesReceived` | INTEGER | Default 0 |


### `stock_lots` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `itemId` | INTEGER FK | Required |
| `aisle` | CHAR(1) | A through T |
| `side` | ENUM | North, South |
| `qty` | INTEGER | Units |
| `receivedDate` | DATE | Oldest is picked first |


---

## Acceptance Criteria (Gherkin)

### US-5.1 — Order automatically when stock is low

#### Scenario: Low stock triggers an automatic order
*   **Given** an item has a minimum of 50, a maximum of 200, and 60 on hand with none on order
*   **When** the quantity on hand drops to 30
*   **Then** the system creates a supplier order for 170 units
*   **And** the item's quantity on order increases by 170

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

### US-5.3 — Put items away

#### Scenario: Stocker puts received items away
*   **Given** 8 cases were received
*   **When** I put them in aisle `C` on the `North` side
*   **Then** a stock lot is saved with that location and today's received date
