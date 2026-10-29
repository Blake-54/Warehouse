# Feature: Supplier Management

**Feature ID:** 3
**Branch pattern:** `feature/3-supplier-management`
**Status:** Draft
**Created:** 2026-09-29
**Updated:** 2026-10-28 (design draft)
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
- **FR-003**: Supplier names MUST be unique (case-insensitive).
- **FR-004**: A supplier with open orders MUST NOT be deleted.
- **FR-005**: Only Office Workers and Managers MUST be able to change suppliers.
- **FR-006**: A supplier that still has items assigned MUST NOT be deleted.

---

## Assumptions

- An **open** supplier order is one with status `Open` or `Partial` ([ADR-0007](../docs/adr/0007-order-status-state-machines.md)).
- Supplier orders are faxed or phoned in by the office from the printed order (Feature 5). The system does not send faxes.
- Address is a single text field (street, city, state, ZIP). A structured address is out of scope.

## Edge Cases

- Name empty or whitespace only → `400` with `{ "message": "Supplier name is required." }`.
- Name matches an existing supplier ignoring case and spaces (for example ` big fizz ` vs `Big Fizz`) → `400` with `{ "message": "Supplier name already in use." }`.
- Fax or address empty → `400` with `{ "message": "Address and fax are required." }`.
- Deleting a supplier with an `Open` or `Partial` order → `409` with `{ "message": "Supplier has open orders." }`.
- Deleting a supplier that still has items → `409` with `{ "message": "Supplier has items assigned." }`.
- Deleting a supplier whose orders are all `Received` and that has no items → allowed. Historical orders keep the supplier's name in `supplierName` on the order (Feature 5).

## Success Criteria

- **SC-001**: Every Gherkin scenario has at least one automated test before merge.
- **SC-002**: **0** suppliers with open orders or assigned items can be deleted (checked by API tests).
- **SC-003**: An office worker can add a supplier in **≤ 1 minute** from the supplier list.

---

## Data Ownership & Isolation

Suppliers are shared warehouse data; access is controlled by **role**.

| Rule | Requirement |
|------|-------------|
| **Read scope** | Office, Receiver, and Manager can list suppliers (Receiver needs names on supplier orders). |
| **Write scope** | Only Office and Manager can add, edit, or delete. |
| **Wrong role** | `403`. |
| **Not found** | `404` with `{ "message": "Supplier with id=<id> not found." }`. |
| **Implementation** | `requireRole("office", "manager")` on write routes; the delete checks run in the controller through a shared `hasOpenSupplierOrders(supplierId)` helper that Feature 5 also uses. |

---

## API Requirements

| Method | Endpoint | Auth | Roles | Purpose |
|--------|----------|------|-------|---------|
| `GET` | `/api/suppliers` | Yes | office, receiver, manager | List suppliers sorted by name |
| `GET` | `/api/suppliers/:id` | Yes | office, receiver, manager | One supplier |
| `POST` | `/api/suppliers` | Yes | office, manager | Add a supplier |
| `PUT` | `/api/suppliers/:id` | Yes | office, manager | Edit a supplier |
| `DELETE` | `/api/suppliers/:id` | Yes | office, manager | Delete a supplier with no open orders and no items |

**Create / edit request body:**
```json
{
  "name": "Big Fizz Bottling",
  "address": "100 Industrial Rd, Tulsa, OK 74107",
  "phone": "918-555-0100",
  "fax": "918-555-0101"
}
```

**Supplier response** (`200` / `201`):
```json
{ "id": 3, "name": "Big Fizz Bottling", "address": "100 Industrial Rd, Tulsa, OK 74107", "phone": "918-555-0100", "fax": "918-555-0101" }
```

**Errors:**

| Case | Status | Message |
|------|--------|---------|
| Missing name | `400` | `Supplier name is required.` |
| Duplicate name | `400` | `Supplier name already in use.` |
| Missing address or fax | `400` | `Address and fax are required.` |
| Open orders | `409` | `Supplier has open orders.` |
| Items assigned | `409` | `Supplier has items assigned.` |

---

## Screen Requirements

UI images: [docs/ui/feature-3/](../docs/ui/feature-3/) (Supplier List, Supplier Dialog)

### [View: Suppliers] — route name `suppliers`, path `/suppliers`
*   Heading: **Suppliers**
*   Primary action: **+ New Supplier** (`oc-cta`)
*   `<v-data-table>` columns: **Name**, **Phone**, **Fax**, **Address**, actions
*   Row actions: **Edit supplier**, **Delete supplier** (icon buttons with `aria-label`)
*   Empty state: **"No suppliers yet. Add your first supplier."**
*   Loading and error states as in Feature 2

### [Dialog: Supplier] — add and edit
*   Title: **New Supplier** or **Edit Supplier**
*   Fields: **Name**, **Address**, **Phone**, **Fax**
*   Actions: **Save** (`oc-cta`), **Cancel**

### [Dialog: Delete Supplier]
*   Text: **"Delete {name}?"**
*   Actions: **Delete**, **Cancel**. A `409` message is shown in the dialog.

---

## Key Entities

- **Supplier**: a manufacturer the warehouse buys from.
- **Item**: products a supplier provides (Feature 2).
- **Supplier Order**: orders placed with this supplier (Feature 5).

---

## Data Model Requirements

### `suppliers` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `name` | STRING(100) | Required, unique (case-insensitive) |
| `address` | STRING(200) | Required |
| `phone` | STRING(20) | Optional |
| `fax` | STRING(20) | Required |
| `createdAt` / `updatedAt` | DATE | Sequelize timestamps |

### Associations
*   `Supplier hasMany Item` (Feature 2)
*   `Supplier hasMany SupplierOrder` (Feature 5)

---

## Acceptance Criteria (Gherkin)

### US-3.1 — Add and edit suppliers

#### Scenario: Office worker adds a supplier
*   **Given** I am signed in as an office worker
*   **When** I add a supplier with a name, address, and fax number
*   **Then** the API returns `201`
*   **And** the supplier appears in the supplier list

#### Scenario: Duplicate supplier name is rejected
*   **Given** a supplier named `Big Fizz Bottling` exists
*   **When** I add a supplier named `big fizz bottling`
*   **Then** the API returns `400` with `{ "message": "Supplier name already in use." }`

#### Scenario: Office worker edits a supplier
*   **Given** a supplier exists with fax `918-555-0101`
*   **When** I change the fax to `918-555-0199` and save
*   **Then** the API returns `200`
*   **And** the supplier list shows the new fax number

### US-3.2 — Protect suppliers with open orders

#### Scenario: Supplier with open orders cannot be deleted
*   **Given** a supplier has an open supplier order
*   **When** I try to delete the supplier
*   **Then** the API returns `409` with `{ "message": "Supplier has open orders." }`

#### Scenario: Supplier with no orders or items can be deleted
*   **Given** a supplier has no items and no open orders
*   **When** I delete the supplier
*   **Then** the API returns `200`
*   **And** the supplier is no longer in the list

---

## Definition of Done

*   [ ] Backend and frontend implemented per this spec (**FR-001 – FR-006** satisfied)
*   [ ] **Success Criteria (SC-001 – SC-003)** met
*   [ ] All Gherkin scenarios have tests, and `npm test` passes
*   [ ] `features/reference/api.md` and `data-model.md` updated
*   [ ] Merged to `dev` through a pull request

---

## Out of Scope

*   Sending orders to suppliers by fax, email, or EDI from the system
*   Supplier price lists and contracts
*   Multiple contacts per supplier
*   Supplier performance reports (late or short deliveries)
