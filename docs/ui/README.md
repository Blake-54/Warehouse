# UI design — Warehouse Management System

The screen designs are made in **Figma**. This folder holds the exported images, one sub-folder per feature, plus the screen hierarchy and flow. The **Screen Requirements** section in each feature spec is still the source of truth for labels, buttons, and states; the images show the layout.

**Figma file:** _paste the Figma link here (Share → Copy link, "Anyone with the link can view")_

---

## 1. Screen hierarchy

Export as `docs/ui/screen-hierarchy.png`.

```text
Login  (F1)
└── Home: role-based menu cards  (F1)
    ├── Items  (F2)
    │   ├── Item dialog (new/edit)
    │   ├── Reorder Levels dialog        [manager]
    │   └── Adjust Quantity dialog       [manager]
    ├── Suppliers  (F3)
    │   └── Supplier dialog (new/edit)
    ├── Customers  (F4)
    │   └── Customer dialog (new/edit, route select)
    ├── Supplier Orders  (F5)
    │   ├── Receive Order               [receiver, manager]
    │   └── Print Order
    ├── Customer Orders  (F6)           [office, manager]
    │   └── Order Detail
    ├── My Orders  (F6)                 [customer]
    │   ├── New Order (catalog + cart)
    │   └── Order Detail
    ├── Pick Queue  (F6)                [picker, manager]
    │   └── Pick List
    ├── Truck Routes  (F7)              [office, manager]
    │   └── Route Assignments           [manager]
    └── My Route  (F7)                  [driver]
```

### Which cards each role sees on Home

| Role | Cards |
|------|-------|
| Office Worker | Items, Suppliers, Customers, Supplier Orders, Customer Orders, Truck Routes |
| Manager | Items, Suppliers, Customers, Supplier Orders, Customer Orders, Pick Queue, Truck Routes |
| Receiver | Items, Supplier Orders |
| Picker | Items, Pick Queue |
| Driver | My Route |
| Customer | My Orders |

---

## 2. Screen flows (Figma prototype)

Export the prototype overview as `docs/ui/screen-flow.png`, and put the prototype link at the top of this file. Make at least flows **A** and **B** clickable.

| Flow | Steps |
|------|-------|
| **A. Customer places an order** | Login → Home → My Orders → **+ New Order** → New Order (search, **Add**, one item shows **Out of stock**) → **Submit Order** → Order Detail (status **New**) |
| **B. Picker fills an order** | Login → Home → Pick Queue → **Start Picking** (Bin Number dialog) → Pick List → **Record** each row (one short) → **Finish Picking** → Pick Queue |
| **C. Receiver checks a delivery** | Home → Supplier Orders → **Receive** → Receive Order (8 of 10 cases, "2 cases short", split location) → **Save Receipt** |
| **D. Driver ships** | Login → My Route → **Mark Shipped** → confirm → order shows **Shipped** |
| **E. Manager plans routes** | Home → Truck Routes → **Assignments** → pick date, truck, driver → **Save** |
| **F. Office maintains data** | Home → Items → **+ New Item** → Save; Suppliers → **+ New Supplier**; Customers → **+ New Customer** (assign route) |

---

## 3. Screen inventory and image files

Frame sizes: **Desktop 1440 × 900** unless noted. Floor screens use **Tablet 1024 × 768**; the driver screen uses **Phone 390 × 844**.

