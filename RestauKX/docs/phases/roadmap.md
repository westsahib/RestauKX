# Development Roadmap — RestauKX

Three phases. Each one ships complete, usable value. No phase breaks what came before.

---

## Phase 1 — The Billing Engine
**Goal:** A fully operational billing system. Walk in, take an order, generate a clean invoice, close the day.

### Collections Introduced
`storeSettings` · `users` · `menuItems` · `orders` · `orderLines` · `invoiceSequence` · `finalOrders` · `eventLog`

### Deliverables

**Infrastructure**
- [ ] MERN project scaffolding — monorepo, ESLint, Prettier, folder conventions
- [ ] MongoDB Atlas cluster — replica set enabled (required for multi-doc transactions)
- [ ] Mongoose models for all Phase 1 collections
- [ ] JWT auth with httpOnly refresh token cookies
- [ ] RBAC middleware — owner / admin / waiter roles enforced at route level
- [ ] Startup health checks — invoice sequence integrity, stuck-order reconciliation, stale file cleanup

**Core Order Flow**
- [ ] New order creation — mode selection (DINE_IN / PARCEL / TAKEAWAY)
- [ ] Order line add / edit / remove — price snapshot written at line creation
- [ ] Order total recalculation on every line mutation
- [ ] Slide-to-finalize UI control with configurable undo window
- [ ] Atomic finalization — `invoiceSequence` $inc + `finalOrder` write in single transaction
- [ ] Order void flow with admin password confirmation and EventLog entry

**Invoice & Export**
- [ ] Automated PDF generation via PDFKit on finalization
- [ ] Invoice history listing with filters (date, payment method, mode)
- [ ] Individual invoice PDF download
- [ ] Streaming CSV/Excel export by date range (no full-dataset memory load)

**Menu Management**
- [ ] Menu item CRUD with half/full price and GST per item
- [ ] Availability toggle (fast waiter-facing action)
- [ ] Bulk import via CSV/Excel — preview, policy selection, transactional write
- [ ] Admin password gate for REPLACE_ALL import policy

**React Web App**
- [ ] Login · Register screens
- [ ] Admin dashboard — daily revenue, active orders, recent invoices
- [ ] New order screen — mode/table select → menu browse → line management → finalize
- [ ] Order history with live status indicators
- [ ] Invoice detail + PDF preview + download
- [ ] Menu management screen
- [ ] Store settings screen

**Estimated Duration:** 6–8 weeks

---

## Phase 2 — Live Operations
**Goal:** The kitchen and the counter work as one. Customers order from the table. Everyone sees the same truth in real time.

### Collections Introduced
`tables` · `printJobs`

### Fields Activated
`orders.kitchenStatus` — defined in Phase 1 schema, activated in Phase 2 flows

### Deliverables

**Table Management**
- [ ] Table layout admin — create, edit, assign floor/capacity
- [ ] QR code generation per table (PNG + shareable URL)
- [ ] Table status management — AVAILABLE / OCCUPIED / RESERVED
- [ ] Table map view (visual grid with live status colours)

**Customer QR Ordering**
- [ ] Unauthenticated QR order submission endpoint (table token auth only)
- [ ] Customer-facing order page — no login, no app install required
- [ ] Order confirmation screen post-submission

**Real-Time Layer (Socket.IO)**
- [ ] `/kitchen` namespace — new order events, status update broadcasts
- [ ] `/waiter` namespace — kitchen status changes, order-ready alerts
- [ ] Order lifecycle events wired to Socket.IO throughout
- [ ] Reconnection and missed-event recovery on client reconnect

**Kitchen Display Screen (KDS)**
- [ ] Web-based KDS — full-screen, role-gated kitchen view
- [ ] Live incoming order queue with item-level detail
- [ ] One-tap kitchen status advancement (CONFIRMED → PREPARING → READY)
- [ ] Audio alert on new order arrival

**React Native Android App**
- [ ] Waiter app — table map, open orders, finalize flow
- [ ] Kitchen app — KDS view, status updates
- [ ] Push notifications via Firebase Cloud Messaging (FCM)
- [ ] Offline queue — actions taken offline sync on reconnect

**Print Jobs**
- [ ] KOT (Kitchen Order Ticket) print job enqueue on order creation
- [ ] Invoice print job enqueue on finalization
- [ ] Retry logic for failed print jobs — max 3 attempts with backoff

**Estimated Duration:** 6–8 weeks

---

## Phase 3 — The Control Room
**Goal:** Management has full visibility. Staff are coordinated. The platform runs itself.

### Collections Introduced
`staff` · `shifts` · `messages` · `notifications`

### Deliverables

**Staff & Shift Management**
- [ ] Staff profiles — designation, contact, role assignment
- [ ] Clock-in / clock-out with shift duration tracking
- [ ] Shift history per staff member
- [ ] Activity log — every finalization, void, and import attributed to a user

**In-App Messaging**
- [ ] Role-broadcast messages (e.g., message all kitchen staff)
- [ ] Direct staff-to-staff messaging
- [ ] Read receipts and unread count badges
- [ ] Socket.IO-powered real-time delivery

**Notifications**
- [ ] In-app notification centre
- [ ] Push notifications for: new order (kitchen), order ready (waiter), low stock flag
- [ ] Notification preferences per role

**Analytics Dashboard**
- [ ] Revenue trends — daily / weekly / monthly aggregation
- [ ] Top-selling items by quantity and revenue
- [ ] Peak hours heatmap — order count by hour of day
- [ ] Order type breakdown — DINE_IN vs PARCEL vs TAKEAWAY
- [ ] Export analytics data as CSV

**Advanced Settings**
- [ ] Receipt branding — logo, footer text, GST display toggle
- [ ] Backup configuration — scheduled encrypted export to cloud storage
- [ ] Backup restore flow — decrypt → validate checksum → validate schema → apply
- [ ] Data retention settings — EventLog archive threshold
- [ ] Multi-device LAN mode configuration (future: multi-till support)

**Adapter Implementations**
- [ ] `BackupAdapter` — encrypted snapshot to cloud (S3-compatible)
- [ ] `PrinterAdapter` — thermal printer integration via printJobs queue
- [ ] `AnalyticsAdapter` — EventLog replay to external BI webhook

**Estimated Duration:** 8–10 weeks

---

## Milestones

| Milestone | Phase | Status |
|---|---|---|
| Repository & architecture complete | Pre-impl | ✅ Done |
| All Phase 1 collections & schemas defined | Pre-impl | ✅ Done |
| API specification complete | Pre-impl | ✅ Done |
| Phase 1 MVP — billing engine live | 1 | TBD |
| Phase 2 — real-time ops + Android app | 2 | TBD |
| Phase 3 — full platform launch | 3 | TBD |

---

## Architectural Guarantee Across All Phases

Later phases are strictly additive. Phase 2 introduces new collections and activates dormant fields — it never modifies Phase 1 schemas. Phase 3 extends Phase 2 the same way. A restaurant running Phase 1 billing can upgrade to Phase 2 with zero data migration.

---

*Roadmap v1.0 · RestauKX · Pre-Implementation · Dates TBD at kickoff*
