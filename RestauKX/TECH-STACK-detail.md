# Tech Stack — RestauKX

Every technology choice here was made deliberately. This document records not just what was chosen, but why — and what was rejected.

---

## The Core Stack

**MERN — MongoDB · Express.js · React · Node.js**

One language end-to-end. The same developer who writes the Mongoose schema writes the React component that renders the invoice. No context-switching tax. No translation layer between teams.

---

## Frontend — Web

| Technology | Version | Role |
|---|---|---|
| React | 18.x | UI component tree — concurrent rendering for snappy interactions |
| React Router | v6 | Client-side routing with protected route wrappers |
| Tailwind CSS | 3.x | Utility-first styling — no custom CSS sprawl |
| React Query | v5 | Server state management — cache, background refetch, optimistic updates |
| Axios | latest | HTTP client with interceptors for JWT refresh |
| React-PDF | latest | In-browser invoice PDF rendering |
| Socket.IO Client | 4.x | Real-time order events (Phase 2+) |

**Why React Query over Redux?**
RestauKX is server-state-heavy — orders, invoices, menu items. Redux is designed for client state. React Query handles server state natively: caching, invalidation, background sync, optimistic updates. Redux would be engineering overhead with no benefit here.

---

## Frontend — Mobile (Android)

| Technology | Role |
|---|---|
| React Native | Android app — shares hooks, services, and constants with the web codebase |
| Expo | Managed build workflow — faster iteration, OTA updates, FCM integration |
| React Native Paper | Material Design component library |
| Firebase Cloud Messaging | Push notifications for kitchen and waiter alerts (Phase 2+) |

**Why React Native over Flutter?**
The web app is React. Shared business logic — API service layer, order status constants, invoice formatters — is plain JavaScript that runs in both environments. Flutter would require rewriting all of that in Dart. React Native keeps the codebase unified.

---

## Backend

| Technology | Version | Role |
|---|---|---|
| Node.js | 20 LTS | Runtime — non-blocking I/O handles concurrent order mutations efficiently |
| Express.js | 4.x | REST API framework — minimal, well-understood, composable middleware |
| Socket.IO | 4.x | WebSocket abstraction with fallback — real-time KDS and waiter events (Phase 2+) |
| PDFKit | latest | Server-side invoice PDF generation — no browser dependency |
| Multer | latest | Multipart file upload handling for menu CSV/Excel import |
| node-cron | latest | Background job scheduler — export jobs, cleanup, backup triggers (Phase 3+) |
| express-validator | latest | Input sanitization and validation at route level |
| Helmet.js | latest | Sets secure HTTP headers (CSP, HSTS, X-Frame-Options) |
| express-rate-limit | latest | 100 req/15min on public routes; 500/15min on authenticated |

---

## Database

| Technology | Role |
|---|---|
| MongoDB Atlas | Cloud-hosted MongoDB — replica set required for multi-document transactions |
| Mongoose | ODM — schema validation, pre/post hooks, index definitions |

**Why MongoDB over PostgreSQL?**
The menu item schema evolves constantly — half/full pricing, custom metadata, freeform tags. A relational schema would require migrations for every menu structure change. MongoDB's flexible document model absorbs these changes without touching existing records.

More importantly: the `finalOrders` collection embeds a complete line snapshot per invoice. In PostgreSQL this would require a join across three tables to reconstruct a historical invoice. In MongoDB it's a single document read. Invoice retrieval is one of the most frequent operations in the system.

**Replica set is non-negotiable.** The atomic finalization flow (`invoiceSequence.$inc` + `finalOrder` insert + `orderLines` lock) requires multi-document transactions. MongoDB transactions require a replica set. Atlas M10+ provides this out of the box.

---

## Authentication & Security

| Technology | Role |
|---|---|
| JSON Web Tokens (JWT) | Stateless auth — access token (15min) + refresh token (7 days) |
| bcryptjs | Password hashing — cost factor 12 |
| httpOnly cookies | Refresh token storage — not accessible to JavaScript, XSS-resistant |
| CORS | Restricted to known frontend origins — no wildcard in production |

**Why JWTs over sessions?**
RestauKX is a SaaS platform with a web app, an Android app, and a customer-facing QR page — all hitting the same API. Server-side sessions would require a shared session store (Redis) across all deployments. JWTs are stateless and work identically across every client type.

---

## PDF & Export

| Technology | Role |
|---|---|
| PDFKit | Programmatic PDF generation on the server — no headless browser required |
| ExcelJS | Streaming Excel export for sales data — writes rows without buffering the full dataset |
| csv-stringify | Streaming CSV export — same pattern as ExcelJS |

**Why streaming for exports?**
A restaurant running for one year may have 15,000+ finalized invoices. Loading all of them into memory to build an export file would cause OOM errors on constrained servers. Both `ExcelJS` and `csv-stringify` support streaming writes — rows are piped directly to the HTTP response as they are read from the database cursor.

---

## DevOps (Planned)

| Tool | Purpose |
|---|---|
| GitHub Actions | CI pipeline — lint, test, build on every PR |
| Docker | Containerised backend for consistent deployments |
| Vercel | React web app hosting — zero-config, edge CDN |
| Railway | Node.js API server hosting — Dockerized, auto-deploy from main |
| MongoDB Atlas | M10 replica set — multi-document transaction support |
| Cloudflare R2 | PDF storage (S3-compatible) — invoice PDFs and export files |

---

## What Was Considered and Rejected

| Alternative | Rejected For |
|---|---|
| Next.js (over React) | SSR adds complexity with no benefit — RestauKX is an authenticated SaaS dashboard, not a public-facing marketing site |
| PostgreSQL | Join-heavy invoice reconstruction; rigid schema for evolving menu structures |
| GraphQL | Overkill for a REST-shaped domain with well-defined resource boundaries |
| Prisma | MongoDB support is experimental in Prisma — Mongoose is the proven choice |
| Firebase Auth | Vendor lock-in on a core security primitive; JWT gives full control |
| Flutter | Would require full rewrite of shared JS business logic in Dart |
| Redis (Phase 1) | Not needed until multi-instance Socket.IO in Phase 3 — added only when required |

---

*Tech Stack v1.0 · RestauKX · Pre-Implementation*
