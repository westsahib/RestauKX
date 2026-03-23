# 🏗️ RestauKX — Full System Architecture
### SaaS · MERN Stack · Web + Mobile + Local · Phase-Gated Construction

> **Version:** 1.0 — Pre-Implementation  
> **Stack:** MongoDB · Express.js · React · Node.js · React Native (Android)  
> **Deployment:** Cloud SaaS (web) + Local LAN mode (Android) + Offline-capable PWA  

---

## 1. Architectural Philosophy

RestauKX is designed as a **three-phase SaaS platform** with an offline-resilient core. Every phase adds capability without breaking what came before.

Three guiding principles inherited from the reference architecture:

1. **No data loss** — every order mutation is persisted immediately; no RAM-only state for billing data.
2. **Historical integrity** — finalized invoices are immutable; menu price changes never alter past orders.
3. **Atomic finalization** — invoice number assignment and order finalization happen in a single atomic operation.

---

## 2. High-Level System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                            │
│                                                                  │
│  ┌──────────────────┐   ┌─────────────────┐  ┌───────────────┐  │
│  │  React Web App   │   │ React Native    │  │  Customer QR  │  │
│  │  (Admin/Waiter)  │   │ Android App     │  │  Order Page   │  │
│  │  Phase 1+        │   │ Phase 2+        │  │  Phase 2+     │  │
│  └────────┬─────────┘   └───────┬─────────┘  └──────┬────────┘  │
└───────────│───────────────────── │─────────────────── │──────────┘
            │ HTTPS/REST           │ HTTPS/REST          │ HTTPS/REST
            └──────────────────────┼─────────────────────┘
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                       API SERVER (Node.js + Express)             │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌──────────────┐  │
│  │  Auth    │  │  Orders  │  │  Invoices  │  │  Menu/Items  │  │
│  │  Module  │  │  Module  │  │  Module    │  │  Module      │  │
│  └──────────┘  └──────────┘  └────────────┘  └──────────────┘  │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌──────────────┐  │
│  │  Tables  │  │  Staff   │  │  EventLog  │  │  Socket.IO   │  │
│  │  Module  │  │  Module  │  │  Service   │  │  (Phase 2+)  │  │
│  │ Phase 2+ │  │ Phase 3+ │  │  Phase 1+  │  │              │  │
│  └──────────┘  └──────────┘  └────────────┘  └──────────────┘  │
└──────────────────────────────────┬───────────────────────────────┘
                                   │ Mongoose ODM
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                        MONGODB ATLAS                             │
│                                                                  │
│  storeSettings · users · menuItems · orders · orderLines        │
│  finalOrders · invoiceSequence · eventLog · tables · staff      │
│  backupRecords · printJobs (Phase 2+) · notifications (P3+)     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Logical Layer Breakdown

The system follows a strict four-layer model. Each layer communicates only with the layer directly above or below it.

### Layer 1 — Presentation (React / React Native)
- All screens, components, modals, and user interactions.
- Reads data via API; dispatches actions via API calls or Socket.IO events.
- No business logic — purely orchestrates use-case calls.
- React Query handles server state caching and background refetch.

### Layer 2 — Domain / Use-case (Express Route Controllers + Services)
- Business rules: create draft, add line, finalize order, generate invoice, import menu.
- Validates input, enforces policy (e.g., order must be in DRAFT to add items).
- Manages atomic transactions for critical flows.
- Appends to EventLog for every significant action.

### Layer 3 — Persistence (Mongoose Models + MongoDB)
- Canonical entity storage: orders, orderLines, finalOrders, invoices, etc.
- Append-only EventLog collection.
- Atomic operations via MongoDB transactions (replica set / Atlas).
- Index strategy defined per collection.

### Layer 4 — Infrastructure
- JWT auth, bcrypt, Helmet.js (security).
- PDFKit (invoice PDF generation).
- Socket.IO (real-time events, Phase 2+).
- Cloud file storage for PDF URLs.
- Background job scheduler (node-cron) for exports and cleanup.

---

## 4. Core Domain Entities

These are the canonical MongoDB collections. Designed to be stable across all three phases — later phases add collections, never remove or mutate early ones.

