# RestauKX

**Restaurant Billing System — MERN SaaS**

> Pre-Implementation · Architecture & Design Complete · Implementation Pending

---

Small restaurants run on paper and memory. Orders get lost. Invoices are handwritten. The kitchen has no idea what the counter just promised a customer. RestauKX replaces all of it — one platform, every role, from order to invoice.

---

## What This Repository Contains

This is a pre-implementation repository. The entire system — architecture, database schema, API contracts, phase roadmap, and Android UI mockups — was designed and documented before a line of production code was written. That is intentional.

```
RestauKX/
├── README.md                               ← you are here
├── SHOWCASE.md                             ← portfolio summary
├── docs/
│   ├── architecture/
│   │   ├── system-architecture.md          ← 4-layer architecture, all key flows
│   │   └── database-schema.md              ← 15 MongoDB collections, phase-gated
│   ├── api/
│   │   └── api-spec.md                     ← 40+ endpoints across 3 phases
│   ├── phases/
│   │   └── roadmap.md                      ← phase-by-phase delivery checklist
│   ├── TECH-STACK.md                       ← every decision + what was rejected
│   └── CONTRIBUTING.md                     ← branching, commits, code style
├── ui-mockups/
│   ├── android/
│   │   └── android-mockups.html            ← 4 Android screens (open in browser)
│   └── web/
└── src/
    ├── client/                             ← React web app (to be scaffolded)
    ├── server/                             ← Express + Node.js (to be scaffolded)
    └── shared/                             ← constants, types, utilities
```

---

## The Platform in Three Phases

### Phase 1 — The Billing Engine
The core. A waiter creates an order, adds items, and slides to confirm. At that moment, a MongoDB transaction atomically increments an invoice sequence counter, writes an immutable `FinalOrder` document, locks all order lines, and generates a PDF — in a single committed operation or not at all. Every price is captured as a snapshot at the time of entry. Menu changes never touch historical records.

### Phase 2 — Live Operations
A customer scans a QR code at their table and submits an order. The kitchen sees it on a display screen within milliseconds via Socket.IO. The waiter's app updates the moment the kitchen marks it ready. No polling. No refresh.

### Phase 3 — The Control Room
Role-based dashboards, in-app staff messaging, shift management, and an analytics engine built on MongoDB aggregation pipelines — revenue trends, peak-hour heatmaps, top-selling items. Plus encrypted backup and restore with schema validation.

---

## Architecture in Brief

Four layers — Presentation, Domain, Persistence, Infrastructure — communicating strictly downward through defined interfaces.

Two independent state machines live on every order document: `billingStatus` tracks the invoice lifecycle (`DRAFT → PENDING_FINALIZE → FINALIZED → VOIDED`). `kitchenStatus` tracks kitchen operations (`PENDING → CONFIRMED → PREPARING → READY → SERVED`). They never interfere.

`FinalOrders` is a separate, immutable collection. Invoices are never updated after they are written. Corrections go through a void-and-reissue flow recorded in the append-only `EventLog`.

Every phase is strictly additive. Phase 2 introduces new collections and activates dormant fields. It never modifies Phase 1 schemas. A restaurant running Phase 1 upgrades to Phase 2 with zero data migration.

---

## Stack

```
Frontend    React 18 · React Query · Tailwind CSS · React Router
Mobile      React Native · Expo · Firebase Cloud Messaging
Backend     Node.js 20 LTS · Express.js · Socket.IO
Database    MongoDB Atlas (replica set) · Mongoose
Auth        JWT · bcryptjs · httpOnly cookies
PDF         PDFKit · ExcelJS (streaming export)
DevOps      GitHub Actions · Docker · Vercel · Railway
```

---

## Documentation

| Document | What It Answers |
|---|---|
| [System Architecture](RestauKX/docs/architecture/system-architecture.md) | How the system is structured, how data flows, how each phase builds on the last |
| [Database Schema](RestauKX/docs/architecture/database-schema.md) | Every collection, every field, every index — annotated |
| [API Specification](RestauKX/docs/api/api-spec.md) | Every endpoint across all 3 phases — request shapes, response shapes, error codes |
| [Roadmap](RestauKX/docs/phases/roadmap.md) | Granular delivery checklist per phase, with architectural guarantees |
| [Tech Stack](RestauKX/docs/TECH-STACK.md) | Every technology choice — and every alternative that was considered and rejected |
| [Contributing](RestauKX/docs/CONTRIBUTING.md) | Branching strategy, commit conventions, code style |

---

## Getting Started *(Post-Implementation)*

```bash
git clone https://github.com/your-org/RestauKX.git
cd RestauKX

# Backend
cd src/server && npm install && npm run dev

# Frontend
cd src/client && npm install && npm run dev
```

Environment variables, seed scripts, and deployment guides will be added here when Phase 1 scaffolding begins.

---

*RestauKX · Pre-Implementation · MERN SaaS · Web + Android · 2026*
