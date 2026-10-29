# Feature: Truck Routes and Shipping

**Feature ID:** 7
**Branch pattern:** `feature/7-shipping-routes`
**Status:** Draft
**Created:** 2026-09-29
**Updated:** 2026-10-28 (design draft)
**Input:** Office workers keep truck routes, managers assign trucks and drivers, and drivers ship the picked orders on their route
**Depends on:** [Feature 4 — Customer Management](feature-4-customer-management.md), [Feature 6 — Customer Ordering and Picking](feature-6-customer-ordering-picking.md)

---

## User Stories

### US-7.1: Manage truck routes
**As a** office worker  
**I want to** add, edit, and delete truck routes  
**So that** deliveries are organized

**Priority:** P1  
**Independent test:** Create a route and see it in the list  
**Acceptance scenarios:** see ### US-7.1 under Acceptance Criteria

### US-7.2: Assign a truck and driver
**As a** manager  
**I want to** assign a truck and a driver to a route for a day  
**So that** every route has a vehicle and a driver

**Priority:** P1  
**Independent test:** Assignment shows on the route for that date  
**Acceptance scenarios:** see ### US-7.2 under Acceptance Criteria

### US-7.3: See my route's orders
**As a** delivery driver  
**I want to** see the picked orders on my route  
**So that** I know what to deliver

**Priority:** P1  
**Independent test:** Driver sees only orders on their own route  
**Acceptance scenarios:** see ### US-7.3 under Acceptance Criteria

### US-7.4: Mark an order shipped
**As a** delivery driver  
**I want to** mark an order as shipped  
**So that** the order record is accurate

**Priority:** P2  
**Independent test:** A Picked order becomes Shipped  
**Acceptance scenarios:** see ### US-7.4 under Acceptance Criteria

---

## Requirements

### Functional Requirements

- **FR-001**: Only Office Workers and Managers MUST be able to add, edit, or delete routes.
- **FR-002**: A route with customers assigned MUST NOT be deleted.
- **FR-003**: Only a Manager MUST be able to assign a truck and driver to a route.
- **FR-004**: A driver MUST only see orders on the route assigned to them for the day.
- **FR-005**: Only orders with status Picked MUST be marked Shipped.
- **FR-006**: A route MUST have at most one assignment per date; a truck and a driver MUST each be assigned to at most one route per date.
- **FR-007**: The driver on an assignment MUST be a user with role `driver`.
- **FR-008**: Marking an order shipped MUST record the driver and the date and time.

---

## Assumptions

- Trucks are seeded. Adding and editing trucks is out of scope; `GET /api/trucks` only lists them.
- "Today" is the warehouse's local date (America/Chicago), computed on the server.
- A driver's route orders are the `Picked` orders for customers whose `routeId` matches the driver's assignment for today. Orders shipped today stay in the list, marked **Shipped**, for reference.
- The `truck_routes` table and `GET /api/truck-routes` already exist from [Feature 4](feature-4-customer-management.md); this feature adds create, edit, and delete.

## Edge Cases

- Route name empty → `400` with `{ "message": "Route name is required." }`; duplicate → `400` with `{ "message": "Route name already in use." }`.
- Delete a route with customers → `409` with `{ "message": "Route has customers assigned." }`.
- Delete a route with future assignments → `409` with `{ "message": "Route has upcoming assignments." }`.
- Second assignment for the same route and date → `400` with `{ "message": "Route already assigned for that date." }`.
- Same truck or driver on two routes for the same date → `400` with `{ "message": "Truck already assigned on that date." }` or `{ "message": "Driver already assigned on that date." }`.
- `driverId` that is not a driver → `400` with `{ "message": "Selected user is not a driver." }`.
- Assignment date in the past → `400` with `{ "message": "Assignment date cannot be in the past." }`.
- Driver with no assignment today → `200` with `{ "assignment": null, "orders": [] }`; the screen shows **"You have no route assigned today."**
- Driver ships an order that is not on their route today → `404` (scope rule, the same as cross-store access).
- Driver ships an order on their route that is not `Picked` → `400` with `{ "message": "Only Picked orders can be shipped." }`.

## Success Criteria

- **SC-001**: Every Gherkin scenario has at least one automated test before merge.
- **SC-002**: **0** orders from other routes are visible to a driver in automated tests.
- **SC-003**: A manager can assign trucks and drivers for 5 routes for a day in **≤ 3 minutes**.
- **SC-004**: **100%** of shipped orders record who shipped them and when.

---

## Data Ownership & Isolation

