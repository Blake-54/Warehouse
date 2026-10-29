# Feature: User Login and Roles

**Feature ID:** 1
**Branch pattern:** `feature/1-user-login`
**Status:** Draft
**Created:** 2026-09-29
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

---

## Key Entities

- **User**: a person who signs in; has one role (employees and customer logins).
- **Session**: record tying a sign-in token to a user.
- **Customer**: the store a Customer user belongs to.

---

## Data Model Requirements

### `users` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `name` | STRING | Required |
| `username` | STRING | Required, unique |
| `passwordHash` | STRING | Required |
| `role` | ENUM | office, receiver, picker, driver, manager, customer |
| `customerId` | INTEGER FK | Only for role `customer`; references `customers.id` |
| `active` | BOOLEAN | Default true |


### `sessions` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `token` | STRING | Required, unique |
| `userId` | INTEGER FK | Required, references `users.id` |


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

### US-1.2 — Sign out

#### Scenario: User signs out
*   **Given** I am signed in
*   **When** I click **Sign out**
*   **Then** my session ends
*   **And** I am redirected to the login screen

### US-1.3 — Limit inventory changes to managers

#### Scenario: Picker cannot change inventory
*   **Given** I am signed in as a picker
*   **When** I try to change an item's quantity on hand
*   **Then** the API returns `403`

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
