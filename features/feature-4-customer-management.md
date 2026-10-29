# Feature: Customer Management

**Feature ID:** 4
**Branch pattern:** `feature/4-customer-management`
**Status:** Draft
**Created:** 2026-09-29
**Updated:** 2026-10-28 (design draft)
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
- **FR-005**: Store names MUST be unique (case-insensitive).
- **FR-006**: The customer list MUST show each customer's route, or **Unassigned** when there is none.

---

## Assumptions

- An **open** customer order is one with status `New`, `Picking`, or `Picked` ([ADR-0007](../docs/adr/0007-order-status-state-machines.md)).
- This feature creates the `truck_routes` table and a read-only route list so customers can be assigned. Adding, editing, and deleting routes comes in [Feature 7](feature-7-shipping-routes.md). Until then, routes come from the seed script.
- Customer sign-in accounts (role `customer`) are linked to a customer by `users.customerId` and are seeded (see Feature 1, Out of Scope).
- A customer without a route can still place orders, but the orders will not appear on any driver's route until a route is assigned.

## Edge Cases

- Store name empty → `400` with `{ "message": "Store name is required." }`.
- Address empty → `400` with `{ "message": "Address is required." }`.
- Duplicate store name → `400` with `{ "message": "Store name already in use." }`.
- `routeId` that does not exist → `400` with `{ "message": "Route not found." }`.
- Clearing a route (`routeId: null`) → allowed; the customer shows **Unassigned**.
- Delete with an open order → `409` with `{ "message": "Customer has open orders." }`.
- Delete when customer login accounts are still linked → `409` with `{ "message": "Customer has user accounts." }`.
- Changing a customer's route while one of its orders is `Picked` → allowed; the order moves to the new route's driver list on the next refresh.

## Success Criteria

- **SC-001**: Every Gherkin scenario has at least one automated test before merge.
- **SC-002**: **100%** of customers in the list show a route or **Unassigned**.
- **SC-003**: **0** customers with open orders can be deleted (checked by API tests).

---

## Data Ownership & Isolation

Customer records are shared warehouse data managed by staff. Customer **users** never use these endpoints.

| Rule | Requirement |
|------|-------------|
| **Read scope** | Office and Manager can list all customers. |
| **Write scope** | Office and Manager only. |
| **Customer role** | A signed-in customer calling `/api/customers` gets `403`. (Their own store name comes back in the login payload and on their orders.) |
| **Not found** | `404` with `{ "message": "Customer with id=<id> not found." }`. |
| **Implementation** | `requireRole("office", "manager")` on all `/api/customers` routes. |

---

## API Requirements

| Method | Endpoint | Auth | Roles | Purpose |
|--------|----------|------|-------|---------|
| `GET` | `/api/customers?routeId=` | Yes | office, manager | List customers sorted by store name; optional route filter |
| `GET` | `/api/customers/:id` | Yes | office, manager | One customer with route |
| `POST` | `/api/customers` | Yes | office, manager | Add a customer |
| `PUT` | `/api/customers/:id` | Yes | office, manager | Edit a customer, including `routeId` |
| `DELETE` | `/api/customers/:id` | Yes | office, manager | Delete a customer with no open orders and no linked users |
| `GET` | `/api/truck-routes` | Yes | office, manager | Route list for the route dropdown (full maintenance in Feature 7) |

**Create / edit request body:**
```json
{
  "storeName": "Quick Stop #12",
  "address": "455 N Main St, Edmond, OK 73034",
  "phone": "405-555-0112",
  "contactName": "Maria Lopez",
  "routeId": 1
}
```

**Customer response** (`200` / `201`):
```json
{
  "id": 12,
  "storeName": "Quick Stop #12",
  "address": "455 N Main St, Edmond, OK 73034",
  "phone": "405-555-0112",
  "contactName": "Maria Lopez",
  "routeId": 1,
  "route": { "id": 1, "name": "Route 1" }
}
```

**Errors:**

