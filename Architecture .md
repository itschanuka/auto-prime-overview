# System Architecture — Auto Prime DMS

> Full-stack dealership management system. Designed for production use — not a prototype.  
> Built solo over 6 months: architecture, database, API, frontend, deployment, documentation.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INTERNET                                     │
└──────────────┬──────────────────────────────┬───────────────────────┘
               │                              │
               ▼                              ▼
  ┌────────────────────────┐      ┌────────────────────────┐
  │   PUBLIC WEBSITE       │      │   ADMIN DASHBOARD      │
  │   Next.js (Vercel)     │      │   Next.js (Vercel)     │
  │                        │      │                        │
  │  • Vehicle listing     │      │  • Protected routes    │
  │  • Vehicle detail      │      │  • JWT middleware       │
  │  • Contact form        │      │  • Role-based UI       │
  │  • About / Contact     │      │  • 13 modules          │
  └──────────┬─────────────┘      └──────────┬─────────────┘
             │                               │
             │  REST API calls               │  REST API calls
             │  (anon key routes only)       │  (Bearer JWT required)
             │                               │
             └───────────────┬───────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │      EXPRESS API             │
              │      Node.js (Render)        │
              │                              │
              │  • Auth middleware           │
              │  • Permission enforcement    │
              │  • Business logic            │
              │  • Audit log middleware      │
              │  • Report generation         │
              │  • Backup scheduler          │
              │  • Export engine             │
              └───────────────┬──────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
  ┌──────────────────┐ ┌────────────┐ ┌────────────────┐
  │  Supabase        │ │  Supabase  │ │  Supabase Auth │
  │  PostgreSQL      │ │  Storage   │ │                │
  │                  │ │            │ │  • JWT tokens  │
  │  18 tables       │ │  Buckets:  │ │  • TOTP/MFA    │
  │  8 migrations    │ │  vehicle-  │ │  • Sessions    │
  │  50+ indexes     │ │   media    │ │                │
  │  6 triggers      │ │  customer- │ └────────────────┘
  │  RLS enabled     │ │   docs     │
  └──────────────────┘ └────────────┘
