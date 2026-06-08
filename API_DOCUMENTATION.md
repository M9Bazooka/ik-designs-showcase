# API Documentation — IK Designs

Clean reference for the application's REST-style API, implemented as Next.js App Router route handlers under `app/api/**`. All endpoints exchange JSON unless noted otherwise (photo upload uses `multipart/form-data`).

> Admin endpoints require an authenticated session (signed JWT in an httpOnly cookie). They return `401 Unauthorized` if the caller isn't authenticated — implementation details of that check are intentionally omitted here; see [ARCHITECTURE.md](ARCHITECTURE.md) for the auth flow.

---

## Public — Orders

### `POST /api/orders`
Submit a new custom furniture order from the order builder.

**Request body** (subset of fields)
| Field | Type | Notes |
|---|---|---|
| `room` | string | e.g. `"Bedroom"`, `"Living Room"` |
| `furniture` | string[] | Selected furniture items |
| `furnitureCustomizations` | object | Per-item shape/material/finish/dimensions/colour |
| `woodType`, `finish`, `finishSheen` | string | Catalog selections |
| `dimensions` | `{ width, height, depth }` | Millimetres |
| `colors`, `styles` | string[] | Aesthetic preferences |
| `notes`, `budget`, `timeline` | mixed | Mood-board fields |
| `name`, `phone`, `email`, `locality`, `contactMethod`, `howHeard` | string | Contact details |

**Response** `201 Created`
```json
{ "referenceNumber": "IKD-2026-4821" }
```

---

### `GET /api/orders/{ref}`
Public order lookup by reference number — powers the `/track` page. No authentication required by design (see [TECHNICAL_DECISIONS.md](TECHNICAL_DECISIONS.md#2-how-clients-access-their-order-status)).

**Response** `200 OK`
```json
{
  "referenceNumber": "IKD-2026-4821",
  "status": "in_production",
  "room": "Bedroom",
  "furniture": ["Bed", "Wardrobe"],
  "woodType": "teak",
  "finish": "natural_teak",
  "budget": 250000,
  "timeline": "2-3months",
  "notesList": [{ "text": "Frame assembly complete", "createdAt": "..." }],
  "photos": [{ "url": "/uploads/orders/.../progress.jpg", "caption": "Frame stage" }]
}
```
**Response** `404 Not Found` — `{ "error": "Order not found" }`

---

## Public — Consultations

### `POST /api/consult`
Book a design consultation (home visit, showroom visit, or video call).

**Request body**
| Field | Type | Notes |
|---|---|---|
| `name`, `phone`, `email` | string | Contact details |
| `visitType` | string | `"home"` \| `"showroom"` \| `"video"` |
| `date`, `timeSlot` | string | Selected slot |
| `locality` | string | Hyderabad locality |
| `notes` | string | Optional message |

**Response** `201 Created`
```json
{ "referenceNumber": "IKC-2026-3309" }
```

---

## Public — Catalog

### `GET /api/settings`
Returns the active product catalog that drives the order builder's selection screens.

**Response** `200 OK`
```json
{
  "woodTypes":   [{ "id": "teak", "label": "Teak", "color": "#8B5E3C", "sortOrder": 0 }],
  "finishTypes": [{ "id": "walnut", "label": "Walnut", "swatch": "...", "sheens": ["Matte", "Glossy"] }],
  "colorPalette":[{ "id": "c1", "hex": "#F5ECD7", "name": "Warm Cream" }]
}
```
Fetched in parallel server-side (`Promise.all`) and returned with `sheens` already deserialised from its stored JSON-string form.

---

## Admin — Authentication

### `POST /api/admin/login`
Password-only login (no username field).

**Request body:** `{ "password": "..." }`
**Response** `200 OK` — sets `ik_admin_token` httpOnly cookie (12-hour expiry)
**Response** `401 Unauthorized` — `{ "error": "Invalid password" }`

### `POST /api/admin/logout`
Clears the admin session cookie. **Response** `200 OK`

### `PUT /api/admin/settings/password`
Change the admin password. Requires the current password; new password is hashed with `scrypt` and stored via the `SystemConfig` table.

**Request body:** `{ "currentPassword": "...", "newPassword": "..." }`

---

## Admin — Orders

### `GET /api/admin/orders`
List and filter orders for the operations dashboard.

**Query params**
| Param | Type | Notes |
|---|---|---|
| `status` | string | Filter by pipeline stage |
| `search` | string | Matches name, reference number, or phone |
| `from`, `to` | date string | Filter by creation date range |

Returns orders ordered newest-first, each with its single most recent note attached.

### `GET /api/admin/orders/{id}`
Full detail for a single order (used by the order detail view and the printable quote page).

### `PATCH /api/admin/orders/{id}`
Update an order's status or fields as it moves through the pipeline (`received` → `design_review` → `materials_sourced` → `in_production` → `quality_check` → `ready` → `delivered` → `closed`).

### `POST /api/admin/orders/{id}/notes`
Append an internal note to an order's timeline (visible to staff and, depending on context, surfaced to the client on `/track`).

**Request body:** `{ "text": "..." }`

### `GET /api/admin/orders/{id}/photos`
List progress photos attached to an order.

### `POST /api/admin/orders/{id}/photos`
Upload one or more progress photos. **Content-Type:** `multipart/form-data`

**Form fields**
| Field | Type | Notes |
|---|---|---|
| `photos` | file[] | One or more image files |
| `caption` | string | Optional caption applied to the batch |

Files are sanitised, timestamp-prefixed, written to `public/uploads/orders/{id}/`, and recorded in `OrderPhoto`.

### `DELETE /api/admin/orders/{id}/photos/{photoId}`
Remove a progress photo record.

---

## Admin — Consultations

### `GET /api/admin/consultations`
List booked consultations for staff review and scheduling.

### `PATCH /api/admin/consultations/{id}`
Update a consultation's status or details (e.g., reschedule, mark as no-show).

### `POST /api/admin/consultations/{id}/convert`
Convert a consultation into a full order — creates a new `Order` pre-filled with the consultation's contact details, generates a fresh reference number, and marks the consultation `completed`.

**Request body:** `{ "room": "...", "furniture": ["..."], "budget": 250000 }`
**Response** `201 Created`
```json
{ "orderId": "clx...", "referenceNumber": "IKD-2026-7790" }
```

---

## Admin — Analytics

### `GET /api/admin/analytics`
Aggregated business metrics computed server-side from order records.

**Response** `200 OK`
```json
{
  "totalOrders": 142,
  "totalRevenue": 28500000,
  "totalConsults": 96,
  "months": [{ "label": "Jan 26", "revenue": 4200000, "count": 18 }],
  "topFurniture": [{ "name": "Wardrobe", "count": 41 }],
  "topRooms": [{ "name": "Bedroom", "count": 58 }],
  "statusCount": { "in_production": 22, "delivered": 64 }
}
```

---

## Admin — Catalog Management

Each catalog resource (`wood-types`, `finishes`, `colors`) follows the same CRUD shape:

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/admin/settings/{resource}` | List all entries, ordered by `sortOrder` |
| `POST` | `/api/admin/settings/{resource}` | Create a new catalog entry |
| `PATCH` | `/api/admin/settings/{resource}/{id}` | Update an entry (label, swatch, description, sort order, …) |
| `DELETE` | `/api/admin/settings/{resource}/{id}` | Remove an entry |

These power the selection screens in the public order builder (`/order`) — staff can add a new wood finish or retire a discontinued colour without a code deploy.