| Case | Status | Message |
|------|--------|---------|
| Missing store name | `400` | `Store name is required.` |
| Missing address | `400` | `Address is required.` |
| Duplicate store name | `400` | `Store name already in use.` |
| Unknown route | `400` | `Route not found.` |
| Open orders | `409` | `Customer has open orders.` |
| Linked users | `409` | `Customer has user accounts.` |

---

## Screen Requirements

UI images: [docs/ui/feature-4/](../docs/ui/feature-4/) (Customer List, Customer Dialog)

### [View: Customers] — route name `customers`, path `/customers`
*   Heading: **Customers**
*   Filter: **Route** select (All routes, each route, Unassigned)
*   Primary action: **+ New Customer** (`oc-cta`)
*   `<v-data-table>` columns: **Store**, **Contact**, **Phone**, **Address**, **Route**, actions
*   The **Route** column shows the route name, or an **Unassigned** chip.
*   Row actions: **Edit customer**, **Delete customer**
*   Empty state: **"No customers yet. Add your first store."**

### [Dialog: Customer] — add and edit
*   Title: **New Customer** or **Edit Customer**
*   Fields: **Store Name**, **Address**, **Contact Name**, **Phone**, **Truck Route** (select with a **None** option)
*   Actions: **Save** (`oc-cta`), **Cancel**

### [Dialog: Delete Customer]
*   Text: **"Delete {storeName}?"**. A `409` message is shown in the dialog.

---

## Key Entities

- **Customer**: a convenience store that orders from the warehouse.
- **Truck Route**: the delivery route a customer belongs to (maintained in Feature 7).
- **User**: customer login accounts linked to a store (Feature 1).

---

## Data Model Requirements

### `customers` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `storeName` | STRING(100) | Required, unique (case-insensitive) |
| `address` | STRING(200) | Required |
| `phone` | STRING(20) | Optional |
| `contactName` | STRING(100) | Optional |
| `routeId` | INTEGER FK | Optional; references `truck_routes.id`; `ON DELETE RESTRICT` |
| `createdAt` / `updatedAt` | DATE | Sequelize timestamps |

### `truck_routes` table (created here; maintained in Feature 7)
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `name` | STRING(50) | Required, unique |

### Associations
*   `TruckRoute hasMany Customer`, `Customer belongsTo TruckRoute` (as `route`)
*   `Customer hasMany User` (Feature 1), `Customer hasMany CustomerOrder` (Feature 6)

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

#### Scenario: Customer user cannot manage customers
*   **Given** I am signed in as a customer
*   **When** I request `GET /api/customers`
*   **Then** the API returns `403`

### US-4.2 — Assign a truck route

#### Scenario: Office worker assigns a route
*   **Given** a customer and a route `Route 1` exist
*   **When** I assign the customer to `Route 1`
*   **Then** the customer shows `Route 1`

#### Scenario: Customer without a route shows Unassigned
*   **Given** a customer has no route
*   **When** I open the customer list
*   **Then** the customer's route shows **Unassigned**

### US-4.3 — Protect customers with open orders

#### Scenario: Customer with open orders cannot be deleted
*   **Given** a customer has an open order
*   **When** I try to delete the customer
*   **Then** the API returns `409` with `{ "message": "Customer has open orders." }`

---

## Definition of Done

*   [ ] Backend and frontend implemented per this spec (**FR-001 – FR-006** satisfied)
*   [ ] **Success Criteria (SC-001 – SC-003)** met
*   [ ] `truck_routes` table and `GET /api/truck-routes` added and seeded with at least 2 routes
*   [ ] All Gherkin scenarios have tests, and `npm test` passes
*   [ ] `features/reference/api.md` and `data-model.md` updated
*   [ ] Merged to `dev` through a pull request

---

## Out of Scope

*   Adding, editing, and deleting truck routes (Feature 7)
*   Creating customer login accounts from this screen (seeded for now)
*   Credit limits, billing, and invoicing
*   Delivery time windows and stop order within a route