```

---

## Layer Breakdown

### Layer 1 — Public Website (Next.js, Vercel)

**Purpose:** Customer-facing dealership presence. Completely decoupled from the admin system.

**Technical decisions:**
- Next.js SSR/SSG for fast load and SEO
- Calls API with anon key — only ever sees `status=available AND show_on_website=true` vehicles
- No login, no signup button — access to admin is invitation-only
- Shared Header/Footer components — one file per route rule enforced

**Routes:**
```
/           → Home — hero, featured vehicles, highlights
/inventory  → Live stock — filters, search, card/list view
/inventory/[stock_id] → Vehicle detail
/about      → Dealership story, team
/contact    → Contact form → POST /api/public/contact
```

---

### Layer 2 — Admin Dashboard (Next.js, Vercel)

**Purpose:** The internal business operating system — all 13 modules run here.

**Technical decisions:**
- Next.js middleware on every protected route — JWT validation on server
- Permission flags fetched on login, stored in memory — UI renders conditionally based on flags
- Sensitive fields (profit, minimum price) stripped from UI unless `VIEW_PROFIT = true`
- No direct Supabase calls from frontend — all data goes through the Express API

**Route protection pattern:**
```
/dashboard/*  → middleware checks JWT → resolves employee + permissions
              → redirects to /login if invalid
              → renders 403 if permission flag missing
```

**Frontend permission pattern:**
```typescript
// Fields only rendered if employee has VIEW_PROFIT flag
{employee.permissions.view_profit && (
  <ProfitCard value={vehicle.estimated_profit} />
)}
```

---

### Layer 3 — Express API (Node.js + TypeScript, Render)

**Purpose:** All business logic lives here. The frontend never touches the database directly.

**Middleware stack (applied to every request):**
```
Request
  → CORS validation
  → Rate limiting
  → JWT verification (Supabase Auth)
  → Employee resolution (loads employee + permissions from DB)
  → Route handler
  → Audit log middleware (auto-logs non-GET actions)
  → Response
```

**Permission enforcement pattern:**
```typescript
// Every protected route has a permission guard
router.get('/inventory', 
  requirePermission('inventory_read'), 
  inventoryController.list
);

router.patch('/inventory/:id',
  requirePermission('inventory_edit'),
  inventoryController.update
);

// Sensitive data stripped in response layer
function sanitizeVehicle(vehicle, permissions) {
  if (!permissions.view_profit) {
    delete vehicle.total_cost_cache;
    delete vehicle.minimum_price;
    delete vehicle.estimated_profit;
  }
  return vehicle;
}
```

**Background jobs (scheduled):**
```
2:00 AM daily   → backup job (DB snapshot + file archive)
3:00 AM Sunday  → weekly backup
Daily           → 365-day junk purge check
Daily           → backup retention cleanup (>14 daily, >8 weekly)
```

**Report export engine:**
The API generates reports in 4 formats from the same data query:
```
GET /reports/:type?format=excel|csv|pdf|docx
  → Query DB with filters
  → Format data per export type
  → Stream file download to client
  → Log export to audit_logs
```

---

### Layer 4 — Supabase PostgreSQL

**Purpose:** Single source of truth for all business data.

**Design decisions:**
- Service role key used by Express API — bypasses RLS for full access
- RLS enabled as safety net — prevents any direct DB access without API
- Anon key restricted to public vehicle reads only
- All computed values cached on the row and maintained by triggers — no expensive recalculation on every read
- Audit log table has INSERT+SELECT only at DB permission level — no application logic can override this

**Connection pattern:**
```
Express API → Supabase service role key → Full DB access
Public Next.js routes → Supabase anon key → vehicles (available+public) only
Admin Next.js → Never calls DB directly → All through Express API
```

---

### Layer 5 — Supabase Storage

**Purpose:** Binary file storage. DB stores only URLs, never binary data.

**Buckets:**
```
vehicle-media/
  └── {vehicle_id}/
        ├── main.jpg         ← main display image
        ├── photo-001.jpg
        ├── photo-002.jpg
        └── cr-copy.pdf

customer-docs/
  └── {customer_id}/
        ├── agreement.pdf
        └── id-scan.jpg
```

**Access policies:**
```
vehicle-media:  public read (photos shown on website)
                authenticated insert/update/delete (staff uploads)

customer-docs:  authenticated only (no public access)
```

---

## Data Flow Examples

### Deal Completion Flow
```
1. Salesperson clicks "Complete Deal"
2. API: verify JWT + check deals_edit permission
3. API: verify payment_status = fully_paid (business rule)
4. API: UPDATE deals SET status = 'completed', completed_at = now()
5. API: UPDATE vehicles SET status = 'sold', sold_price = deal.selling_price
6. API: INSERT INTO commissions (calculated from employee.commission_type)
7. API: INSERT INTO audit_logs (complete_deal action, before/after metadata)
8. Response: 200 OK with updated deal
```

### Payment Entry Flow
```
1. User adds a payment entry
2. API: INSERT INTO deal_payments
3. Trigger fires: fn_recalculate_deal_payment_status()
   → SELECT SUM(amount) FROM deal_payments WHERE deal_id = X
   → Determine new payment_status: unpaid / partially_paid / fully_paid
   → UPDATE deals SET total_paid_cache = ..., payment_status = ...
4. API: INSERT INTO audit_logs (payment added)
5. Response: updated deal with new totals
```

### Vehicle Cost Entry Flow
```
1. User adds a cost entry (e.g. repair: 15,000)
2. API: INSERT INTO vehicle_costs
3. Trigger fires: fn_recalculate_vehicle_total_cost()
   → SELECT purchase_price FROM vehicles
   → SELECT SUM(amount) FROM vehicle_costs WHERE vehicle_id = X
   → UPDATE vehicles SET total_cost_cache = purchase_price + sum
4. API: INSERT INTO audit_logs (cost added)
5. Frontend: estimated_profit updates live (asking_price − new total_cost_cache)
```

### First Login Flow
```
1. Employee enters email + password (issued by admin)
2. API: verify credentials via Supabase Auth
3. API: check must_change_password = true
4. Redirect to mandatory setup page (no other routes accessible)
5. Employee: changes password
6. Employee: scans QR code in Google Authenticator
7. Employee: enters 6-digit TOTP to confirm
8. Employee: updates profile (name, phone)
9. API: UPDATE employees SET must_change_password = false, totp_enabled = true
10. Employee: now has full system access based on permission flags
```

---

## Security Architecture

| Concern | Implementation |
|---------|---------------|
| **Authentication** | Email + Password + TOTP (all 3 required, no bypass) |
| **Session tokens** | Supabase JWT — short-lived, refreshed automatically |
| **API authorization** | Permission flags checked server-side on every protected endpoint |
| **DB access control** | RLS on all tables — service role for API, anon for public routes only |
| **Audit integrity** | INSERT+SELECT only on audit_logs at DB permission level |
| **Soft delete** | 3-layer protection — no accidental permanent data loss |
| **Secrets** | All keys in environment variables — never in codebase or committed |
| **Search path** | All trigger functions use `SET search_path = public` — prevents schema hijack |
| **File access** | Customer docs: authenticated only. Vehicle photos: public read |
| **Profit visibility** | Sensitive financial fields stripped from API response for unauthorized roles |

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Vercel                                │
│  ┌─────────────────────┐  ┌───────────────────────────┐ │
│  │ Public Website       │  │ Admin Dashboard           │ │
│  │ dms-web-chi.vercel  │  │ admin.dms.vercel.app      │ │
│  │ .app                │  │                           │ │
│  │ Auto CI/CD on push  │  │ Auto CI/CD on push        │ │
│  └─────────────────────┘  └───────────────────────────┘ │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    Render                                │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Express API                                        │  │
│  │ Always-on web service                             │  │
│  │ Environment: NODE_ENV, SUPABASE_URL,              │  │
│  │   SUPABASE_SERVICE_KEY, SUPABASE_ANON_KEY         │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    Supabase                              │
│  ┌─────────────────┐  ┌────────────┐  ┌──────────────┐ │
│  │ PostgreSQL       │  │ Storage    │  │ Auth         │ │
│  │ 18 tables        │  │ 2 buckets  │  │ TOTP/MFA     │ │
│  └─────────────────┘  └────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Environment separation:**
```
Development:  Local Next.js + Local Express + Supabase dev project
Production:   Vercel (frontend) + Render (API) + Supabase production project
```

---

## Key Technical Decisions & Why

| Decision | Reason |
|----------|--------|
| **Next.js for frontend** | SSR for public website SEO, middleware for auth protection, single framework for both public and admin |
| **Separate Express API** | Business logic isolated from frontend — permissions enforced server-side, not client-side |
| **Supabase PostgreSQL** | Managed Postgres with built-in Auth, Storage, and RLS — reduces infrastructure overhead without sacrificing control |
| **RLS as safety net** | Even if API has a bug, direct DB access is blocked. Defense in depth. |
| **Trigger-maintained caches** | total_cost_cache and total_paid_cache updated instantly on write — list views are fast reads with no aggregation query |
| **Append-only audit log** | DB-level INSERT-only permission — not just application convention. Tamper-proof by design. |
| **3-layer soft delete** | Protects against accidental data loss without relying on developer discipline — built into the data model |
| **Permission flags over role hierarchy** | Roles are a starting point — individual flags allow fine-grained access per employee beyond role defaults |
| **UUID primary keys** | No sequential ID exposure to clients — prevents enumeration attacks and reveals nothing about data volume |
