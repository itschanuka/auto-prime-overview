# API Overview — Auto Prime DMS

> **Node.js + Express (TypeScript) · REST API · Hosted on Render**  
> All endpoints require authentication via Supabase Auth JWT except public website routes.  
> All business logic, permission checks, and audit logging runs server-side in the API.

---

## Authentication

Every request to protected endpoints must include:

```
Authorization: Bearer <supabase_jwt_token>
```

The API validates the token via Supabase Auth, resolves the employee record and their permission flags, then enforces access before any business logic runs.

**Auth levels used in this document:**

| Level | Description |
|-------|-------------|
| `PUBLIC` | No auth required — public website routes |
| `AUTH` | Any logged-in employee |
| `PERMISSION: flag` | Requires specific permission flag on employee_permissions |
| `ROLE: admin` | Admin role only |
| `ROLE: admin/manager` | Admin or Manager role |

---

## Base URL

```
Production:  https://your-api.onrender.com/api
```

---

## Public Website Routes
No authentication required. Used by the Next.js public website.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/public/vehicles` | List available vehicles — only `status=available AND show_on_website=true`. Supports query params: `make`, `model`, `year`, `fuel_type`, `body_type`, `min_price`, `max_price`, `sort` |
| `GET` | `/public/vehicles/:stock_id` | Single vehicle detail — public fields only, no cost/profit data |
| `POST` | `/public/contact` | Submit contact form — saves to contact_submissions |

---

## Auth Routes

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/auth/login` | PUBLIC | Validate email + password + TOTP code. Returns JWT on success. |
| `POST` | `/auth/logout` | AUTH | Invalidate session, log to audit |
| `POST` | `/auth/first-setup` | AUTH | First login mandatory setup — change password + enable TOTP |
| `POST` | `/auth/change-password` | AUTH | Change own password |
| `POST` | `/auth/setup-totp` | AUTH | Set up Google Authenticator — returns QR code |
| `POST` | `/auth/verify-totp` | AUTH | Verify TOTP code during setup |
| `GET` | `/auth/me` | AUTH | Get own employee profile + permissions |

---

## Dashboard

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/dashboard/summary` | `PERMISSION: dashboard_read` | All KPI cards: stock counts, revenue, leads, commissions, follow-ups, finance pending |

---

## Inventory

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/inventory` | `PERMISSION: inventory_read` | List all vehicles. Filters: `status`, `make`, `model`, `year`, `location`, `aging_bucket`, `show_on_website` |
| `GET` | `/inventory/:id` | `PERMISSION: inventory_read` | Single vehicle detail — includes costs, documents, profit (if VIEW_PROFIT) |
| `POST` | `/inventory` | `PERMISSION: inventory_create` | Create new vehicle profile |
| `PATCH` | `/inventory/:id` | `PERMISSION: inventory_edit` | Update vehicle fields. Price fields require `EDIT_PRICE` permission |
| `PATCH` | `/inventory/:id/status` | `PERMISSION: inventory_edit` | Change vehicle status |
| `PATCH` | `/inventory/:id/website-toggle` | `PERMISSION: inventory_edit` | Toggle show_on_website |
| `DELETE` | `/inventory/:id` | `PERMISSION: inventory_delete` | Soft delete vehicle — moves to Trash |
| `POST` | `/inventory/:id/costs` | `PERMISSION: inventory_edit` | Add a cost entry to a vehicle |
| `PATCH` | `/inventory/:id/costs/:cost_id` | `PERMISSION: inventory_edit` | Edit a cost entry |
| `DELETE` | `/inventory/:id/costs/:cost_id` | `PERMISSION: inventory_edit` | Delete a cost entry |
| `POST` | `/inventory/:id/documents` | `PERMISSION: inventory_edit` | Upload document or photo |
| `PATCH` | `/inventory/:id/documents/:doc_id/main` | `PERMISSION: inventory_edit` | Set as main image |
| `DELETE` | `/inventory/:id/documents/:doc_id` | `PERMISSION: inventory_edit` | Remove document |

