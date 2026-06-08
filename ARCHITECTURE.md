# Architecture — IK Designs

This document describes the system architecture of IK Designs at a level a senior engineer could follow without reading the source: how requests flow through the system, how data moves between layers, how the admin area is secured, and how the application is deployed.

## 1. System Overview

IK Designs is a single Next.js 14 (App Router) application that serves three audiences from one codebase: the public marketing/portfolio site, the client-facing order builder and tracking experience, and an internal admin dashboard. There is no separate backend service — API route handlers under `app/api/**` act as the application's backend, talking to PostgreSQL through Prisma.

```mermaid
graph TD
    subgraph "Public Internet"
        Visitor["Prospective Client"]
        Staff["Studio Staff (Admin)"]
    end

    subgraph "Next.js 14 Application"
        MW["Middleware<br/>JWT check on /admin/*"]
        Pub["Public Routes<br/>/, /portfolio, /order, /track, /consult, /how-it-works"]
        Admin["Admin Routes<br/>/admin/**"]
        API["API Route Handlers<br/>/api/**"]
    end

    subgraph "Data & Storage"
        Prisma["Prisma Client"]
        DB[("PostgreSQL")]
        FS["Local Filesystem<br/>public/uploads/orders/{id}/"]
    end

    Visitor -->|HTTPS| Pub
    Visitor -->|HTTPS| API
    Staff -->|HTTPS| MW
    MW -->|valid JWT| Admin
    MW -->|no/invalid JWT| LoginRedirect["Redirect → /admin/login"]
    Admin --> API
    Pub --> API
    API --> Prisma
    Prisma --> DB
    API -->|photo uploads| FS
    Pub -->|served as static assets| FS
```

## 2. Request Flow — Public Order Submission

This trace shows how a client's six-step order becomes a trackable database record.

```mermaid
sequenceDiagram
    participant U as Client (Browser)
    participant Z as Zustand Store (client-side)
    participant API as POST /api/orders
    participant P as Prisma
    participant DB as PostgreSQL

    U->>Z: Fill steps 1–6 (room, furniture, customisation, mood, contact, review)
    Z-->>U: Persist state across steps & page reloads
    U->>API: Submit (step 6) → JSON payload
    API->>API: Generate reference number (IKD-{year}-{random4})
    API->>P: order.create({ ...serialized fields })
    P->>DB: INSERT INTO "Order" (...)
    DB-->>P: Row created
    P-->>API: Order record
    API-->>U: 201 { referenceNumber }
    U->>U: Show success screen with reference number
```

Arrays and nested objects (`furniture`, `furnitureCustomizations`, `colors`, `styles`) are JSON-serialised by the API handler before insertion into `TEXT` columns, and parsed back out on every read path (`GET /api/orders/{ref}`, `GET /api/admin/orders`, etc.).

## 3. Request Flow — Order Tracking (No Account Required)

```mermaid
sequenceDiagram
    participant U as Client (Browser)
    participant API as GET /api/orders/{ref}
    participant P as Prisma
    participant DB as PostgreSQL

    U->>API: GET /api/orders/IKD-2026-1234
    API->>P: order.findUnique({ referenceNumber, include: notes + photos })
    P->>DB: SELECT ... WHERE referenceNumber = $1 (unique index)
    DB-->>P: Order + OrderNote[] + OrderPhoto[]
    P-->>API: Hydrated order
    API->>API: Parse JSON-text fields (furniture, colors, styles, ...)
    API-->>U: Order detail: status, stage timeline, notes, photos
    U->>U: Render 8-stage progress timeline + gallery
```

Because the lookup key is a unique, generated reference number rather than a user session, this entire flow requires no authentication — by design, to keep tracking frictionless for clients.

## 4. Admin Authentication & Route Protection

Two layers protect the admin area: edge middleware (coarse-grained, runs before any page renders) and per-handler checks (fine-grained, runs inside each API route).

