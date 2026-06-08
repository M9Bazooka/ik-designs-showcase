# Technical Decisions — IK Designs

Six decisions that shaped how this product is built, framed as problem → options → choice → reasoning → outcome.

---

## 1. How to store flexible, per-item order configuration

**The Problem**
Each order can include multiple furniture items, and *each item* carries its own shape, primary/secondary material, finish, sheen, dimensions, and colours. The "shape" of that data isn't fixed — a sofa's options (chesterfield, mid-century, loveseat...) look nothing like a wardrobe's (hinged, sliding, walk-in...). A rigid relational schema for this would mean either a wide, mostly-empty `Order` table or a deep web of join tables for data that's never queried in isolation.

**Options Considered**
- Fully normalised schema: separate tables for `OrderItem`, `ItemMaterial`, `ItemFinish`, etc.
- Native PostgreSQL `JSON`/`JSONB` columns via Prisma's `Json` type
- JSON-encoded strings stored in plain `TEXT` columns, parsed at the application boundary

**What I Chose**
JSON-encoded strings in `TEXT` columns (`furniture`, `furnitureDetails`, `colors`, `styles`), serialised on write and parsed on every read path.

**Why**
The nested configuration is always read and written as a whole — the UI needs "this order's full furniture list with each item's customisation," never "all orders containing a chesterfield sofa in walnut." A normalised schema would have added migration and query overhead for a query pattern that doesn't exist. Plain `TEXT` over `JSONB` was a deliberate simplicity call: it keeps the schema portable and avoids reasoning about Prisma's `Json` type quirks, at the cost of losing in-database query/index capability on nested fields — a capability this product doesn't currently need.

**Outcome**
The `Order` model stays flat and readable, migrations stay simple, and the API layer owns a small, consistent serialise/parse boundary (visible in `app/api/orders/route.ts` and `app/api/orders/[ref]/route.ts`). If query-by-nested-field ever becomes a real requirement, the migration path to `JSONB` (or selective normalisation) is straightforward because the boundary is already centralised.

---

## 2. How clients access their order status

**The Problem**
Clients need to check on their custom order's progress. The conventional answer is "create an account, log in, see your dashboard" — but that adds signup friction for a one-time (or infrequent) furniture purchase, and most of this studio's clients are used to tracking things like courier deliveries by reference number, not portals.

**Options Considered**
- Full user accounts with authentication (email/password or OTP)
- Magic-link emails per order
- Public lookup by a generated, hard-to-guess reference number

**What I Chose**
A generated reference number (`IKD-YYYY-NNNN` for orders, `IKC-YYYY-NNNN` for consultations) that doubles as the public lookup key — no account, no login, just `GET /api/orders/{ref}`.

**Why**
This mirrors a pattern clients already understand (tracking a parcel), removes every bit of signup friction, and matches the actual usage pattern — a client checks their order status a handful of times over a multi-week build, not daily. The trade-off is explicit: anyone with the reference number can view that order's status. That's an acceptable exposure for "is my sofa ready yet," and avoiding an account system also means there's no password-reset flow, no email-verification flow, and no user table to secure for the public side of the product.

**Outcome**
The tracking experience (`/track`) is a single search box. There's no auth surface to maintain on the client side at all — the entire public attack surface for "view an order" is "guess a reference number," which is intentionally a much smaller problem than "compromise an account system."

---

## 3. How to secure the admin area for a single administrator

**The Problem**
The studio has effectively one admin user. A full multi-user auth system (sign-up, roles, permissions, password resets, sessions table) would be substantial infrastructure for a system that, at launch, needs to authenticate exactly one person.

**Options Considered**
- Database-backed sessions with a `User`/`Session` model
- Third-party auth provider (e.g., NextAuth/Auth.js, Clerk)
- Stateless signed JWT in an httpOnly cookie, verified in middleware

**What I Chose**
A password-only login that issues a signed JWT (HS256, 12-hour expiry, via `jose`) stored in an httpOnly `ik_admin_token` cookie, verified both in Next.js edge middleware (`middleware.ts`, matcher `/admin/:path*`) and again in each protected handler via `isAdminAuthenticated()`.

**Why**
For one administrator, a session table and user-management UI would be pure overhead — infrastructure to secure with no corresponding benefit. A signed JWT is stateless (no DB round-trip to validate a session), the middleware layer rejects unauthenticated requests at the edge before any page or handler runs, and the 12-hour expiry bounds the blast radius of a leaked token without forcing daily re-logins. The password itself is hashed with `scrypt` and compared via `timingSafeEqual` (`lib/password.ts`) — so even the one credential that exists is handled correctly.