**Profit visibility rule:** Cost fields (`minimum_price`, `total_cost_cache`, profit calculations) only returned if employee has `VIEW_PROFIT = true`. Stripped from response otherwise.

---

## CRM — Lead Management

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/crm/leads` | `PERMISSION: crm_read` | List all leads. Filters: `status`, `assigned_to`, `source`, `overdue` (boolean) |
| `GET` | `/crm/leads/:id` | `PERMISSION: crm_read` | Lead detail including follow-up history |
| `POST` | `/crm/leads` | `PERMISSION: crm_create` | Create new lead |
| `PATCH` | `/crm/leads/:id` | `PERMISSION: crm_edit` | Update lead fields, status, next follow-up |
| `POST` | `/crm/leads/:id/followup` | `PERMISSION: crm_edit` | Log a follow-up action — appends to lead_follow_ups |
| `PATCH` | `/crm/leads/:id/status` | `PERMISSION: crm_edit` | Advance pipeline status with timestamp |
| `PATCH` | `/crm/leads/:id/lost` | `PERMISSION: crm_edit` | Mark lead as lost with reason and note |
| `DELETE` | `/crm/leads/:id` | `PERMISSION: crm_delete` | Soft delete lead |
| `GET` | `/crm/today` | `PERMISSION: crm_read` | Leads with follow-up due today |
| `GET` | `/crm/overdue` | `PERMISSION: crm_read` | Leads with overdue follow-up date |

---

## Sales & Deals

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/deals` | `PERMISSION: deals_read` | List all deals. Filters: `status`, `payment_status`, `salesperson_id`, `date_from`, `date_to` |
| `GET` | `/deals/:id` | `PERMISSION: deals_read` | Full deal detail — payments, finance, trade-in, delivery, commission, profit |
| `POST` | `/deals` | `PERMISSION: deals_create` | Create new deal |
| `PATCH` | `/deals/:id` | `PERMISSION: deals_edit` | Update deal fields |
| `PATCH` | `/deals/:id/complete` | `PERMISSION: deals_edit` | Complete deal — requires `payment_status = fully_paid` |
| `PATCH` | `/deals/:id/complete/override` | `ROLE: admin` | Admin override to complete without full payment — logged to audit |
| `PATCH` | `/deals/:id/cancel` | `PERMISSION: cancel_deal` | Cancel deal — reason required, vehicle reverts to available |
| `POST` | `/deals/:id/payments` | `PERMISSION: deals_edit` | Add payment entry — triggers total_paid_cache recalculation |
| `PATCH` | `/deals/:id/finance` | `PERMISSION: deals_edit` | Update finance/loan details and status |
| `PATCH` | `/deals/:id/tradein` | `PERMISSION: deals_edit` | Update trade-in details |
| `POST` | `/deals/:id/tradein/add-to-inventory` | `PERMISSION: inventory_create` | Add trade-in vehicle to inventory |
| `PATCH` | `/deals/:id/delivery` | `PERMISSION: deals_edit` | Update delivery checklist and status |
| `PATCH` | `/deals/:id/discount` | `PERMISSION: approve_discount` | Apply or approve discount on a deal |

**Profit visibility:** Profit fields in deal responses only returned if `VIEW_PROFIT = true`.

---

