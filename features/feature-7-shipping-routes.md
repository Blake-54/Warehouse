# Feature: Truck Routes and Shipping

**Feature ID:** 7
**Branch pattern:** `feature/7-shipping-routes`
**Status:** Draft
**Created:** 2026-09-29
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

---

## Key Entities

- **Truck Route**: a delivery route serving a set of customers.
- **Truck**: a delivery vehicle.
- **Route Assignment**: the truck and driver for a route on a date.

---

## Data Model Requirements

### `truck_routes` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `name` | STRING | Required, unique |


### `trucks` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `label` | STRING | Required, unique |


### `route_assignments` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `routeId` | INTEGER FK | Required |
| `truckId` | INTEGER FK | Required |
| `driverId` | INTEGER FK | Required; references `users.id` |
| `routeDate` | DATE | One assignment per route per date |


---

## Acceptance Criteria (Gherkin)

### US-7.1 — Manage truck routes

#### Scenario: Office worker adds a route
*   **Given** I am signed in as an office worker
*   **When** I add a route named `Route 1`
*   **Then** the API returns `201`
*   **And** the route appears in the route list

### US-7.2 — Assign a truck and driver

#### Scenario: Manager assigns a truck and driver
*   **Given** I am signed in as a manager
*   **When** I assign a truck and a driver to `Route 1` for a date
*   **Then** the assignment is saved

### US-7.3 — See my route's orders

#### Scenario: Driver sees only their route
*   **Given** I am a driver assigned to `Route 1` today
*   **When** I open my route
*   **Then** I see only picked orders for stores on `Route 1`

### US-7.4 — Mark an order shipped

#### Scenario: Driver ships a picked order
*   **Given** an order on my route has status `Picked`
*   **When** I mark it shipped
*   **Then** its status is `Shipped`

#### Scenario: Unpicked order cannot be shipped
*   **Given** an order has status `New`
*   **When** I try to mark it shipped
*   **Then** the API returns `400`
