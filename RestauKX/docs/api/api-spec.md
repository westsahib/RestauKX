# API Specification — RestauKX

**Base URL:** `https://api.restaukx.com/v1`
**Auth:** `Authorization: Bearer <accessToken>` on all protected routes
**Format:** JSON · UTF-8
**Versioning:** URI-based (`/v1`) — breaking changes increment the version

---

## Authentication

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/auth/register` | — | Register restaurant + owner account |
| POST | `/auth/login` | — | Login, receive token pair |
| POST | `/auth/refresh` | — | Exchange refresh token for new access token |
| POST | `/auth/logout` | ✓ | Invalidate refresh token |

### POST `/auth/register`
```json
// Request
{
  "storeName": "Spice Garden",
  "ownerName": "Ravi Kumar",
  "email": "ravi@spicegarden.com",
  "password": "••••••••",
  "gstNumber": "33AABCU9603R1ZM",
  "currency": "INR"
}

// Response 201
{
  "restaurantId": "663f...",
  "accessToken": "eyJ...",
  "refreshToken": "eyJ..."
}
```

### POST `/auth/login`
```json
// Request
{ "email": "ravi@spicegarden.com", "password": "••••••••" }

// Response 200
{
  "accessToken": "eyJ...",
  "refreshToken": "eyJ...",
  "user": { "id": "...", "name": "Ravi Kumar", "role": "owner" }
}
```

---

## Store Settings

| Method | Endpoint | Role | Description |
|---|---|---|---|
| GET | `/store/settings` | any | Get current store configuration |
| PATCH | `/store/settings` | owner, admin | Update settings (requires password confirm for tax/GST changes) |

### PATCH `/store/settings` — Body
```json
{
  "defaultTaxRate": 5,
  "packagingCharge": 20,
  "undoWindowMs": 3000,
  "receiptFooter": "Thank you for dining with us!",
  "autoExportEnabled": false
}
```

---

## Menu Items

| Method | Endpoint | Role | Description |
|---|---|---|---|
| GET | `/menu` | any | List all items (filterable by category, availability) |
| GET | `/menu/:id` | any | Get single item |
| POST | `/menu` | admin, owner | Create item |
| PUT | `/menu/:id` | admin, owner | Update item (future orders only — no history mutation) |
| DELETE | `/menu/:id` | admin, owner | Soft-delete item |
| PATCH | `/menu/:id/availability` | admin, owner, waiter | Toggle availability |
| POST | `/menu/import` | owner | Bulk import via CSV/Excel upload |

### POST `/menu` — Body
```json
{
  "itemCode": "CHK-TKM-01",
  "name": "Chicken Tikka Masala",
  "category": "Main Course",
  "halfPrice": null,
  "fullPrice": 280,
  "gstPct": 5,
  "tags": ["non-veg", "bestseller"],
  "isAvailable": true
}
```

### POST `/menu/import` — Multipart Form
```
Field: file       → .csv or .xlsx
Field: policy     → "ADD_NEW_ONLY" | "ADD_AND_UPDATE" | "REPLACE_ALL"
Field: adminPass  → required only when policy = "REPLACE_ALL"
```

---

## Orders

| Method | Endpoint | Role | Description |
|---|---|---|---|
| GET | `/orders` | admin, owner, waiter | List orders — filter by `billingStatus`, `kitchenStatus`, `date`, `mode` |
| POST | `/orders` | waiter, admin | Create new DRAFT order |
| GET | `/orders/:uid` | any | Get order with lines |
| POST | `/orders/:uid/lines` | waiter | Add line to DRAFT order |
| PATCH | `/orders/:uid/lines/:lineId` | waiter | Update qty / notes on a line |
| DELETE | `/orders/:uid/lines/:lineId` | waiter | Remove line from DRAFT order |
| PATCH | `/orders/:uid/kitchen-status` | kitchen, waiter | Update kitchen status |
| POST | `/orders/:uid/finalize` | waiter, admin | Atomic finalization → creates FinalOrder + invoice |
| POST | `/orders/:uid/void` | owner | Void a finalized order (admin password required) |

### POST `/orders` — Body
```json
{
  "mode": "DINE_IN",
  "tableRef": "T5"
}
// mode = "PARCEL" → packagingCharge auto-filled from storeSettings
```

### POST `/orders/:uid/lines` — Body
```json
{
  "menuItemId": "663a...",
  "size": "FULL",
  "qty": 2,
  "notes": "extra spicy, no onion"
}
// Server snapshots: itemCode, itemName, unitPrice, gstPct at write time
```

### POST `/orders/:uid/finalize` — Body
```json
{ "paymentMethod": "UPI" }

