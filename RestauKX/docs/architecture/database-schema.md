# 🗄️ Database Schema — RestauKX (MongoDB)

All collections are scoped by `restaurantId` for multi-tenant isolation.  
Collections are introduced phase-by-phase — later phases never modify Phase 1 schemas.

---

## Phase 1 Collections

### `storeSettings`
One document per restaurant. Separates operational config from identity.
```js
{
  restaurantId:       ObjectId,      // ref: restaurants
  storeName:          String,
  gstNumber:          String,
  currency:           String,        // default: "INR"
  defaultTaxRate:     Number,        // %
  packagingCharge:    Number,        // default surcharge for PARCEL orders
  receiptFooter:      String,
  undoWindowMs:       Number,        // finalize undo window (e.g. 3000)
  autoExportEnabled:  Boolean,
  backupEnabled:      Boolean,
  updatedAt:          Date
}
```

### `users`
```js
{
  _id:          ObjectId,
  restaurantId: ObjectId,
  name:         String,
  email:        String,             // unique per restaurant
  passwordHash: String,
  role:         "owner"|"admin"|"waiter"|"kitchen"|"staff",
  isActive:     Boolean,
  createdAt:    Date,
  updatedAt:    Date
}
```

### `menuItems`
```js
{
  _id:          ObjectId,
  restaurantId: ObjectId,
  itemCode:     String,             // unique per restaurant
  name:         String,
  category:     String,
  halfPrice:    Number | null,      // null if size not applicable
  fullPrice:    Number,
  gstPct:       Number,             // item-level GST %
  isAvailable:  Boolean,
  tags:         [String],           // ["veg","non-veg","spicy","bestseller"]
  metadata:     Object,             // freeform extension field
  createdAt:    Date,
  updatedAt:    Date
}
```
> Price changes here never affect existing `finalOrders` — those use snapshots.

### `orders`
```js
{
  _id:            ObjectId,
  orderUid:       String,           // unique, human-readable (e.g. "ORD-20260323-0041")
  restaurantId:   ObjectId,
  mode:           "DINE_IN"|"PARCEL"|"TAKEAWAY",
  tableRef:       String | null,    // table code for DINE_IN
  packagingCharge: Number,          // prefilled from storeSettings for PARCEL
  billingStatus:  "DRAFT"|"PENDING_FINALIZE"|"FINALIZED"|"VOIDED",
  kitchenStatus:  "PENDING"|"CONFIRMED"|"PREPARING"|"READY"|"SERVED",
  subtotal:       Number,
  taxAmount:      Number,
  totalAmount:    Number,
  createdBy:      ObjectId,         // ref: users
  createdAt:      Date,
  updatedAt:      Date,
  finalizedAt:    Date | null
}
```
> `billingStatus` = invoice lifecycle. `kitchenStatus` = kitchen operations. Independent state machines.

### `orderLines`
```js
{
  _id:                ObjectId,
  orderUid:           String,        // FK → orders.orderUid
  restaurantId:       ObjectId,
  menuItemId:         ObjectId,      // ref for traceability only
  itemCodeSnapshot:   String,        // frozen at time of add
  itemNameSnapshot:   String,        // frozen — menu rename won't affect this
  size:               "HALF"|"FULL"|"STANDARD",
  qty:                Number,
  unitPriceSnapshot:  Number,        // frozen — price changes don't affect this
  gstPctSnapshot:     Number,        // frozen
  lineTotalSnapshot:  Number,        // unitPrice × qty
  notes:              String,
  isImmutable:        Boolean,       // set true when order is FINALIZED
  createdAt:          Date
}
```
> Once `isImmutable: true`, the API rejects all update attempts on this line.

### `invoiceSequence`
One document per restaurant. Atomically incremented on every finalization.
```js
{
  restaurantId:  ObjectId,
  lastInvoiceNo: Number,            // $inc atomically inside transaction
  prefix:        String,            // e.g. "INV" → produces "INV-0042"
  updatedAt:     Date
}
```