## Customer Management

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/customers` | `PERMISSION: customers_read` | List all customers. Filters: `customer_type`, `status`, `city` |
| `GET` | `/customers/:id` | `PERMISSION: customers_read` | Customer detail + linked deals, vehicles, payments, notes |
| `POST` | `/customers` | `PERMISSION: customers_create` | Create new customer |
| `PATCH` | `/customers/:id` | `PERMISSION: customers_edit` | Update customer profile |
| `DELETE` | `/customers/:id` | `PERMISSION: customers_delete` | Soft delete customer |
| `POST` | `/customers/:id/notes` | `PERMISSION: customers_edit` | Add internal note |
| `PATCH` | `/customers/:id/blacklist` | `PERMISSION: blacklist_customer` | Blacklist customer with reason — logged to audit |
| `PATCH` | `/customers/:id/unblacklist` | `ROLE: admin` | Remove blacklist status |

---

## Employee Management

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/employees` | `PERMISSION: employees_read` | List all employees |
| `GET` | `/employees/:id` | `PERMISSION: employees_read` | Employee detail + permissions + performance stats |
| `POST` | `/employees` | `PERMISSION: employees_create` | Create new employee — creates Supabase Auth account + employee_permissions row |
| `PATCH` | `/employees/:id` | `PERMISSION: employees_edit` | Update employee profile |
| `PATCH` | `/employees/:id/permissions` | `ROLE: admin` | Update permission flags |
| `PATCH` | `/employees/:id/deactivate` | `PERMISSION: employees_delete` | Deactivate account — cannot log in |
| `PATCH` | `/employees/:id/activate` | `ROLE: admin` | Reactivate account |
| `POST` | `/employees/:id/reset-password` | `ROLE: admin` | Force password reset + TOTP reset |
| `DELETE` | `/employees/:id` | `ROLE: admin` | Soft delete employee record |

---

## Commission Tracking

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/commissions` | `PERMISSION: commissions_read` | List commissions. Filters: `status`, `employee_id`, `month` |
| `GET` | `/commissions/summary` | `PERMISSION: commissions_read` | Per-employee summary: earned, paid, unpaid, deal count |
| `GET` | `/commissions/:id` | `PERMISSION: commissions_read` | Single commission detail |
| `PATCH` | `/commissions/:id/mark-paid` | `PERMISSION: commissions_edit` | Mark commission as paid — sets paid_at + paid_by |
| `PATCH` | `/commissions/:id/override` | `ROLE: admin` | Override commission amount — logged to audit |

---

## Reports & Analytics

All report endpoints accept query params: `date_from`, `date_to`, and report-specific filters.  
All `/export` sub-routes return file download in requested format.

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/reports/dashboard` | `PERMISSION: reports_read` | Analytics dashboard KPIs + charts data |
| `GET` | `/reports/inventory/stock` | `PERMISSION: reports_read` | Current stock report |
| `GET` | `/reports/inventory/aging` | `PERMISSION: reports_read` | Aging / dead stock report with buckets |
| `GET` | `/reports/inventory/profit-potential` | `PERMISSION: view_profit` | Estimated profit per vehicle |
| `GET` | `/reports/crm/lead-source` | `PERMISSION: reports_read` | Leads by source + conversion rates |
| `GET` | `/reports/crm/pipeline` | `PERMISSION: reports_read` | Lead count per pipeline stage |
| `GET` | `/reports/crm/followup-discipline` | `PERMISSION: reports_read` | Overdue follow-ups by salesperson |
| `GET` | `/reports/crm/lost-reasons` | `PERMISSION: reports_read` | Lost lead analysis |
| `GET` | `/reports/sales/summary` | `PERMISSION: reports_read` | Total deals, revenue, outstanding balances |
| `GET` | `/reports/sales/pending-balance` | `PERMISSION: reports_read` | All deals with outstanding balance |
| `GET` | `/reports/sales/finance` | `PERMISSION: reports_read` | Finance/loan deals by provider and status |
| `GET` | `/reports/sales/profit` | `PERMISSION: view_profit` | Profit per deal, monthly net profit summary |
| `GET` | `/reports/customers/repeat` | `PERMISSION: reports_read` | Repeat customers ranked by spend |
| `GET` | `/reports/customers/top` | `PERMISSION: reports_read` | Top customers by total spend + purchases |
| `GET` | `/reports/employees/performance` | `PERMISSION: reports_read` | Salesperson deals, conversion, revenue |
| `GET` | `/reports/commissions/summary` | `PERMISSION: commissions_read` | Commission totals and trends |
| `GET` | `/reports/commissions/unpaid` | `PERMISSION: commissions_read` | All unpaid commissions |
| `GET` | `/reports/expenses/summary` | `PERMISSION: expenses_read` | Expenses by category + monthly trend |
| `POST` | `/reports/:report_name/export` | `PERMISSION: export_reports` | Export any report — body: `{ format: 'pdf' | 'excel' | 'csv' | 'docx', filters: {} }` — logged to audit |

