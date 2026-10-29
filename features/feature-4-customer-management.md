# Feature: Customer Management

**Feature ID:** 4
**Branch pattern:** `feature/4-customer-management`
**Status:** Draft
**Created:** 2026-09-29
**Input:** Office workers keep store records and assign each store to a truck route
**Depends on:** [Feature 1 — User Login and Roles](feature-1-user-login.md)

---

## User Stories

### US-4.1: Add and edit customers
**As a** office worker  
**I want to** add, edit, and delete customers  
**So that** every store that orders from us is on file

**Priority:** P1  
**Independent test:** Create a customer and see it in the list  
**Acceptance scenarios:** see ### US-4.1 under Acceptance Criteria

### US-4.2: Assign a truck route
**As a** office worker  
**I want to** assign each customer to a truck route  
**So that** deliveries are organized by route

**Priority:** P1  
**Independent test:** Customer shows its assigned route  
**Acceptance scenarios:** see ### US-4.2 under Acceptance Criteria

### US-4.3: Protect customers with open orders
**As a** office worker  
**I want to** be stopped from deleting a customer that has open orders  
**So that** order history stays correct

**Priority:** P2  
**Independent test:** Deleting a customer with an open order is refused  
**Acceptance scenarios:** see ### US-4.3 under Acceptance Criteria

---

## Requirements

### Functional Requirements

- **FR-001**: Each customer MUST have a store name and a delivery address.
- **FR-002**: A customer MAY be assigned to one truck route.
- **FR-003**: A customer with open orders MUST NOT be deleted.
- **FR-004**: Only Office Workers and Managers MUST be able to change customers.

---

## Key Entities

- **Customer**: a convenience store that orders from the warehouse.
- **Truck Route**: the delivery route a customer belongs to.

---

## Data Model Requirements

### `customers` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `storeName` | STRING | Required |
| `address` | STRING | Required |
| `phone` | STRING | Optional |
| `contactName` | STRING | Optional |
| `routeId` | INTEGER FK | Optional; references `truck_routes.id` |


---

## Acceptance Criteria (Gherkin)

### US-4.1 — Add and edit customers

#### Scenario: Office worker adds a customer
*   **Given** I am signed in as an office worker
*   **When** I add a customer with a store name and delivery address
*   **Then** the API returns `201`
*   **And** the customer appears in the customer list

#### Scenario: Address is required
*   **Given** I am adding a customer
*   **When** I leave the address empty
*   **Then** the API returns `400` with `{ "message": "Address is required." }`

### US-4.2 — Assign a truck route

#### Scenario: Office worker assigns a route
*   **Given** a customer and a route `Route 1` exist
*   **When** I assign the customer to `Route 1`
*   **Then** the customer shows `Route 1`

### US-4.3 — Protect customers with open orders

#### Scenario: Customer with open orders cannot be deleted
*   **Given** a customer has an open order
*   **When** I try to delete the customer
*   **Then** the API returns `409`
