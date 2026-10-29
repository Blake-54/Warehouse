# Feature: User Login and Roles

**Feature ID:** 1
**Branch pattern:** `feature/1-user-login`
**Status:** Draft
**Created:** 2026-09-29
**Updated:** 2026-10-28 (design draft)
**Input:** Employees and customers sign in with a username and password, and the role controls what each person can do

---

## User Stories

### US-1.1: Sign in
**As a** registered user  
**I want to** sign in with my username and password  
**So that** only authorized people can use the warehouse system

**Priority:** P1  
**Independent test:** Sign in with known credentials and receive a token and role  
**Acceptance scenarios:** see ### US-1.1 under Acceptance Criteria

### US-1.2: Sign out
**As a** signed-in user  
**I want to** sign out  
**So that** my session ends on a shared device

**Priority:** P2  
**Independent test:** After sign out, the home screen redirects to login  
**Acceptance scenarios:** see ### US-1.2 under Acceptance Criteria

### US-1.3: Limit inventory changes to managers
**As a** manager  
**I want to** be the only role that can change inventory quantities  
**So that** stock counts stay accurate

**Priority:** P1  
**Independent test:** A picker is denied and a manager is allowed to change a quantity  
**Acceptance scenarios:** see ### US-1.3 under Acceptance Criteria

### US-1.4: See only my own orders
**As a** customer  
**I want to** see only my own store's orders  
**So that** other stores cannot see what I ordered

**Priority:** P1  
**Independent test:** Customer A cannot see customer B's orders  
**Acceptance scenarios:** see ### US-1.4 under Acceptance Criteria

---

## Requirements

### Functional Requirements

- **FR-001**: Users MUST sign in with a username and password.
- **FR-002**: Passwords MUST be stored as a hash, never as plain text.
- **FR-003**: Each user MUST have exactly one role: Office Worker, Receiver, Picker, Driver, Manager, or Customer.
- **FR-004**: A Customer user MUST be linked to exactly one customer (store).
- **FR-005**: Only a Manager MUST be able to change inventory quantities.
- **FR-006**: Customers MUST NOT see other customers' orders.
- **FR-007**: Requests without a valid session MUST be rejected with `401`.
- **FR-008**: A signed-in user whose role is not allowed for an action MUST be rejected with `403`.
- **FR-009**: An inactive user (`active = false`) MUST NOT be able to sign in.
- **FR-010**: After sign in, the home screen MUST show only the menu options allowed for the user's role.

---

## Assumptions

- User accounts are created by a seed script for this release; there is no user-administration screen (see Out of Scope).
- Usernames are compared case-insensitively (`trim().toLowerCase()`), per [security.mdc](../.cursor/rules/security.mdc).
- Sessions last 24 hours, per [auth-patterns.mdc](../.cursor/rules/auth-patterns.mdc).
- This feature delivers the shared guards (`authenticate`, `requireRole`, `customerScope`). The US-1.3 and US-1.4 scenarios are proven against the endpoints added in [Feature 2](feature-2-item-management.md) and [Feature 6](feature-6-customer-ordering-picking.md).
- "Stocker" in Feature 5 is the Receiver role; there is no separate Stocker role.

## Edge Cases

- Username or password missing → `400` with `{ "message": "Username and password are required." }`.
- Wrong password or unknown username → the same `401` message, so the response does not reveal which usernames exist.
- Inactive user signs in → `401` with the same message as a wrong password.
- Expired or deleted session token → `401`; the frontend clears `localStorage` and redirects to the login screen.
- A Customer user whose `customerId` is missing → treated as a data error; sign in returns `401` and the problem is logged at `error`.
- A user signs in on two devices → both sessions stay valid until each one signs out or expires.

## Success Criteria

- **SC-001**: Every Gherkin scenario has at least one automated test before merge.
- **SC-002**: **100%** of `/api` routes except `POST /api/login` and `GET /api/health` require a valid session.
- **SC-003**: Every role sees only its own menu options on the home screen (checked for all 6 roles).
- **SC-004**: A user can sign in and reach the home screen in **≤ 10 seconds**.

---

## Data Ownership & Isolation

This feature defines two kinds of access control that every later feature reuses:

1. **Role-based access:** what a role may do. A wrong role gets `403`.
2. **Ownership scope:** which rows a Customer or Driver may see. A row outside their scope gets `404`, never `403`, so the response does not confirm the row exists.

