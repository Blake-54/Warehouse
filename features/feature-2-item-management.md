# Feature: Item Management

**Feature ID:** 2
**Branch pattern:** `feature/2-item-management`
**Status:** Draft
**Created:** 2026-09-29
**Input:** Office workers keep the list of items the warehouse sells, and managers set reorder levels
**Depends on:** [Feature 1 — User Login and Roles](feature-1-user-login.md)

---

## User Stories

### US-2.1: Add and edit items
**As a** office worker  
**I want to** add, edit, and delete items  
**So that** the catalog matches what we buy and sell

**Priority:** P1  
**Independent test:** Create an item and see it in the item list  
**Acceptance scenarios:** see ### US-2.1 under Acceptance Criteria

### US-2.2: Keep UPCs unique
**As a** office worker  
**I want to** be stopped from saving a UPC that another item already uses  
**So that** scanning always finds the right item

**Priority:** P1  
**Independent test:** Saving a duplicate UPC is rejected  
**Acceptance scenarios:** see ### US-2.2 under Acceptance Criteria

### US-2.3: Set reorder levels
**As a** manager  
**I want to** set a minimum and maximum quantity for each item  
**So that** the system knows when to reorder and how much

**Priority:** P1  
**Independent test:** Save levels and see them on the item  
**Acceptance scenarios:** see ### US-2.3 under Acceptance Criteria

### US-2.4: See quantities
**As a** office worker  
**I want to** see quantity on hand and quantity on order for each item  
**So that** I know what stock we have and what is coming

**Priority:** P2  
**Independent test:** Item list shows both quantities  
**Acceptance scenarios:** see ### US-2.4 under Acceptance Criteria

---

## Requirements

### Functional Requirements

- **FR-001**: Each item MUST have a description, case UPC, unit UPC, units per case, price, supplier cost, and supplier.
- **FR-002**: Case UPC and unit UPC MUST each be unique.
- **FR-003**: An item's minimum quantity MUST be less than its maximum quantity.
- **FR-004**: Quantity on hand MUST NOT be edited on the normal item form; only a Manager MUST be able to change it.
- **FR-005**: An item with stock on hand or open orders MUST NOT be deleted.

---

## Key Entities

- **Item**: a product sold to convenience stores.
- **Supplier**: the manufacturer the item is bought from.

---

## Data Model Requirements

### `items` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `description` | STRING | Required |
| `caseUpc` | STRING | Required, unique |
| `unitUpc` | STRING | Required, unique |
| `unitsPerCase` | INTEGER | Required, greater than 0 |
| `price` | DECIMAL | Selling price |
| `supplierCost` | DECIMAL | What the supplier charges us |
| `minQty` | INTEGER | Reorder level |
| `maxQty` | INTEGER | Order-up-to level |
| `qtyOnHand` | INTEGER | Default 0 |
| `qtyOnOrder` | INTEGER | Default 0 |
| `supplierId` | INTEGER FK | References `suppliers.id` |


---

## Acceptance Criteria (Gherkin)

### US-2.1 — Add and edit items

#### Scenario: Office worker adds an item
*   **Given** I am signed in as an office worker
*   **When** I add an item with a description, case UPC, unit UPC, units per case, and price
*   **Then** the API returns `201` with the item
*   **And** the item appears in the item list

### US-2.2 — Keep UPCs unique

#### Scenario: Duplicate UPC is rejected
*   **Given** an item exists with unit UPC `012345678905`
*   **When** I add another item with unit UPC `012345678905`
*   **Then** the API returns `400` with `{ "message": "UPC already in use." }`

### US-2.3 — Set reorder levels

#### Scenario: Manager sets reorder levels
*   **Given** I am signed in as a manager
*   **When** I set an item's minimum to 50 and maximum to 200
*   **Then** the levels are saved

#### Scenario: Minimum must be below maximum
*   **Given** I am editing an item
*   **When** I set the minimum to 200 and the maximum to 50
*   **Then** the API returns `400`

### US-2.4 — See quantities

#### Scenario: Item list shows quantities
*   **Given** an item has 30 on hand and 100 on order
*   **When** I open the item list
*   **Then** I see 30 on hand and 100 on order for that item
