# Feature: Item Management

**Feature ID:** 2
**Branch pattern:** `feature/2-item-management`
**Status:** Draft
**Created:** 2026-09-29
**Updated:** 2026-10-28 (design draft)
**Input:** Office workers keep the list of items the warehouse sells, and managers set reorder levels
**Depends on:** [Feature 1 — User Login and Roles](feature-1-user-login.md), [Feature 3 — Supplier Management](feature-3-supplier-management.md) (supplier dropdown)

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

### US-2.5: Adjust quantity on hand
**As a** manager  
**I want to** correct an item's quantity on hand and record why  
**So that** counts match the shelf after damage or a recount

**Priority:** P2  
**Independent test:** An adjustment changes quantity on hand and is saved with a reason  
**Acceptance scenarios:** see ### US-2.5 under Acceptance Criteria

---

## Requirements

### Functional Requirements

- **FR-001**: Each item MUST have a description, case UPC, unit UPC, units per case, price, supplier cost, and supplier.
- **FR-002**: Case UPC and unit UPC MUST each be unique across all items. A case UPC also MUST NOT match any unit UPC.
- **FR-003**: An item's minimum quantity MUST be less than its maximum quantity.
- **FR-004**: Quantity on hand MUST NOT be edited on the normal item form; only a Manager MUST be able to change it, through an adjustment with a reason.
- **FR-005**: An item with stock on hand or open orders MUST NOT be deleted.
- **FR-006**: Only Office Workers and Managers MUST be able to add, edit, or delete items; only Managers MUST be able to set reorder levels.
- **FR-007**: Quantity on order MUST be maintained by the system (Feature 5) and MUST NOT be editable by any user.
- **FR-008**: Every quantity adjustment MUST record the old quantity, the new quantity, the reason, the manager, and the date.

---

## Assumptions

- All quantities (`qtyOnHand`, `qtyOnOrder`, `minQty`, `maxQty`) are in **units**. Supplier orders are placed in **cases** (Feature 5), converted using `unitsPerCase`.
- `qtyOnHand` always equals the sum of the item's `stock_lots.qty` ([ADR-0008](../docs/adr/0008-inventory-stock-lots-fifo.md)). An adjustment changes stock lots, not only the item's total.
- The `stock_lots` table is defined in [Feature 5](feature-5-supplier-ordering-receiving.md). US-2.1 – US-2.4 can ship first; **US-2.5 (adjustments) is built with or after Feature 5**.
- A new item starts with `qtyOnHand = 0` and `qtyOnOrder = 0`. Stock arrives only through receiving (Feature 5) or a manager adjustment.
- UPCs are stored as text so leading zeros are kept.
- Money is in US dollars with 2 decimal places.

## Edge Cases

- Description empty → `400` with `{ "message": "Description is required." }`.
- Units per case is 0 or negative → `400` with `{ "message": "Units per case must be greater than 0." }`.
- Price or supplier cost is negative → `400`.
- Editing an item and keeping its own UPC → allowed; the uniqueness check excludes the item being edited.
- Minimum equals maximum → `400`, because the minimum must be strictly less.
- Reorder levels left blank → the item is never auto-ordered (Feature 5 skips it).
- Deleting an item with `qtyOnHand > 0`, or one that is on an open supplier or customer order → `409`.
- An adjustment down to more than is on hand → not possible, because the new quantity is entered as an absolute value ≥ 0. A negative value → `400`.
- An adjustment that raises stock needs an aisle and side for the new stock lot → otherwise `400` with `{ "message": "Location is required when adding stock." }`.

## Success Criteria

- **SC-001**: Every Gherkin scenario has at least one automated test before merge.
- **SC-002**: **0** items can be saved with a duplicate UPC (checked by API tests).
- **SC-003**: **100%** of quantity changes made through this feature leave an adjustment record.
- **SC-004**: An office worker can find an item by description or UPC in **≤ 2 seconds** with 1,000 items loaded.

---

## Data Ownership & Isolation