---

## Trash System

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/trash` | `PERMISSION: trash_read` | List soft-deleted records. Filter: `record_type` |
| `POST` | `/trash/:id/restore` | `PERMISSION: trash_read` | Restore record to original state — exact status preserved |
| `DELETE` | `/trash/:id/permanent` | `ROLE: admin` | Move to Junk folder (stage 2) |
| `GET` | `/trash/junk` | `ROLE: admin` | List stage 2 junk records with days-until-purge |
| `POST` | `/trash/junk/:id/restore` | `ROLE: admin` | Restore from Junk before 365-day purge |

> **Auto-purge:** Background job runs daily. Hard-deletes records in Junk where `permanent_deleted_at > 365 days ago`.

---

## Backup System

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/backups` | `PERMISSION: backups_read` | List backup history with status, sizes, timestamps |
| `POST` | `/backups/trigger` | `PERMISSION: backups_create` | Trigger a manual backup |
| `GET` | `/backups/:id/download/db` | `PERMISSION: backup_download` | Download database snapshot file |
| `GET` | `/backups/:id/download/files` | `PERMISSION: backup_download` | Download file archive |
| `GET` | `/backups/:id/download/full` | `PERMISSION: backup_download` | Download full package (DB + files) |
| `DELETE` | `/backups/:id` | `PERMISSION: backups_delete` | Delete backup record |

> All backup downloads logged to audit_logs with actor, timestamp, and which backup was accessed.

---

## Audit Log

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/audit` | `PERMISSION: audit_read` | List audit entries. Filters: `date_from`, `date_to`, `module`, `action`, `actor_id`, `entity_type` |
| `GET` | `/audit/:id` | `PERMISSION: audit_read` | Full entry detail — includes metadata JSON with before/after |
| `POST` | `/audit/export` | `PERMISSION: audit_read` | Export filtered audit log to CSV or PDF |

> Audit log is read-only via API. INSERT only — no update or delete endpoint exists.

---

## Contact Submissions (Admin)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/contact-submissions` | `ROLE: admin/manager` | List all submissions. Filter: `status` |
| `PATCH` | `/contact-submissions/:id/read` | `ROLE: admin/manager` | Mark as read |
| `PATCH` | `/contact-submissions/:id/archive` | `ROLE: admin/manager` | Archive submission |

---

## Standard Response Format

```json
{
  "success": true,
  "data": { },
  "message": "Optional message"
}
```

**Error format:**
```json
{
  "success": false,
  "error": "PERMISSION_DENIED",
  "message": "You do not have permission to view profit data."
}
```

**Common error codes:**

| Code | Meaning |
|------|---------|
| `UNAUTHORIZED` | No or invalid JWT token |
| `PERMISSION_DENIED` | Missing required permission flag |
| `NOT_FOUND` | Record does not exist or is soft-deleted |
| `VALIDATION_ERROR` | Invalid request body |
| `BUSINESS_RULE_VIOLATION` | e.g. completing a deal without full payment |
| `CONFLICT` | e.g. duplicate chassis VIN |

---

## Audit Logging

Every non-read API action is automatically logged to `audit_logs`. The API middleware handles this transparently — no endpoint needs to call it manually.

**Logged automatically on:**
- Any `POST`, `PATCH`, `DELETE` to protected endpoints
- Report exports and downloads
- Login / logout / failed login attempts
- MFA setup and password changes
- Permission changes
- Backup downloads

**Not logged:**
- `GET` requests (read actions) — keeps audit log clean and useful