// Response 201 — after atomic transaction completes
{
  "invoiceNumber": "INV-0042",
  "finalOrderId": "664b...",
  "totalAmount": 1071,
  "pdfUrl": "https://cdn.restaukx.com/invoices/INV-0042.pdf"
}
```

---

## Final Orders / Invoices

| Method | Endpoint | Role | Description |
|---|---|---|---|
| GET | `/invoices` | admin, owner | List finalized invoices — filter by date, payment method, tableRef |
| GET | `/invoices/:id` | admin, owner, waiter | Get invoice detail with line snapshot |
| GET | `/invoices/:id/pdf` | any | Download PDF |
| PATCH | `/invoices/:id/paid` | waiter, admin | Mark invoice as paid (cash flow) |
| GET | `/invoices/export` | owner, admin | Stream CSV/Excel export by date range |

### GET `/invoices/export` — Query Params
```
?from=2026-03-01&to=2026-03-31&format=csv
```
Response: streamed file download — does not buffer entire dataset in memory.

---

## Tables *(Phase 2)*

| Method | Endpoint | Role | Description |
|---|---|---|---|
| GET | `/tables` | any | List all tables with live status |
| POST | `/tables` | admin | Add table to layout |
| PATCH | `/tables/:id` | admin | Edit table (capacity, floor, code) |
| DELETE | `/tables/:id` | admin | Remove table |
| PATCH | `/tables/:id/status` | waiter | Manually update status |
| GET | `/tables/:id/qr` | admin | Get QR code (PNG + URL) |
| POST | `/orders/qr` | — (table token) | Customer submits order via QR — no auth required |

### POST `/orders/qr` — Body (unauthenticated)
```json
{
  "tableToken": "qt_T5_abc123",
  "items": [
    { "itemCode": "CHK-TKM-01", "size": "FULL", "qty": 1 },
    { "itemCode": "BRD-NAN-01", "size": "STANDARD", "qty": 2 }
  ]
}
```

---

## Kitchen Display *(Phase 2)*

| Method | Endpoint | Role | Description |
|---|---|---|---|
| GET | `/kds/queue` | kitchen | Live order queue — active DINE_IN orders |
| PATCH | `/kds/orders/:uid/status` | kitchen | Advance kitchen status |

Socket.IO events (namespace `/kitchen`):

| Event | Direction | Payload |
|---|---|---|
| `order:new` | server → kitchen | `{ orderUid, tableRef, lines[] }` |
| `order:status_update` | server → waiter | `{ orderUid, kitchenStatus }` |
| `order:finalized` | server → all | `{ orderUid, invoiceNumber }` |

---

## Staff & Shifts *(Phase 3)*

| Method | Endpoint | Role | Description |
|---|---|---|---|
| GET | `/staff` | admin, owner | List staff members |
| POST | `/staff` | admin, owner | Add staff member |
| PUT | `/staff/:id` | admin, owner | Update details |
| DELETE | `/staff/:id` | owner | Deactivate staff |
| POST | `/shifts/clock-in` | staff | Clock in |
| POST | `/shifts/clock-out` | staff | Clock out |
| GET | `/shifts` | admin, owner | View shift history |

---

## Analytics *(Phase 3)*

| Method | Endpoint | Description |
|---|---|---|
| GET | `/analytics/revenue` | Daily/weekly/monthly revenue aggregation |
| GET | `/analytics/items` | Top-selling items by qty and revenue |
| GET | `/analytics/peak-hours` | Order count by hour of day (heatmap data) |
| GET | `/analytics/order-types` | DINE_IN vs PARCEL vs TAKEAWAY breakdown |

All analytics endpoints accept `?from=` and `?to=` date range params.

---

## Event Log *(Internal — Admin only)*

| Method | Endpoint | Description |
|---|---|---|
| GET | `/events` | Query event log — filter by `orderUid`, `eventType`, `actorId`, date |

---

## Error Response Format

All errors follow a consistent envelope:

```json
{
  "success": false,
  "error": {
    "code": "IMMUTABLE_ORDER_LINE",
    "message": "Cannot modify a line on a FINALIZED order.",
    "field": "qty"
  }
}
```

### Error Code Catalogue

| Code | HTTP | Meaning |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Input failed schema validation |
| `UNAUTHORIZED` | 401 | Missing or expired token |
| `FORBIDDEN` | 403 | Insufficient role for this action |
| `NOT_FOUND` | 404 | Resource does not exist |
| `IMMUTABLE_ORDER_LINE` | 409 | Attempt to edit a finalized order line |
| `INVOICE_SEQ_CONFLICT` | 409 | Race condition on invoice number (retry) |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unhandled server error |

---

*API Specification v1.0 · RestauKX · Pre-Implementation*