### 4.1 `storeSettings` (one document per restaurant)
```js
{
  restaurantId:        ObjectId,        // FK → restaurants
  storeName:           String,
  gstNumber:           String,
  currency:            String,          // default: "INR"
  defaultTaxRate:      Number,          // e.g., 5 (%)
  packagingCharge:     Number,          // default parcel surcharge
  receiptFooter:       String,
  backupEnabled:       Boolean,
  undoWindowMs:        Number,          // finalize undo window, e.g. 3000
  autoExportEnabled:   Boolean,
  updatedAt:           Date
}
```

### 4.2 `menuItems` (catalog — changes affect future orders only)
```js
{
  _id:          ObjectId,
  restaurantId: ObjectId,
  itemCode:     String,               // unique per restaurant
  name:         String,
  category:     String,
  halfPrice:    Number | null,        // null if size not applicable
  fullPrice:    Number,
  gstPct:       Number,               // item-level GST %
  isAvailable:  Boolean,
  tags:         [String],             // ["veg","bestseller","spicy"]
  metadata:     Object,               // freeform extension field
  createdAt:    Date,
  updatedAt:    Date
}
```

> **Key rule:** changes to `menuItems` never retroactively affect finalized orders. All finalized orders use price snapshots (see `orderLines`).

### 4.3 `orders` (canonical mutable order — billing lifecycle)
```js
{
  _id:               ObjectId,
  orderUid:          String,          // globally unique, human-readable prefix
  restaurantId:      ObjectId,
  mode:              "DINE_IN" | "PARCEL" | "TAKEAWAY",
  tableRef:          String | null,   // table code for dine-in
  packagingCharge:   Number,          // prefilled from storeSettings for PARCEL
  billingStatus:     "DRAFT" | "PENDING_FINALIZE" | "FINALIZED" | "VOIDED",
  kitchenStatus:     "PENDING" | "CONFIRMED" | "PREPARING" | "READY" | "SERVED",
  subtotal:          Number,
  taxAmount:         Number,
  totalAmount:       Number,
  createdBy:         ObjectId,        // ref: users
  createdAt:         Date,
  updatedAt:         Date,
  finalizedAt:       Date | null
}
```

> **Note:** Two status fields coexist — `billingStatus` (invoice lifecycle, from the reference doc) and `kitchenStatus` (kitchen flow, Phase 2). They are independent state machines.

### 4.4 `orderLines` (line items — snapshot on write)
```js
{
  _id:                  ObjectId,
  orderUid:             String,         // FK → orders.orderUid
  restaurantId:         ObjectId,
  menuItemId:           ObjectId,       // ref for traceability only
  itemCodeSnapshot:     String,         // frozen at time of add
  itemNameSnapshot:     String,         // frozen — menu rename won't affect this
  size:                 "HALF" | "FULL" | "STANDARD",
  qty:                  Number,
  unitPriceSnapshot:    Number,         // frozen — price change won't affect this
  gstPctSnapshot:       Number,         // frozen
  lineTotalSnapshot:    Number,         // unitPrice × qty
  notes:                String,
  isImmutable:          Boolean,        // set true on FINALIZED
  createdAt:            Date
}
```

> **Key rule:** once `billingStatus` moves to `FINALIZED`, all associated `orderLines` are marked `isImmutable: true`. No further edits allowed.

### 4.5 `invoiceSequence` (atomic counter — one per restaurant)
```js
{
  restaurantId:   ObjectId,
  lastInvoiceNo:  Number,             // atomically incremented on finalize
  prefix:         String,             // e.g. "INV" → "INV-0042"
  updatedAt:      Date
}
```

> **Key rule:** `invoiceSequence.lastInvoiceNo` is incremented using `findOneAndUpdate` with `$inc` inside a MongoDB transaction. This guarantees no two orders ever share an invoice number.

### 4.6 `finalOrders` (immutable snapshot — written once on finalize)
```js
{
  _id:              ObjectId,
  orderUid:         String,
  restaurantId:     ObjectId,
  invoiceNumber:    String,           // e.g. "INV-0042" — never changes
  mode:             String,
  tableRef:         String | null,
  lines:            [ ...orderLine snapshots ],  // embedded copy
  subtotal:         Number,
  packagingCharge:  Number,
  taxAmount:        Number,
  totalAmount:      Number,
  paymentMethod:    "CASH" | "CARD" | "UPI",
  isPaid:           Boolean,
  pdfUrl:           String,
  finalizedAt:      Date,
  createdBy:        ObjectId
}
```