| Feature | Screen | Frame | Export to | Spec |
|---------|--------|-------|-----------|------|
| F1 | Login | Desktop | `feature-1/login.png` | [Feature 1](../../features/feature-1-user-login.md#screen-requirements) |
| F1 | Home (show the office-worker version) | Desktop | `feature-1/home.png` | 〃 |
| F2 | Items list | Desktop | `feature-2/items.png` | [Feature 2](../../features/feature-2-item-management.md#screen-requirements) |
| F2 | Item dialog | Desktop | `feature-2/item-dialog.png` | 〃 |
| F2 | Reorder Levels dialog | Desktop | `feature-2/reorder-levels-dialog.png` | 〃 |
| F2 | Adjust Quantity dialog | Desktop | `feature-2/adjust-quantity-dialog.png` | 〃 |
| F3 | Suppliers list | Desktop | `feature-3/suppliers.png` | [Feature 3](../../features/feature-3-supplier-management.md#screen-requirements) |
| F3 | Supplier dialog | Desktop | `feature-3/supplier-dialog.png` | 〃 |
| F4 | Customers list | Desktop | `feature-4/customers.png` | [Feature 4](../../features/feature-4-customer-management.md#screen-requirements) |
| F4 | Customer dialog | Desktop | `feature-4/customer-dialog.png` | 〃 |
| F5 | Supplier Orders list | Desktop | `feature-5/supplier-orders.png` | [Feature 5](../../features/feature-5-supplier-ordering-receiving.md#screen-requirements) |
| F5 | Receive Order | Tablet | `feature-5/receive-order.png` | 〃 |
| F5 | Print Order | Desktop (letter page) | `feature-5/print-order.png` | 〃 |
| F6 | My Orders | Desktop | `feature-6/my-orders.png` | [Feature 6](../../features/feature-6-customer-ordering-picking.md#screen-requirements) |
| F6 | New Order | Desktop | `feature-6/new-order.png` | 〃 |
| F6 | Order Detail | Desktop | `feature-6/order-detail.png` | 〃 |
| F6 | Customer Orders | Desktop | `feature-6/customer-orders.png` | 〃 |
| F6 | Pick Queue | Tablet | `feature-6/pick-queue.png` | 〃 |
| F6 | Pick List | Tablet | `feature-6/pick-list.png` | 〃 |
| F7 | Truck Routes | Desktop | `feature-7/truck-routes.png` | [Feature 7](../../features/feature-7-shipping-routes.md#screen-requirements) |
| F7 | Route Assignments | Desktop | `feature-7/route-assignments.png` | 〃 |
| F7 | My Route | Phone | `feature-7/my-route.png` | 〃 |

**Export:** select the frame → **Export** panel (right side) → **PNG**, **2x** → save with the file name above.

---

## 4. Layout notes per screen (what to draw)

Use the exact labels from each feature's **Screen Requirements**. Shared pieces to build once as Figma **components**: MenuBar, page header (title + primary button), data table row, status chip, confirm dialog, error alert.

- **MenuBar:** maroon bar, "Warehouse" on the left; on the right, user name, role chip, and **Sign out**. Hidden on Login.
- **Login:** centered card (~400 px wide) with title **Warehouse Sign In**, Username, Password with eye icon, full-width **Sign in**; show the error alert state in a second frame.
- **Home:** **Welcome, {name}** heading + role chip; grid of large cards (icon + title + one-line description).
- **Lists (Items, Suppliers, Customers, Supplier Orders, Customer Orders, Truck Routes):** page title left, primary **+ New …** button right, search/filter row, table, icon actions at the end of each row. Draw one frame with data and one **empty state**.
- **Dialogs:** ~560 px wide card over a dimmed page; title, fields, **Cancel** (text) + primary button (maroon).
- **Receive Order:** a table with a number input in **Cases Received Now**, an orange "2 cases short" text, and a **+ Split location** link that shows a second Aisle/Side/Cases row.
- **New Order:** catalog table on the left (one row with a red **Out of stock** chip and a disabled **Add**); cart card on the right with lines, total, and **Submit Order**.
- **Pick List:** big rows, sorted by Aisle → Side; a **Picked** number input and **Record** button per row; a green check on recorded rows; a progress bar and **Finish Picking** at the bottom.
- **My Route:** phone frame; header card with route, truck, and date; one card per store with **Mark Shipped**; a **Shipped** chip state.

---

## 5. Design tokens (match the code)

From [ui-style-system.mdc](../../.cursor/rules/ui-style-system.mdc). Create these as Figma color and text styles:

| Token | Hex | Use |
|-------|-----|-----|
| primary | `#801328` | MenuBar, primary buttons, headings |
| primary-container | `#FFDAD9` | Chips, selected rows |
| secondary | `#775656` | Secondary buttons, icons |
| surface | `#F9F9FF` | Cards and dialogs |
| surface-variant | `#E7E0E1` | Table header fill, dividers |
| background | `#FFFFFF` | Page |
| outline | `#857373` | Borders |
| body text | `#1C1B1F` | Body |

Status chip colors (suggested): New = blue, Picking = amber, Picked = teal, Shipped = green, Cancelled = grey; Open = blue, Partial = amber, Received = green, Closed = grey.

Type: **Inter**; 16 px body; headings 700 weight in primary (20 / 25 / 31 px). Corner radius **8 px** everywhere. Spacing on a 4 px grid (8, 16, 24, 32).
