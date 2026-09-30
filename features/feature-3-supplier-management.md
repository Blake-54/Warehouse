# Feature: Supplier Management

**Feature ID:** 3
**Branch pattern:** `feature/3-supplier-management`
**Status:** Draft
**Created:** 2026-09-29
**Input:** Office workers keep supplier contact information so orders go to the right place
**Depends on:** [Feature 1 — User Login and Roles](feature-1-user-login.md)

---

## User Stories

### US-3.1: Add and edit suppliers
**As a** office worker  
**I want to** add, edit, and delete suppliers  
**So that** I know who we buy from

**Priority:** P1  
**Independent test:** Create a supplier and see it in the list  
**Acceptance scenarios:** see ### US-3.1 under Acceptance Criteria

### US-3.2: Protect suppliers with open orders
**As a** office worker  
**I want to** be stopped from deleting a supplier that has open orders  
**So that** order history stays correct

**Priority:** P2  
**Independent test:** Deleting a supplier with an open order is refused  
**Acceptance scenarios:** see ### US-3.2 under Acceptance Criteria

---

## Requirements

### Functional Requirements

- **FR-001**: Each supplier MUST have a name.
- **FR-002**: A supplier MUST store an address and fax number.
- **FR-003**: Supplier names MUST be unique.
- **FR-004**: A supplier with open orders MUST NOT be deleted.
- **FR-005**: Only Office Workers and Managers MUST be able to change suppliers.

---

## Key Entities

- **Supplier**: a manufacturer the warehouse buys from.
- **Item**: products a supplier provides.

---

## Data Model Requirements

### `suppliers` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `name` | STRING | Required, unique |
| `address` | STRING | Optional |
| `phone` | STRING | Optional |
| `fax` | STRING | Optional |


---

## Acceptance Criteria (Gherkin)

### US-3.1 — Add and edit suppliers

#### Scenario: Office worker adds a supplier
*   **Given** I am signed in as an office worker
*   **When** I add a supplier with a name, address, and fax number
*   **Then** the API returns `201`
*   **And** the supplier appears in the supplier list

### US-3.2 — Protect suppliers with open orders

#### Scenario: Supplier with open orders cannot be deleted
*   **Given** a supplier has an open supplier order
*   **When** I try to delete the supplier
*   **Then** the API returns `409` with `{ "message": "Supplier has open orders." }`