> **Key rule:** `finalOrders` documents are **never updated after creation**. If a correction is needed, a new `VOIDED` event is appended to EventLog and a replacement invoice is issued.

### 4.7 `eventLog` (append-only — never modified or deleted)
```js
{
  _id:           ObjectId,
  restaurantId:  ObjectId,
  timestamp:     Date,
  eventType:     String,     // See event catalogue below
  orderUid:      String | null,
  actorId:       ObjectId,   // user who triggered it
  payload:       Object,     // event-specific data
  processed:     Boolean     // for background job tracking
}
```

**Event catalogue (Phase 1):**

| Event Type | Trigger |
|---|---|
| `ORDER_DRAFT_CREATED` | New order initiated |
| `ORDER_LINE_ADDED` | Item added to order |
| `ORDER_LINE_UPDATED` | Item qty/notes changed |
| `ORDER_LINE_REMOVED` | Item removed from order |
| `ORDER_FINALIZE_STARTED` | Slide-to-confirm initiated |
| `ORDER_FINALIZE_UNDONE` | Undo triggered in window |
| `ORDER_FINALIZED` | Invoice committed atomically |
| `ORDER_VOIDED` | Order cancelled post-finalize |
| `MENU_ITEM_CREATED` | New item added |
| `MENU_ITEM_UPDATED` | Item edited |
| `MENU_IMPORT_COMPLETED` | Bulk import finished |
| `INVOICE_PDF_GENERATED` | PDF created |

**Additional events added in Phase 2+:** `TABLE_STATUS_CHANGED`, `KDS_ORDER_UPDATED`, `QR_ORDER_SUBMITTED`  
**Phase 3+:** `STAFF_SHIFT_STARTED`, `STAFF_MESSAGE_SENT`, `BACKUP_COMPLETED`

---

## 5. Key Flows (Phase-by-Phase)

### 5.1 New Order Creation (Phase 1)

```
User taps "New Order"
        │
        ▼
Modal: select mode
  ├── DINE_IN → require table selection
  └── PARCEL  → prefill packagingCharge from storeSettings
        │
        ▼
POST /api/orders
  → creates Order { billingStatus: "DRAFT" }
  → appends EventLog { ORDER_DRAFT_CREATED }
        │
        ▼
Draft persisted immediately to MongoDB
(survives server restart / client disconnect)
```

### 5.2 Order Line Add / Edit / Remove (Phase 1)

Every mutation:
1. Writes `orderLine` with price snapshot immediately.
2. Recalculates and updates `order.subtotal`, `taxAmount`, `totalAmount`.
3. Appends EventLog entry.
4. **No in-memory-only state** — React Query cache reflects persisted state.

### 5.3 Slide-to-Finalize Flow (Phase 1)

```
User triggers slide-to-confirm control
        │
        ▼
PATCH /api/orders/:id/status → { billingStatus: "PENDING_FINALIZE" }
EventLog: ORDER_FINALIZE_STARTED
        │
        ▼
Frontend starts undo countdown (undoWindowMs from storeSettings, e.g. 3000ms)
        │
   ┌────┴──────────────────────┐
   │ Undo triggered?           │ No undo → POST /api/orders/:id/finalize
   └───────────────────────────┘         (atomic MongoDB transaction):
           │                               1. $inc invoiceSequence.lastInvoiceNo
           ▼                               2. format invoiceNumber (e.g. "INV-0042")
   PATCH billingStatus → DRAFT             3. write finalOrder (immutable snapshot)
   EventLog: ORDER_FINALIZE_UNDONE         4. mark all orderLines isImmutable: true
                                           5. set order.billingStatus → FINALIZED
                                           6. set order.finalizedAt
                                           7. EventLog: ORDER_FINALIZED
                                           8. enqueue PDF generation job
```

**Guarantee:** invoice number is unique and gap-free. If the transaction fails at any step, MongoDB rolls back all changes atomically.

### 5.4 Real-Time Order Flow (Phase 2 — Socket.IO)