### `finalOrders`
Immutable snapshot written once at finalization. Never updated.
```js
{
  _id:             ObjectId,
  orderUid:        String,
  restaurantId:    ObjectId,
  invoiceNumber:   String,          // e.g. "INV-0042" — permanent, unique
  mode:            String,
  tableRef:        String | null,
  lines: [{                         // embedded snapshot — fully self-contained
    itemCodeSnapshot:  String,
    itemNameSnapshot:  String,
    size:              String,
    qty:               Number,
    unitPriceSnapshot: Number,
    gstPctSnapshot:    Number,
    lineTotalSnapshot: Number,
    notes:             String
  }],
  subtotal:        Number,
  packagingCharge: Number,
  taxAmount:       Number,
  totalAmount:     Number,
  paymentMethod:   "CASH"|"CARD"|"UPI",
  isPaid:          Boolean,
  pdfUrl:          String,
  finalizedAt:     Date,
  createdBy:       ObjectId
}
```

### `eventLog`
Append-only. Never modified or deleted (except archiving).
```js
{
  _id:          ObjectId,
  restaurantId: ObjectId,
  timestamp:    Date,
  eventType:    String,             // e.g. "ORDER_FINALIZED"
  orderUid:     String | null,
  actorId:      ObjectId,           // user who triggered the action
  payload:      Object,             // event-specific data
  processed:    Boolean             // for background job tracking
}
```

---

## Phase 2 Collections

### `tables`
```js
{
  _id:            ObjectId,
  restaurantId:   ObjectId,
  tableCode:      String,           // e.g. "T5", "BAR-2"
  capacity:       Number,
  status:         "AVAILABLE"|"OCCUPIED"|"RESERVED",
  currentOrderUid: String | null,
  qrCodeUrl:      String,
  floor:          String,           // e.g. "Ground", "Terrace"
  createdAt:      Date
}
```

### `printJobs`
```js
{
  _id:          ObjectId,
  restaurantId: ObjectId,
  finalOrderId: ObjectId,
  type:         "KOT"|"INVOICE"|"REPRINT",
  status:       "PENDING"|"SENT"|"FAILED",
  retryCount:   Number,
  createdAt:    Date,
  sentAt:       Date | null
}
```

---

## Phase 3 Collections

### `staff`
```js
{
  _id:          ObjectId,
  restaurantId: ObjectId,
  userId:       ObjectId,           // ref: users
  designation:  String,
  phone:        String,
  joinedAt:     Date,
  isActive:     Boolean
}
```

### `shifts`
```js
{
  _id:          ObjectId,
  restaurantId: ObjectId,
  staffId:      ObjectId,
  clockIn:      Date,
  clockOut:     Date | null,
  durationMins: Number | null
}
```

### `messages`
```js
{
  _id:          ObjectId,
  restaurantId: ObjectId,
  fromUserId:   ObjectId,
  toRole:       String,             // broadcast to role, or...
  toUserId:     ObjectId | null,    // ...direct to user
  body:         String,
  readBy:       [ObjectId],
  createdAt:    Date
}
```

### `notifications`
```js
{
  _id:          ObjectId,
  restaurantId: ObjectId,
  userId:       ObjectId,
  type:         String,             // "ORDER_READY", "NEW_MESSAGE", etc.
  payload:      Object,
  isRead:       Boolean,
  createdAt:    Date
}
```

---

## Index Strategy

```js
// orders
{ orderUid: 1 }                                          // unique
{ restaurantId: 1, billingStatus: 1, createdAt: -1 }

// orderLines
{ orderUid: 1 }
{ restaurantId: 1, isImmutable: 1 }

// finalOrders
{ restaurantId: 1, invoiceNumber: 1 }                    // unique
{ restaurantId: 1, finalizedAt: -1 }

// eventLog
{ restaurantId: 1, timestamp: -1 }
{ orderUid: 1, eventType: 1 }

// menuItems
{ restaurantId: 1, itemCode: 1 }                         // unique
{ restaurantId: 1, isAvailable: 1, category: 1 }

// users
{ email: 1 }                                             // unique

// invoiceSequence
{ restaurantId: 1 }                                      // unique
```

---

*Schema version: 1.0 · Phase 1 baseline · Phases 2 & 3 collections additive only*