| Rule | Requirement |
|------|-------------|
| **Routes** | Shared data; Office and Manager write; `403` for other roles. |
| **Assignments** | Manager only (read and write). |
| **Driver read scope** | `GET /api/my-route` uses `req.user.id` to find today's assignment; a driver can never pass another driver's id. |
| **Driver write scope** | `POST /api/customer-orders/:id/ship` applies only when the order's customer is on the driver's route for today; otherwise `404`. |
| **Audit** | `shippedBy` comes from `req.user.id`; `shippedAt` comes from the server clock. |
| **Implementation** | `getDriverRouteOrderOrNull(req, orderId)` in `app/authorization/` returns the order only when it is on the caller's route today. |

---

## API Requirements

| Method | Endpoint | Auth | Roles | Purpose |
|--------|----------|------|-------|---------|
| `GET` | `/api/truck-routes` | Yes | office, manager | List routes with customer count (from Feature 4) |
| `POST` | `/api/truck-routes` | Yes | office, manager | Add a route |
| `PUT` | `/api/truck-routes/:id` | Yes | office, manager | Rename a route |
| `DELETE` | `/api/truck-routes/:id` | Yes | office, manager | Delete a route with no customers and no upcoming assignments |
| `GET` | `/api/trucks` | Yes | manager | List trucks (seeded) |
| `GET` | `/api/drivers` | Yes | manager | List active users with role `driver` |
| `GET` | `/api/route-assignments?date=YYYY-MM-DD` | Yes | manager | Assignments for a date |
| `POST` | `/api/route-assignments` | Yes | manager | Assign a truck and driver to a route for a date |
| `PUT` | `/api/route-assignments/:id` | Yes | manager | Change the truck or driver |
| `DELETE` | `/api/route-assignments/:id` | Yes | manager | Remove an assignment |
| `GET` | `/api/my-route` | Yes | driver | Today's assignment and the orders on that route |
| `POST` | `/api/customer-orders/:id/ship` | Yes | driver | Mark a `Picked` order `Shipped` |

**Route request body:**
```json
{ "name": "Route 1" }
```

**Route response** (`200` / `201`):
```json
{ "id": 1, "name": "Route 1", "customerCount": 8 }
```

**Assignment request body:**
```json
{ "routeId": 1, "truckId": 3, "driverId": 9, "routeDate": "2026-10-29" }
```

**Assignment response** (`201`):
```json
{
  "id": 55,
  "routeId": 1, "routeName": "Route 1",
  "truckId": 3, "truckLabel": "Truck 3",
  "driverId": 9, "driverName": "Sam Lee",
  "routeDate": "2026-10-29"
}
```

**My route response** (`200`):
```json
{
  "assignment": { "id": 55, "routeName": "Route 1", "truckLabel": "Truck 3", "routeDate": "2026-10-29" },
  "orders": [
    { "id": 42, "storeName": "Quick Stop #12", "address": "455 N Main St, Edmond, OK 73034", "binNumber": "B-07", "lineCount": 14, "status": "Picked" }
  ]
}
```

**Ship response** (`200`):
```json
{ "id": 42, "status": "Shipped", "shippedBy": 9, "shippedAt": "2026-10-29T14:05:00.000Z" }
```

---

## Screen Requirements

UI images: [docs/ui/feature-7/](../docs/ui/feature-7/) (Truck Routes, Route Assignments, My Route)

### [View: Truck Routes] — route name `truck-routes`, path `/truck-routes` (office, manager)
*   Heading: **Truck Routes**
*   Primary action: **+ New Route** (`oc-cta`)
*   Table: **Route**, **Customers** (count), actions **Edit route**, **Delete route**
*   Manager only: an **Assignments** button in the header → `route-assignments`
*   Empty state: **"No routes yet. Add your first route."**
*   Dialogs: **New Route** / **Edit Route** (field **Route Name**; **Save**, **Cancel**); **Delete Route** confirm with the `409` message shown in the dialog

### [View: Route Assignments] — route name `route-assignments`, path `/truck-routes/assignments` (manager)
*   Heading: **Route Assignments**
*   **Date** picker (defaults to tomorrow)
*   One row per route: **Route**, **Truck** select, **Driver** select, row action **Save** (or **Clear** when already assigned)
*   Trucks and drivers already used on the selected date are disabled in the other rows' selects.
*   Success snackbar: **"Assignment saved."** API errors show in a `<v-alert type="error">`.

### [View: My Route] — route name `my-route`, path `/my-route` (driver)
*   Header: **{routeName} — {truckLabel} — {date}**
*   One card per order: **Store**, **Address**, **Order #**, **Bin**, **Lines**, status chip
*   `Picked` orders have **Mark Shipped** (`oc-cta`), which opens a confirm dialog **"Mark order #{id} shipped?"**
*   `Shipped` orders show a green **Shipped** chip and no button.
*   No assignment: **"You have no route assigned today."**
*   Layout is designed for a phone or tablet (single column, large buttons).