```
Customer scans QR at table
        │
        ▼
POST /api/orders/qr  (no auth — table token only)
  → creates DRAFT order linked to table
        │
        ▼
Socket.IO event emitted → "order:new" → Kitchen Display Screen
        │
        ▼
Kitchen updates status: CONFIRMED → PREPARING → READY
  → Socket.IO event → "order:status_update" → Waiter app
        │
        ▼
Waiter finalizes → atomic finalize flow (5.3)
```

### 5.5 Menu Import (Phase 1 — Admin)

```
Admin uploads CSV/Excel
        │
        ▼
Server parses file → preview mapping shown in UI
        │
        ▼
Admin selects policy:
  ├── ADD_NEW_ONLY              (safe default)
  ├── ADD_AND_UPDATE_EXISTING   (updates prices for future orders only)
  └── REPLACE_ENTIRE_MENU       (requires admin password confirmation)
        │
        ▼
Transactional batch write to menuItems
EventLog: MENU_IMPORT_COMPLETED { count, errors[] }

RULE: existing finalOrders and their line snapshots are never touched.
```

### 5.6 Sales Export (Phase 1)

- Query `finalOrders` by date range.
- Stream rows to CSV/Excel (do not load entire dataset into memory).
- Columns: `invoiceNumber`, `orderUid`, `mode`, `tableRef`, `itemNameSnapshot`, `qty`, `unitPriceSnapshot`, `lineTotalSnapshot`, `packagingCharge`, `taxAmount`, `totalAmount`, `paymentMethod`, `finalizedAt`.

---

## 6. Security Architecture

### Authentication (Phase 1)
- JWT access tokens (15min expiry) + refresh tokens (7 days).
- Stored: access token in memory (React), refresh token in `httpOnly` cookie.
- bcrypt for password hashing (cost factor 12).

### Role-Based Access Control

| Role | Phase Introduced | Permissions |
|---|---|---|
| `owner` | 1 | Full access including settings, export, menu replace |
| `admin` | 1 | Menu CRUD, order management, invoice history |
| `waiter` | 1 | Create/edit DRAFT orders, finalize |
| `kitchen` | 2 | View & update kitchenStatus only |
| `staff` | 3 | Clock-in/out, messaging |

### Sensitive Operation Guards (Phase 1)
Operations requiring admin password re-confirmation:
- Menu replace (REPLACE_ENTIRE_MENU)
- Export all sales data
- Modify storeSettings
- Void a finalized invoice

### HTTP Security (Phase 1)
- Helmet.js — sets secure HTTP headers.
- express-rate-limit — 100 req/15min per IP (public routes), 500/15min (authenticated).
- express-validator — input sanitization on all routes.
- CORS restricted to known frontend origins.

---

## 7. Data Integrity & Reliability

### 7.1 Instant Writes
Every order mutation (line add/update/remove) is written to MongoDB before the API responds. React Query optimistic updates reflect the server state.

### 7.2 Atomic Transactions
MongoDB multi-document transactions (requires replica set — use Atlas M10+) are used for:
- Order finalization (invoiceSequence increment + finalOrder write + orderLines lock).
- Menu replace (drop + re-insert in single transaction).