| Rule | Requirement |
|------|-------------|
| **Read scope** | Employees (office, receiver, picker, manager) read warehouse-wide data allowed for their role. Customers read only rows where `customerId = req.user.customerId`. Drivers read only orders on their assigned route for the day (Feature 7). |
| **Write scope** | Writes are allowed only when `requireRole(...)` passes **and** the row is in the caller's scope. |
| **Create scope** | Owner fields (`customerId`, `userId`, `driverId` on actions) come from `req.user`, never from the request body. |
| **Cross-user access** | Another customer's order, or another driver's route order → `404`. |
| **Wrong role** | Valid session but role not allowed → `403` with `{ "message": "You do not have permission to do that." }`. |
| **UI scope** | The SPA hides menu items and buttons the role cannot use. This is for UX only; the API is the real enforcement. |
| **Implementation** | `app/authorization/authorization.js` exports `authenticate`, `requireRole(...roles)`, and `customerScope(req)`. Controllers call these helpers; they do not repeat role or scope checks inline. |

### Role permission summary (used by Features 2–7)

| Area | Office | Receiver | Picker | Driver | Manager | Customer |
|------|:-:|:-:|:-:|:-:|:-:|:-:|
| Items (view) | ✓ | ✓ | ✓ | | ✓ | in-stock flag only |
| Items (add/edit/delete) | ✓ | | | | ✓ | |
| Reorder levels, quantity adjustments | | | | | ✓ | |
| Suppliers, customers, routes | ✓ | | | | ✓ | |
| Supplier orders (view) | ✓ | ✓ | | | ✓ | |
| Receiving and put-away | | ✓ | | | ✓ | |
| Customer orders (place/cancel own) | | | | | | ✓ |
| Customer orders (view all) | ✓ | | ✓ | | ✓ | |
| Picking | | | ✓ | | ✓ | |
| Route assignments | | | | | ✓ | |
| Ship orders on own route | | | | ✓ | | |

---

## API Requirements

All routes are mounted under `/api`.

| Method | Endpoint | Auth | Roles | Purpose |
|--------|----------|------|-------|---------|
| `POST` | `/api/login` | No | — | Sign in; returns session token and role |
| `POST` | `/api/logout` | Yes | Any | End the current session |
| `GET` | `/api/health` | No | — | Health check (already in the starter kit) |

**Login request body:**
```json
{ "username": "jsmith", "password": "correct-horse" }
```

**Login success response** (`200`):
```json
{
  "userId": 7,
  "name": "Jane Smith",
  "username": "jsmith",
  "role": "office",
  "customerId": null,
  "token": "<jwt>"
}
```
`customerId` is set only for role `customer`. The password hash is never returned.

**Logout success response** (`200`):
```json
{ "message": "Signed out." }
```

**Standard error contract (all features):**

| Status | When | Body |
|--------|------|------|
| `400` | Validation failed, or an invalid status change | `{ "message": "<what is wrong>" }` |
| `401` | No session, or the token is invalid or expired | `{ "message": "Unauthorized! Please sign in." }` |
| `403` | Signed in, but the role is not allowed | `{ "message": "You do not have permission to do that." }` |
| `404` | Row does not exist or is outside the caller's scope | `{ "message": "<Thing> with id=<id> not found." }` |
| `409` | Delete blocked by related data (open orders, assigned customers) | `{ "message": "<reason>" }` |

Responses are flat JSON, with no `{ success, data }` envelope ([api-conventions.mdc](../.cursor/rules/api-conventions.mdc)).

---

## Screen Requirements

UI image: [docs/ui/feature-1/](../docs/ui/feature-1/) (Login, Home)

### [View: Login] — route name `login`, path `/login`
*   Title: **Warehouse Sign In**
*   Fields: **Username**, **Password** (masked, with a show/hide icon)
*   Primary action: **Sign in** (`oc-cta`)
*   Client validation: both fields required. Messages: **"Username is required."** and **"Password is required."**
*   Error state: `<v-alert type="error">` showing the API `message` (for example **"Invalid username or password."**)
*   No `MenuBar` on this screen.

### [View: Home] — route name `home`, path `/`
*   Heading: **Welcome, {name}**, with the role shown as a chip (for example **Office Worker**)
*   A grid of menu cards, showing only the cards allowed for the role:

| Card | Route | Roles |
|------|-------|-------|
| **Items** | `items` | office, receiver, picker, manager |
| **Suppliers** | `suppliers` | office, manager |
| **Customers** | `customers` | office, manager |
| **Supplier Orders** | `supplier-orders` | office, receiver, manager |
| **Customer Orders** | `customer-orders` | office, manager |
| **My Orders** | `my-orders` | customer |
| **Pick Queue** | `pick-queue` | picker, manager |
| **Truck Routes** | `truck-routes` | office, manager |
| **My Route** | `my-route` | driver |

### [Component: MenuBar]
*   Shown on every route except `login`
*   Left: app name **Warehouse**, which links to `home`
*   Right: signed-in user's name and role, plus a **Sign out** button

