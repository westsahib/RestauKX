# RestauKX

**Restaurant Billing, Re-engineered.**  
A production-grade MERN SaaS platform — designed before a single line of code was written.

---

## What It Solves

Small and medium restaurants run on chaos — handwritten bills, manual tallies, no visibility between the kitchen and the counter. RestauKX replaces that entirely: one platform, every role, zero paper.

---

## What Was Built (Pre-Implementation)

Full system design completed across three engineering phases before implementation began.

**Phase 1 — The Billing Engine**  
Atomic invoice finalization. Every order line captures a price snapshot at the moment of entry — menu price changes never corrupt historical records. Invoice numbers are assigned via a dedicated atomic counter inside a database transaction, making duplicates architecturally impossible. An append-only event log records every user action from draft creation to final commit.

**Phase 2 — Live Operations**  
Customers order from their table by scanning a QR code. The kitchen sees it in real time via WebSocket. The waiter sees the status update the moment the kitchen confirms. No shouting across the room.

**Phase 3 — The Control Room**  
Role-based dashboards for every stakeholder. Staff shift management. In-app messaging between kitchen, counter, and management. Revenue analytics with peak-hour heatmaps and item performance tracking.

---

## Architecture Decisions Worth Noting

`billingStatus` and `kitchenStatus` are independent state machines on the same order document. One tracks the invoice lifecycle (DRAFT → FINALIZED), the other tracks kitchen operations (PENDING → SERVED). They never interfere — this is intentional.

Finalized invoices live in a separate `finalOrders` collection and are never updated. Corrections are handled via a VOIDED event in the event log and a fresh invoice — not by mutating the past.

A startup health check runs on every server boot: it detects orders stuck mid-finalization, validates invoice sequence integrity, and reconciles any drift — automatically, before the first request is served.

---

## Stack

MongoDB · Express.js · React · Node.js · Socket.IO · React Native (Android)  
JWT · PDFKit · React Query · Tailwind CSS · Mongoose · node-cron

---

## By The Numbers

| | |
|---|---|
| MongoDB collections designed | 15 across 3 phases |
| API endpoints specified | 40+ |
| Event types catalogued | 20+ |
| Android screens mockuped | 4 |
| Lines of documentation | 1,200+ |
| Lines of production code | 0 — and that's the point |

---

## Documentation Index

| Document | What It Contains |
|---|---|
| [`system-architecture.md`](./docs/architecture/system-architecture.md) | 4-layer architecture, all key flows, phase-by-phase construction plan |
| [`database-schema.md`](./docs/architecture/database-schema.md) | All 15 MongoDB collections with field-level annotations and index strategy |
| [`api-spec.md`](./docs/api/api-spec.md) | Full REST API across all 3 phases |
| [`roadmap.md`](./docs/phases/roadmap.md) | Phase-gated delivery checklist |
| [`TECH-STACK.md`](./docs/TECH-STACK.md) | Every technology decision with its rationale |

---

> Designed with the belief that the hardest problems in software are not implementation problems — they are clarity problems.  
> RestauKX was designed to be clear first.

---

*Pre-Implementation · MERN SaaS · Web + Android · 2026*