### 7.3 EventLog as Safety Net
On server startup, a health check scans for orders stuck in `PENDING_FINALIZE` older than 2× `undoWindowMs`. These are either rolled back to `DRAFT` (if no finalOrder exists) or reconciled forward (if finalOrder exists but order status wasn't updated).

### 7.4 Historical Safety
- `finalOrders` documents: never updated after creation.
- `orderLines` with `isImmutable: true`: rejected on any update attempt at the API level.
- `menuItems` updates: only affect orders created after the update.

### 7.5 Invoice Number Guarantee
`invoiceSequence` uses MongoDB's atomic `$inc` inside a transaction. Concurrent finalization attempts cannot produce the same number or skip numbers.

---

## 8. Performance Targets & Strategies

| Operation | Target | Strategy |
|---|---|---|
| Order line add/remove | < 100ms | Indexed writes, no joins |
| Invoice finalize | < 500ms | Single atomic transaction |
| Menu page load | < 300ms | Indexed + cached by React Query |
| Sales export (1000 invoices) | < 5s | Streaming cursor, no full load |
| Socket.IO order event | < 200ms | In-memory pub/sub |

### MongoDB Index Strategy
```js
// orders
{ restaurantId: 1, billingStatus: 1, createdAt: -1 }
{ orderUid: 1 }   // unique

// orderLines
{ orderUid: 1 }
{ restaurantId: 1, isImmutable: 1 }

// finalOrders
{ restaurantId: 1, invoiceNumber: 1 }  // unique
{ restaurantId: 1, finalizedAt: -1 }

// eventLog
{ restaurantId: 1, timestamp: -1 }
{ orderUid: 1, eventType: 1 }

// menuItems
{ restaurantId: 1, isAvailable: 1, category: 1 }
{ restaurantId: 1, itemCode: 1 }  // unique
```

---

## 9. Phase-by-Phase Construction Plan

### Phase 1 — Automated Invoicing (Foundation)

**New collections introduced:** `storeSettings`, `users`, `menuItems`, `orders`, `orderLines`, `invoiceSequence`, `finalOrders`, `eventLog`

**New API modules:**
- `/auth` — register, login, refresh, logout
- `/store` — storeSettings CRUD
- `/menu` — menuItems CRUD + bulk import
- `/orders` — DRAFT lifecycle (create, add lines, update lines, finalize, void)
- `/invoices` — finalOrders query, PDF generation, download
- `/export` — streaming CSV/Excel export

**React Web App screens:**
- Login / Register
- Admin Dashboard (daily stats, recent orders)
- New Order (mode select → table select → menu browse → add lines → finalize)
- Order History
- Invoice Detail + PDF download
- Menu Management
- Store Settings

**EventLog events active:** all Phase 1 events from catalogue (Section 4.7)

---

### Phase 2 — Table-Side Dine-In Ordering

**New collections introduced:** `tables`, `printJobs`

**New capabilities added to existing collections:**
- `orders.kitchenStatus` field activated (was stored from Phase 1 but unused)
- `eventLog` — Phase 2 events added

**New API modules:**
- `/tables` — layout management, status, QR code generation
- `/qr` — unauthenticated table order submission endpoint
- `/kds` — kitchen display read + status update endpoints
- Socket.IO namespaces: `/kitchen`, `/waiter`, `/table`

**React Web App additions:**
- Table Map screen (admin/waiter)
- Kitchen Display Screen (web, large screen)
- Live Order Queue

**React Native Android additions:**
- Waiter app — table map, order management, finalize
- Kitchen app — incoming orders, status updates
- Push notifications (Firebase Cloud Messaging)

**New flows active:**
- QR scan → customer order submission → kitchen notification
- Kitchen status updates → real-time waiter notification
- Socket.IO order lifecycle events

---

### Phase 3 — Admin · Kitchen · Staff Console

**New collections introduced:** `staff`, `shifts`, `messages`, `notifications`

**New API modules:**
- `/staff` — CRUD, role assignment
- `/shifts` — clock-in/out, shift history
- `/messages` — in-app staff messaging
- `/notifications` — push + in-app notification management
- `/analytics` — aggregation pipeline queries (revenue, top items, peak hours)

**React Web App additions:**
- Analytics Dashboard (revenue trends, heatmaps, item performance)
- Staff Management
- Messaging Console
- Advanced Settings (backup config, retention, branding)

**EventLog events active:** all Phase 3 events

**New adapter interfaces (extensibility hooks):**
- `BackupAdapter` — scheduled encrypted export to cloud storage
- `PrinterAdapter` — thermal printer integration via printJobs queue
- Analytics webhook — replay EventLog to external BI tools

---

## 10. Adapter & Extensibility Interfaces

These interfaces are defined in Phase 1 and implemented progressively:

```
BackupAdapter
  └── exportToFile(restaurantId, format) → BackupRecord
  └── uploadToCloud(backupRecord) → url

PrinterAdapter (Phase 2)
  └── enqueuePrintJob(finalOrderId, type) → PrintJob
  └── retryFailedJobs() → void

StorageAdapter
  └── savePDF(buffer, filename) → url
  └── getSignedUrl(filename) → url

AnalyticsAdapter (Phase 3)
  └── replayEventLog(restaurantId, fromDate) → void
  └── pushEvent(event) → void
```

All adapters are injected via the service layer — swappable without touching business logic.

---

## 11. Startup Health Checks

On every server start, the following checks run automatically:

1. **InvoiceSequence integrity** — verify `lastInvoiceNo` ≥ max(`invoiceNumber`) in `finalOrders`. Auto-correct if drifted.
2. **Stuck PENDING_FINALIZE orders** — any order in `PENDING_FINALIZE` older than 2× `undoWindowMs` is rolled back to `DRAFT` and an EventLog entry is appended.
3. **Orphan orderLines** — lines referencing non-existent `orderUid` are flagged and logged.
4. **Stale temp files** — incomplete PDF exports or temp upload files older than 24h are deleted.

---

## 12. Repository Structure (Implementation-Ready)

```
RestauKX/
├── src/
│   ├── client/                    # React web app (Phase 1+)
│   │   ├── components/
│   │   ├── pages/
│   │   │   ├── dashboard/
│   │   │   ├── orders/
│   │   │   ├── menu/
│   │   │   ├── invoices/
│   │   │   ├── tables/            # Phase 2+
│   │   │   ├── staff/             # Phase 3+
│   │   │   └── analytics/         # Phase 3+
│   │   ├── hooks/
│   │   ├── services/              # API call wrappers
│   │   └── store/                 # React Query setup
│   │
│   ├── server/                    # Node.js + Express
│   │   ├── routes/
│   │   │   ├── auth.routes.js
│   │   │   ├── orders.routes.js
│   │   │   ├── invoices.routes.js
│   │   │   ├── menu.routes.js
│   │   │   ├── tables.routes.js   # Phase 2+
│   │   │   └── staff.routes.js    # Phase 3+
│   │   ├── controllers/
│   │   ├── services/
│   │   │   ├── order.service.js
│   │   │   ├── finalize.service.js
│   │   │   ├── eventLog.service.js
│   │   │   ├── pdf.service.js
│   │   │   └── export.service.js
│   │   ├── models/                # Mongoose schemas
│   │   │   ├── Order.js
│   │   │   ├── OrderLine.js
│   │   │   ├── FinalOrder.js
│   │   │   ├── InvoiceSequence.js
│   │   │   ├── EventLog.js
│   │   │   ├── MenuItem.js
│   │   │   └── StoreSettings.js
│   │   ├── middleware/
│   │   │   ├── auth.middleware.js
│   │   │   ├── rbac.middleware.js
│   │   │   └── errorHandler.js
│   │   ├── adapters/              # Swappable infrastructure
│   │   │   ├── backup.adapter.js
│   │   │   ├── printer.adapter.js
│   │   │   └── storage.adapter.js
│   │   └── startup/
│   │       └── healthChecks.js
│   │
│   └── shared/                    # Shared across client + server
│       ├── constants/
│       │   ├── orderStatus.js     # DRAFT, PENDING_FINALIZE, FINALIZED, VOIDED
│       │   └── eventTypes.js      # Full event catalogue
│       └── utils/
│           └── invoiceFormatter.js
│
├── mobile/                        # React Native Android (Phase 2+)
│   ├── screens/
│   ├── components/
│   └── services/
│
├── docs/
│   ├── architecture/
│   │   ├── system-architecture.md   ← this document
│   │   └── database-schema.md
│   ├── api/
│   │   └── api-spec.md
│   ├── phases/
│   │   └── roadmap.md
│   ├── TECH-STACK.md
│   └── CONTRIBUTING.md
│
└── ui-mockups/
    ├── web/
    └── android/
```

---

## 13. Summary

RestauKX is a three-phase MERN SaaS platform built on an offline-resilient core. Phase 1 delivers the complete billing engine — atomic invoice finalization, immutable final orders, price-snapshot order lines, and an append-only EventLog — all running on React + Node.js + MongoDB Atlas. Phase 2 adds real-time table-side ordering via Socket.IO and a React Native Android app. Phase 3 completes the platform with staff management, analytics, and the full admin console.

The separation of `billingStatus` (invoice lifecycle) from `kitchenStatus` (operations flow), the atomic `InvoiceSequence`, and the immutable `finalOrders` collection are the three architectural decisions that guarantee data integrity at scale.

---

*RestauKX · System Architecture v1.0 · Pre-Implementation*