### Router guard
*   No `user` in `localStorage` → redirect to `login`.
*   Signed in, but the route's `meta.roles` does not include the user's role → redirect to `home`. This is UX only; the API still returns `403`.

---

## Key Entities

- **User**: a person who signs in; has one role (employees and customer logins).
- **Session**: record tying a sign-in token to a user.
- **Customer**: the store a Customer user belongs to (full maintenance in Feature 4).

---

## Data Model Requirements

### `users` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `name` | STRING | Required |
| `username` | STRING | Required, unique, stored lowercase |
| `passwordHash` | STRING | Required; excluded by `defaultScope` |
| `role` | ENUM | office, receiver, picker, driver, manager, customer |
| `customerId` | INTEGER FK | Required only for role `customer`; null otherwise; references `customers.id` |
| `active` | BOOLEAN | Default true |
| `createdAt` / `updatedAt` | DATE | Sequelize timestamps |

### `sessions` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `token` | STRING | Required, unique; emptied on sign out |
| `userId` | INTEGER FK | Required, references `users.id` |
| `expirationDate` | DATE | Required; sign in + 24 hours |

### Associations (in `models/index.js`)
*   `User hasMany Session`, `Session belongsTo User`
*   `Customer hasMany User`, `User belongsTo Customer` (customer logins)

---

## Acceptance Criteria (Gherkin)

### US-1.1 — Sign in

#### Scenario: User signs in with valid credentials
*   **Given** a user exists with username `jsmith` and a known password
*   **When** I enter `jsmith` and the correct password
*   **Then** the API returns `200` with a token and my role
*   **And** I see the home screen

#### Scenario: User signs in with an invalid password
*   **Given** a user exists with username `jsmith`
*   **When** I enter `jsmith` and a wrong password
*   **Then** the API returns `401` with `{ "message": "Invalid username or password." }`

#### Scenario: User signs in with a missing field
*   **Given** I am on the login screen
*   **When** I click **Sign in** with the password empty
*   **Then** I see **"Password is required."**
*   **And** no API request is sent

#### Scenario: Inactive user cannot sign in
*   **Given** user `oldclerk` has `active = false`
*   **When** I sign in as `oldclerk` with the correct password
*   **Then** the API returns `401` with `{ "message": "Invalid username or password." }`

#### Scenario: Home screen shows only my role's options
*   **Given** I am signed in as a picker
*   **When** I view the home screen
*   **Then** I see the **Items** and **Pick Queue** cards
*   **And** I do not see the **Suppliers** card

### US-1.2 — Sign out

#### Scenario: User signs out
*   **Given** I am signed in
*   **When** I click **Sign out**
*   **Then** my session ends
*   **And** I am redirected to the login screen

#### Scenario: Request without a session is rejected
*   **Given** I have no valid session token
*   **When** I request `GET /api/items`
*   **Then** the API returns `401`

### US-1.3 — Limit inventory changes to managers

#### Scenario: Picker cannot change inventory
*   **Given** I am signed in as a picker
*   **When** I try to change an item's quantity on hand
*   **Then** the API returns `403` with `{ "message": "You do not have permission to do that." }`

#### Scenario: Manager changes inventory
*   **Given** I am signed in as a manager
*   **When** I change an item's quantity on hand and give a reason
*   **Then** the new quantity is saved

### US-1.4 — See only my own orders

#### Scenario: Customer cannot see another customer's orders
*   **Given** customer B has an order
*   **And** I am signed in as customer A
*   **When** I request the order list
*   **Then** the list contains only customer A's orders

#### Scenario: Customer cannot open another customer's order by id
*   **Given** customer B has order `42`
*   **And** I am signed in as customer A
*   **When** I request `GET /api/customer-orders/42`
*   **Then** the API returns `404`

---

## Definition of Done

*   [ ] Backend and frontend implemented per this spec (**FR-001 – FR-010** satisfied)
*   [ ] **Success Criteria (SC-001 – SC-004)** met
*   [ ] `authenticate`, `requireRole`, and `customerScope` exported from `app/authorization/` and covered by tests
*   [ ] All Gherkin scenarios have tests, and `npm test` passes
*   [ ] Seed script creates one user per role for demos and tests
*   [ ] `features/reference/api.md`, `data-model.md`, and `behavior.md` updated
*   [ ] Merged to `dev` through a pull request

---

## Out of Scope

*   User administration screens (add, edit, or deactivate users). Users are seeded for now; candidate for the final requirements draft.
*   Self-registration and password reset
*   Multi-factor sign-in
*   Remember-me and session refresh beyond 24 hours
*   Per-user permission overrides (permissions are by role only)