---

## Key Entities

- **Truck Route**: a delivery route serving a set of customers.
- **Truck**: a delivery vehicle.
- **Route Assignment**: the truck and driver for a route on a date.
- **Customer Order**: shipped by the driver (Feature 6).

---

## Data Model Requirements

### `truck_routes` table (from Feature 4)
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `name` | STRING(50) | Required, unique |

### `trucks` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `label` | STRING(30) | Required, unique |

### `route_assignments` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `routeId` | INTEGER FK | Required |
| `truckId` | INTEGER FK | Required |
| `driverId` | INTEGER FK | Required; references `users.id`; user role must be `driver` |
| `routeDate` | DATEONLY | Required |
| Unique indexes | | (`routeId`, `routeDate`), (`truckId`, `routeDate`), (`driverId`, `routeDate`) |

### Changes to `customer_orders` (Feature 6)
| Field | Type | Rules |
|---|---|---|
| `shippedBy` | INTEGER FK | Nullable; references `users.id` |
| `shippedAt` | DATE | Nullable; set when status becomes `Shipped` |

### Associations
*   `TruckRoute hasMany RouteAssignment`; `RouteAssignment belongsTo TruckRoute`, `Truck`, and `User` (as `driver`)

---

## Acceptance Criteria (Gherkin)

### US-7.1 — Manage truck routes

#### Scenario: Office worker adds a route
*   **Given** I am signed in as an office worker
*   **When** I add a route named `Route 1`
*   **Then** the API returns `201`
*   **And** the route appears in the route list

#### Scenario: Route with customers cannot be deleted
*   **Given** `Route 1` has customers assigned
*   **When** I delete `Route 1`
*   **Then** the API returns `409` with `{ "message": "Route has customers assigned." }`

### US-7.2 — Assign a truck and driver

#### Scenario: Manager assigns a truck and driver
*   **Given** I am signed in as a manager
*   **When** I assign a truck and a driver to `Route 1` for a date
*   **Then** the assignment is saved

#### Scenario: Driver cannot be on two routes the same day
*   **Given** driver `Sam Lee` is assigned to `Route 1` on `2026-10-29`
*   **When** I assign `Sam Lee` to `Route 2` on `2026-10-29`
*   **Then** the API returns `400` with `{ "message": "Driver already assigned on that date." }`

#### Scenario: Office worker cannot assign drivers
*   **Given** I am signed in as an office worker
*   **When** I send `POST /api/route-assignments`
*   **Then** the API returns `403`

### US-7.3 — See my route's orders

#### Scenario: Driver sees only their route
*   **Given** I am a driver assigned to `Route 1` today
*   **When** I open my route
*   **Then** I see only picked orders for stores on `Route 1`

#### Scenario: Driver with no route today
*   **Given** I am a driver with no assignment today
*   **When** I open my route
*   **Then** I see **"You have no route assigned today."**

### US-7.4 — Mark an order shipped

#### Scenario: Driver ships a picked order
*   **Given** an order on my route has status `Picked`
*   **When** I mark it shipped
*   **Then** its status is `Shipped`
*   **And** the order records me as the driver and the time it shipped

#### Scenario: Unpicked order cannot be shipped
*   **Given** an order on my route has status `New`
*   **When** I try to mark it shipped
*   **Then** the API returns `400` with `{ "message": "Only Picked orders can be shipped." }`

#### Scenario: Driver cannot ship another route's order
*   **Given** a `Picked` order belongs to a store on `Route 2`
*   **And** I am the driver for `Route 1` today
*   **When** I send `POST /api/customer-orders/:id/ship` for that order
*   **Then** the API returns `404`

---

## Definition of Done

*   [ ] Backend and frontend implemented per this spec (**FR-001 – FR-008** satisfied)
*   [ ] **Success Criteria (SC-001 – SC-004)** met
*   [ ] Unique indexes on `route_assignments` exist, and the API returns the friendly `400` messages instead of raw database errors
*   [ ] Seed data includes at least 2 trucks and 2 drivers
*   [ ] All Gherkin scenarios have tests, and `npm test` passes
*   [ ] `features/reference/api.md`, `data-model.md`, and `behavior.md` updated
*   [ ] Merged to `dev` through a pull request

---

## Out of Scope

*   Adding and editing trucks (seeded only)
*   Stop order, GPS tracking, and turn-by-turn directions
*   Proof of delivery (signature or photo) and marking an order delivered
*   Returns and refused deliveries on the route (a candidate operational feature for the final requirements draft)
*   Driver hours and truck maintenance records