**Outcome**
The entire admin auth surface is ~70 lines across `lib/auth.ts`, `lib/password.ts`, and `middleware.ts`. It's small enough to read end-to-end in five minutes, which — for a security-relevant subsystem — is itself a feature. If the studio ever needs multiple admin accounts with different permissions, the `SystemConfig` table and the JWT payload (`{ role: "admin" }`) already anticipate that extension without a redesign.

---

## 4. Where to store client-uploaded order photos

**The Problem**
Staff upload progress photos against orders (e.g., "your wardrobe frame is assembled") that clients then see on the tracking page. Those files need to be stored somewhere durable and servable over HTTP.

**Options Considered**
- Cloud object storage (S3, Cloudinary, UploadThing, etc.)
- Database BLOBs
- The application server's local filesystem, served as static assets

**What I Chose**
Local filesystem storage under `public/uploads/orders/{orderId}/`, written via Node's `fs/promises` in the upload route handler and served by Next.js as static files.

**Why**
At launch scale (a single studio's order volume), an external storage dependency adds an account to manage, an SDK to integrate, credentials to rotate, and — for a low-volume use case — a cost line item with no real benefit yet. Writing straight to disk and serving from `public/` is the simplest correct solution: zero extra services, zero extra latency, zero extra failure modes to handle. Filenames are sanitised (`replace(/[^a-zA-Z0-9._-]/g, "_")`) and timestamp-prefixed to avoid collisions and path-traversal issues.

**Outcome**
Photo upload and display work end-to-end with no third-party dependency. The known trade-off — storage is now coupled to a single server instance, which complicates horizontal scaling and zero-downtime redeploys — is written down here deliberately: it's the first thing I'd change if/when the studio's volume justifies the migration to object storage, and the upload route is small and isolated enough that the migration is a contained change.

---

## 5. How to build the admin analytics dashboard

**The Problem**
The studio needed visibility into business performance — revenue trends, what furniture and rooms are most in demand, where orders sit in the pipeline — without standing up a separate analytics/BI tool.

**Options Considered**
- A charting library (Recharts, Chart.js, visx)
- A hosted BI/embedded-analytics product
- Small custom React components computing layout from aggregated data

**What I Chose**
Server-side aggregation in `GET /api/admin/analytics` (grouping orders into the last six months, tallying furniture/room/status frequencies with `Promise.all`-free, single-pass reduces) feeding small, purpose-built bar-chart components on the client — no charting dependency.

**Why**
The visualisations needed here are simple (bar charts, counts, currency totals) and needed to *look like part of the product* — matching the warm walnut/cream design system exactly, not a generic dashboard aesthetic bolted on top. A charting library would have meant fighting its theming API to match a bespoke design system, plus shipping its bundle weight for four bar charts. Computing the aggregation server-side also means the client never downloads raw order rows just to summarise them.

**Outcome**
The analytics page renders fast, looks native to the brand, and the entire chart-rendering layer is small enough to extend (e.g., adding a new metric is "add a field to the aggregation response and a new `<BarChart>` call," not "learn a new library's API").

---

## 6. How to model the consultation → order relationship

**The Problem**
Not every consultation becomes an order — someone might book a showroom visit and decide not to commission anything. But when a consultation *does* convert, re-typing the client's name, phone, email, and locality into a fresh order form is wasted staff time and a source of data-entry errors.

**Options Considered**
- A single `Order` model with a "lead/consultation" status, promoted in place
- Two separate models with no formal link between them
- Two separate models plus an explicit conversion action that creates a new linked record

**What I Chose**
`Consultation` and `Order` are distinct Prisma models. A dedicated endpoint, `POST /api/admin/consultations/{id}/convert`, looks up the consultation, creates a new `Order` pre-filled with the consultation's `name`, `phone`, `email`, and `locality`, generates a fresh `IKD-` reference number, and marks the original consultation `completed`.

**Why**
Collapsing these into one model with a status field would mean the `Order` table accumulates rows that were never actually orders — polluting every order-focused query (analytics, search, status pipelines) with records that don't belong there. Keeping them separate but linking them through an explicit, auditable action reflects the real business process accurately: a consultation is a *lead*; an order is a *commission*. The conversion endpoint is where that business event is recorded.

**Outcome**
Order analytics and search only ever see real orders. Staff get a one-click path from "client visited the showroom and wants to proceed" to "order created with their details already filled in," and the `completed` status on the original consultation preserves the full history of how that order came to be — useful for understanding which lead sources actually convert.