Items are shared warehouse data. They are not owned by one user; access is controlled by **role** ([Feature 1](feature-1-user-login.md#role-permission-summary-used-by-features-27)).

| Rule | Requirement |
|------|-------------|
| **Read scope** | Office, Receiver, Picker, and Manager can read all items. Customers do **not** use `/api/items`; they see the catalog from Feature 6. |
| **Write scope** | Office and Manager can add, edit, and delete. Only Manager can set reorder levels or adjust quantity. |
| **Create scope** | `qtyOnHand` and `qtyOnOrder` are ignored if sent in the body; they always start at 0. |
| **Wrong role** | `403` (for example, a picker sending `POST /api/items`). |
| **Not found** | Unknown item id → `404` with `{ "message": "Item with id=<id> not found." }`. |
| **Audit** | `inventory_adjustments.userId` comes from `req.user.id`, never from the body. |
| **Implementation** | Routes use `requireRole("office", "manager")` or `requireRole("manager")` from `app/authorization/`. |

---

## API Requirements

| Method | Endpoint | Auth | Roles | Purpose |
|--------|----------|------|-------|---------|
| `GET` | `/api/items?search=` | Yes | office, receiver, picker, manager | List items, sorted by description; `search` matches description or either UPC |
| `GET` | `/api/items/:id` | Yes | office, receiver, picker, manager | One item with supplier name |
| `POST` | `/api/items` | Yes | office, manager | Add an item |
| `PUT` | `/api/items/:id` | Yes | office, manager | Edit item details (not quantities or reorder levels) |
| `DELETE` | `/api/items/:id` | Yes | office, manager | Delete an item with no stock and no open orders |
| `PUT` | `/api/items/:id/reorder-levels` | Yes | manager | Set `minQty` and `maxQty` |
| `POST` | `/api/items/:id/adjustments` | Yes | manager | Change quantity on hand, with a reason |
| `GET` | `/api/items/:id/adjustments` | Yes | manager | Adjustment history for an item |

**Create / edit item request body:**
```json
{
  "description": "Cola 12 oz can",
  "caseUpc": "10012345678902",
  "unitUpc": "012345678905",
  "unitsPerCase": 24,
  "price": 1.25,
  "supplierCost": 0.60,
  "supplierId": 3
}
```

**Item response** (`200` / `201`):
```json
{
  "id": 11,
  "description": "Cola 12 oz can",
  "caseUpc": "10012345678902",
  "unitUpc": "012345678905",
  "unitsPerCase": 24,
  "price": "1.25",
  "supplierCost": "0.60",
  "minQty": 50,
  "maxQty": 200,
  "qtyOnHand": 30,
  "qtyOnOrder": 100,
  "supplierId": 3,
  "supplier": { "id": 3, "name": "Big Fizz Bottling" }
}
```

**Reorder levels request body:**
```json
{ "minQty": 50, "maxQty": 200 }
```

**Adjustment request body:**
```json
{ "newQtyOnHand": 48, "reason": "Damaged cases removed", "aisle": "C", "side": "North" }
```
`aisle` and `side` are required only when `newQtyOnHand` is greater than the current quantity.

**Adjustment response** (`201`):
```json
{ "id": 5, "itemId": 11, "oldQty": 60, "newQty": 48, "reason": "Damaged cases removed", "userId": 2, "createdAt": "2026-10-28T15:00:00.000Z" }
```

**Errors** (contract from [Feature 1](feature-1-user-login.md#api-requirements)):

| Case | Status | Message |
|------|--------|---------|
| Duplicate case or unit UPC | `400` | `UPC already in use.` |
| Min not below max | `400` | `Minimum must be less than maximum.` |
| Missing reason on adjustment | `400` | `Reason is required.` |
| Delete with stock or open orders | `409` | `Item has stock or open orders.` |

**Side effect:** after an adjustment, the API runs the Feature 5 reorder check for that item.

---

## Screen Requirements

UI images: [docs/ui/feature-2/](../docs/ui/feature-2/) (Item List, Item Dialog, Reorder Levels Dialog, Adjust Quantity Dialog)

### [View: Items] — route name `items`, path `/items`
*   Heading: **Items**
*   Search box: **Search description or UPC**
*   Primary action: **+ New Item** (`oc-cta`; office and manager only)
*   `<v-data-table>` columns: **Description**, **Unit UPC**, **Units/Case**, **Price**, **On Hand**, **On Order**, **Min**, **Max**, **Supplier**, actions
*   Row actions (icon buttons with `aria-label`):
    *   **Edit item** (office, manager)
    *   **Reorder levels** (manager)
    *   **Adjust quantity** (manager)
    *   **Delete item** (office, manager)
*   Rows where **On Hand** is below **Min** show a **Low** chip in the On Hand column.
*   Empty state: **"No items yet. Add your first item."**
*   Loading: table loading bar. Error: `<v-alert type="error">` with the API message.

### [Dialog: Item] — add and edit
*   Title: **New Item** or **Edit Item**
*   Fields: **Description**, **Case UPC**, **Unit UPC**, **Units per Case**, **Price**, **Supplier Cost**, **Supplier** (select from `GET /api/suppliers`)
*   On Hand and On Order are **not** shown in this dialog.
*   Actions: **Save** (`oc-cta`), **Cancel**

### [Dialog: Reorder Levels] — manager only
*   Fields: **Minimum**, **Maximum**
*   Inline error: **"Minimum must be less than maximum."**
*   Actions: **Save**, **Cancel**

### [Dialog: Adjust Quantity] — manager only
*   Read-only: **Current on hand: {qty}**
*   Fields: **New quantity on hand**, **Reason** (required), and **Aisle** (A–T) and **Side** (North/South), which appear only when the new quantity is higher than the current one
*   Actions: **Save Adjustment**, **Cancel**

### [Dialog: Delete Item]
*   Text: **"Delete {description}? This cannot be undone."**
*   Actions: **Delete**, **Cancel**. A `409` response is shown in the dialog as an error alert.

---

## Key Entities

- **Item**: a product sold to convenience stores.
- **Supplier**: the manufacturer the item is bought from (Feature 3).
- **Inventory Adjustment**: a manager's correction to quantity on hand, with a reason.
- **Stock Lot**: stock of an item at one location (defined in Feature 5; adjustments add or remove lots).

---

## Data Model Requirements

### `items` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `description` | STRING(120) | Required |
| `caseUpc` | STRING(14) | Required, unique |
| `unitUpc` | STRING(14) | Required, unique |
| `unitsPerCase` | INTEGER | Required, greater than 0 |
| `price` | DECIMAL(10,2) | Required, ≥ 0; selling price per unit |
| `supplierCost` | DECIMAL(10,2) | Required, ≥ 0; cost per unit from the supplier |
| `minQty` | INTEGER | Nullable; reorder level in units |
| `maxQty` | INTEGER | Nullable; order-up-to level in units; must be > `minQty` |
| `qtyOnHand` | INTEGER | Default 0; system-maintained (sum of stock lots) |
| `qtyOnOrder` | INTEGER | Default 0; system-maintained (Feature 5) |
| `supplierId` | INTEGER FK | Required; references `suppliers.id` |
| `createdAt` / `updatedAt` | DATE | Sequelize timestamps |

### `inventory_adjustments` table
| Field | Type | Rules |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `itemId` | INTEGER FK | Required; references `items.id` |
| `userId` | INTEGER FK | Required; the manager; set from `req.user.id` |
| `oldQty` | INTEGER | Required |
| `newQty` | INTEGER | Required, ≥ 0 |
| `reason` | STRING(200) | Required |
| `createdAt` | DATE | Sequelize timestamp |

### Associations
*   `Supplier hasMany Item`, `Item belongsTo Supplier`
*   `Item hasMany InventoryAdjustment`, `InventoryAdjustment belongsTo Item` and `belongsTo User`
*   `Item hasMany StockLot` (Feature 5)

---

## Acceptance Criteria (Gherkin)

### US-2.1 — Add and edit items

#### Scenario: Office worker adds an item
*   **Given** I am signed in as an office worker
*   **When** I add an item with a description, case UPC, unit UPC, units per case, price, supplier cost, and supplier
*   **Then** the API returns `201` with the item
*   **And** the item appears in the item list

#### Scenario: Picker cannot add an item
*   **Given** I am signed in as a picker
*   **When** I send `POST /api/items`
*   **Then** the API returns `403`

#### Scenario: Item with stock cannot be deleted
*   **Given** an item has 30 on hand
*   **When** I delete the item
*   **Then** the API returns `409` with `{ "message": "Item has stock or open orders." }`

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
*   **Given** I am signed in as a manager editing reorder levels
*   **When** I set the minimum to 200 and the maximum to 50
*   **Then** the API returns `400` with `{ "message": "Minimum must be less than maximum." }`

### US-2.4 — See quantities

#### Scenario: Item list shows quantities
*   **Given** an item has 30 on hand and 100 on order
*   **When** I open the item list
*   **Then** I see 30 on hand and 100 on order for that item

### US-2.5 — Adjust quantity on hand

#### Scenario: Manager adjusts quantity with a reason
*   **Given** I am signed in as a manager
*   **And** an item has 60 on hand
*   **When** I adjust it to 48 with reason `Damaged cases removed`
*   **Then** the API returns `201`
*   **And** the item shows 48 on hand
*   **And** an adjustment record shows 60 → 48 with my user

#### Scenario: Adjustment needs a reason
*   **Given** I am signed in as a manager
*   **When** I adjust an item's quantity with no reason
*   **Then** the API returns `400` with `{ "message": "Reason is required." }`

---

## Definition of Done

*   [ ] Backend and frontend implemented per this spec (**FR-001 – FR-008** satisfied)
*   [ ] **Success Criteria (SC-001 – SC-004)** met
*   [ ] Role checks use `requireRole` from Feature 1, with no inline role checks
*   [ ] All Gherkin scenarios have tests, and `npm test` passes
*   [ ] `features/reference/api.md`, `data-model.md`, and `behavior.md` updated
*   [ ] Merged to `dev` through a pull request

---

## Out of Scope

*   Barcode scanner hardware integration (UPCs are typed or pasted)
*   Item images, categories, and multiple suppliers per item
*   Price history and scheduled price changes
*   Bulk import of items from a file
*   Physical inventory count sessions (a full cycle-count workflow). Adjustments are one item at a time; a count workflow is a candidate operational feature for the final requirements draft.