```mermaid
sequenceDiagram
    participant S as Staff (Browser)
    participant MW as Middleware (edge)
    participant Login as POST /api/admin/login
    participant Auth as lib/auth.ts (jose)
    participant Handler as Protected API Handler

    S->>Login: POST { password }
    Login->>Login: verifyPassword(password, storedHash) — scrypt + timingSafeEqual
    Login->>Auth: signAdminToken() → JWT (HS256, 12h expiry)
    Auth-->>Login: signed token
    Login-->>S: Set-Cookie: ik_admin_token (httpOnly)

    S->>MW: GET /admin/orders
    MW->>MW: Read ik_admin_token cookie
    MW->>Auth: jwtVerify(token, SECRET)
    alt valid token
        Auth-->>MW: ok
        MW-->>S: Render admin page
        S->>Handler: fetch /api/admin/orders
        Handler->>Auth: isAdminAuthenticated()
        Auth-->>Handler: true
        Handler-->>S: 200 + data
    else missing/invalid token
        Auth-->>MW: error
        MW-->>S: 302 → /admin/login
    end
```

The password itself is never compared in plaintext: `lib/password.ts` hashes with `scrypt` and a random salt, then compares using `timingSafeEqual` to avoid timing side-channels. The hash is stored via the generic `SystemConfig` key/value table, so rotating the credential doesn't require a schema change.

## 5. Data Flow — Analytics Aggregation

The admin analytics dashboard is computed server-side on each request rather than maintained as a materialised view or precomputed table — appropriate at the studio's current order volume.

```mermaid
flowchart LR
    A["GET /api/admin/analytics"] --> B["Fetch orders<br/>(select: createdAt, budget, furniture, room, status)"]
    B --> C["Group into last 6 months<br/>→ revenue + count per month"]
    B --> D["Tally furniture items<br/>→ top 8 by frequency"]
    B --> E["Tally rooms<br/>→ top 6 by frequency"]
    B --> F["Tally status values<br/>→ distribution map"]
    C & D & E & F --> G["JSON response"]
    G --> H["Custom React bar-chart components<br/>(no charting library)"]
```

## 6. Component & Module Relationships

```mermaid
graph TD
    subgraph "Public Pages (app/*)"
        Home["/ (Hero, FeaturedWorks, BrandStory, Testimonials)"]
        Order["/order → OrderBuilderClient"]
        Track["/track → TrackClient"]
        Consult["/consult → ConsultClient"]
        Portfolio["/portfolio → PortfolioClient"]
    end

    subgraph "Order Builder (components/order-builder/*)"
        S1["Step1Room"]
        S2["Step2Furniture"]
        S3["Step3Customise"]
        S4["Step4Mood"]
        S5["Step5Details"]
        S6["Step6Review"]
        Success["SuccessScreen"]
    end

    subgraph "Shared State & Utilities"
        Store["useOrderStore (Zustand)"]
        Utils["lib/utils.ts (cn, formatCurrency, WHATSAPP_URL)"]
        Data["data/* (catalog seeds, localities, FAQ, testimonials)"]
    end

    subgraph "Admin (app/admin/*)"
        AdminOrders["Orders List & Detail"]
        AdminConsults["Consultations"]
        AdminAnalytics["Analytics"]
        AdminSettings["Catalog Settings (wood/finish/colour/security)"]
        Quote["Printable Quote View"]
    end

    Order --> S1 & S2 & S3 & S4 & S5 & S6 & Success
    S1 & S2 & S3 & S4 & S5 & S6 --> Store
    Order --> Store
    S1 & S2 & S3 --> Data
    Home & Track & Consult & Portfolio --> Utils
    AdminOrders --> Quote
    AdminConsults -->|convert| AdminOrders
```

## 7. Deployment & Runtime

```mermaid
graph LR
    Dev["Local Dev<br/>next dev + ngrok tunnel"] -->|git push| Repo[("Git Repository")]
    Repo --> Build["Next.js Build<br/>(SSR + API routes + static assets)"]
    Build --> Runtime["Production Runtime<br/>ikdesigns.in"]
    Runtime --> PG[("Managed PostgreSQL")]
    Runtime --> Disk["Server Filesystem<br/>public/uploads/"]
```

Database schema changes are managed through Prisma Migrate (`prisma/migrations/`), and the catalog tables are seeded via a dedicated `prisma/seed.ts` script — meaning the entire product catalog (wood types, finishes, colour palette) can be reproduced from source rather than hand-entered per environment.

## Summary

The system deliberately avoids premature distribution: one framework, one database, one deployment unit. The architectural complexity that *does* exist — JWT verification at the edge, JSON-text serialisation for flexible order data, server-computed analytics, multipart photo uploads to local disk — is concentrated where the domain genuinely needs it, rather than spread across a microservice topology that this product's scale doesn't yet justify.
