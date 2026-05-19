# Database Schema — Auto Prime DMS

> **18 tables · 8 migrations · PostgreSQL via Supabase**  
> All tables use UUID primary keys, TIMESTAMPTZ UTC timestamps, enforced foreign keys, and soft-delete columns on all deletable entities.

---

## Design Principles

| Principle | Implementation |
|-----------|---------------|
| **UUID primary keys** | `gen_random_uuid()` on every table — no sequential IDs leaked to clients |
| **Soft delete on all deletable tables** | `is_deleted`, `deleted_at`, `deleted_by`, `permanent_deleted_at`, `permanent_deleted_by` |
| **Append-only audit log** | DB-level INSERT+SELECT only on `audit_logs` — no UPDATE or DELETE ever |
| **Computed caches** | `total_cost_cache` on vehicles, `total_paid_cache` on deals — maintained by triggers |
| **RLS enforced** | Row Level Security enabled on all 18 tables — service role bypasses, anon role restricted |
| **Search path pinned** | All trigger functions use `SET search_path = public` to prevent hijack |
| **No binary in DB** | Files stored in Supabase Storage — only URLs/paths in Postgres |
| **Schema versioning** | 8 numbered migrations — tables never created by clicking |

---

## Entity Relationship Overview

```
employees ──────────────────────────────────────────────────────────┐
    │                                                                │
    ├── employee_permissions (1:1, auto-created by trigger)          │
    │                                                                │
    ├── vehicles ──── vehicle_costs (1:many)                         │
    │            ├─── vehicle_documents (1:many)                     │
    │            │                                                   │
    ├── customers ─── customer_notes (1:many)                        │
    │            │                                                   │
    ├── leads ────────────────────────────────── lead_follow_ups      │
    │            └── (won_deal_id → deals)                           │
    │                                                                │
    ├── deals ────── deal_payments (1:many)                           │
    │           ├─── deal_finance (1:1)                              │
    │           ├─── deal_trade_ins (1:1)                            │
    │           └─── deal_delivery (1:1, auto-created by trigger)    │
    │                                                                │
    ├── commissions (1:1 per deal)                                   │
    ├── expenses (general business expenses)                         │
    ├── contact_submissions (public website enquiries)               │
    ├── audit_logs (APPEND ONLY — no updates ever)                   │
    └── backup_records                                               │
                                                                     │
All tables reference employees.id as created_by / deleted_by ───────┘
```

---

## Table Reference

### TABLE 1: `employees`
Core user table. All system accounts created by Admin — no public self-registration.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Auto-generated |
| `auth_user_id` | UUID | UNIQUE, FK → auth.users | Supabase Auth link |
| `employee_code` | VARCHAR(20) | UNIQUE NOT NULL | e.g. EMP001 |
| `full_name` | VARCHAR(200) | NOT NULL | |
| `email` | VARCHAR(255) | UNIQUE NOT NULL | |
| `phone` | VARCHAR(30) | | |
| `address` | TEXT | | |
| `nic` | VARCHAR(50) | | National ID |
| `join_date` | DATE | NOT NULL | |
| `role` | VARCHAR(30) | CHECK: admin/manager/salesperson/accountant | |
| `status` | VARCHAR(20) | DEFAULT active, CHECK: active/inactive | |
| `commission_type` | VARCHAR(20) | CHECK: fixed/percent_price/percent_profit | NULL for non-sales roles |
| `commission_value` | NUMERIC(12,4) | | Rate or fixed amount |
| `totp_enabled` | BOOLEAN | DEFAULT false | Google Authenticator setup |
| `must_change_password` | BOOLEAN | DEFAULT true | Forces password change on first login |
| `last_login_at` | TIMESTAMPTZ | | |
| `created_by` | UUID | FK → employees | Admin who created this account |
| `created_at` | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | DEFAULT now() | Auto-updated by trigger |
| `is_deleted` | BOOLEAN | DEFAULT false | Soft delete flag |
| `deleted_at` | TIMESTAMPTZ | NULL | |
| `deleted_by` | UUID | FK → employees | |
| `permanent_deleted_at` | TIMESTAMPTZ | NULL | Stage 2 delete |
| `permanent_deleted_by` | UUID | FK → employees | |

**Indexes:** email, auth_user_id, role, status, is_deleted, composite (status + is_deleted WHERE active)

---

### TABLE 2: `employee_permissions`
One row per employee. Auto-created by trigger on employee INSERT. All flags default false — Admin role bypasses all flags at API level.

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| `id` | UUID | PK | |
| `employee_id` | UUID | UNIQUE FK → employees | |
| `view_profit` | BOOLEAN | false | See profit figures in inventory and deals |
| `edit_price` | BOOLEAN | false | Modify asking/minimum price |
| `delete_records` | BOOLEAN | false | Soft-delete across modules |
| `view_reports` | BOOLEAN | false | Access reports section |
| `manage_employees` | BOOLEAN | false | Create/edit/deactivate employees |
| `audit_view` | BOOLEAN | false | Read the audit log |
| `backup_download` | BOOLEAN | false | Download backup files |
| `approve_discount` | BOOLEAN | false | Approve deal discounts |
| `cancel_deal` | BOOLEAN | false | Cancel an active deal |
| `blacklist_customer` | BOOLEAN | false | Mark a customer as blacklisted |
| `export_reports` | BOOLEAN | false | Export reports to file |
| `dashboard_read` | BOOLEAN | false | Access dashboard |
| `inventory_read/create/edit/delete` | BOOLEAN | false | Granular inventory CRUD |
| `crm_read/create/edit/delete` | BOOLEAN | false | Granular CRM CRUD |
| `deals_read/create/edit/delete` | BOOLEAN | false | Granular deals CRUD |
| `customers_read/create/edit/delete` | BOOLEAN | false | Granular customer CRUD |
| `commissions_read/edit` | BOOLEAN | false | Commission access (no create — auto-created by system) |
| `expenses_read/create/edit/delete` | BOOLEAN | false | Expense CRUD |
| `employees_read/create/edit/delete` | BOOLEAN | false | Employee management CRUD |
| `reports_read` | BOOLEAN | false | Reports section access |
| `trash_read/delete` | BOOLEAN | false | Trash management |
| `audit_read` | BOOLEAN | false | Audit log access |
| `backups_read/create/delete` | BOOLEAN | false | Backup management |
| `updated_at` | TIMESTAMPTZ | now() | |
| `updated_by` | UUID | FK → employees | Who last changed permissions |

> **Trigger:** `trg_employees_create_permissions` — auto-inserts a default permissions row on every new employee.

---

### TABLE 3: `vehicles`
Complete vehicle lifecycle — draft through sold. The financial core of the inventory system.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `stock_id` | VARCHAR(30) | UNIQUE — e.g. STK-2024-001 |
| `make` | VARCHAR(100) | Toyota, Honda, etc. |
| `model` | VARCHAR(100) | |
| `variant` | VARCHAR(100) | Trim level |
| `year` | SMALLINT | Manufacturing year |
| `mileage` | INTEGER | DEFAULT 0 |
| `engine_capacity` | VARCHAR(20) | e.g. 1500cc |
| `transmission` | VARCHAR(30) | CHECK: manual/automatic/cvt |
| `fuel_type` | VARCHAR(30) | CHECK: petrol/diesel/hybrid/electric |
| `color` | VARCHAR(60) | |
| `body_type` | VARCHAR(50) | CHECK: sedan/suv/hatchback/van/pickup/coupe/convertible/wagon/other |
| `condition` | VARCHAR(30) | CHECK: used/reconditioned/brand_new |
| `location` | VARCHAR(100) | Yard/branch |
| `chassis_vin` | VARCHAR(50) | Unique active index (partial — WHERE is_deleted = false) |
| `engine_number` | VARCHAR(50) | |
| `registration_number` | VARCHAR(30) | |
| `import_batch_ref` | VARCHAR(50) | Optional grouped import tracking |
| `purchase_date` | DATE | NOT NULL |
| `supplier_name` | VARCHAR(200) | |
| `supplier_contact` | VARCHAR(100) | |
| `purchase_type` | VARCHAR(30) | CHECK: auction/trade_in/direct_seller/private/dealer/other |
| `purchase_price` | NUMERIC(14,2) | Base acquisition cost |
| `purchase_payment_status` | VARCHAR(20) | CHECK: paid/pending/partial |
| `asking_price` | NUMERIC(14,2) | Listed/advertised price |
| `minimum_price` | NUMERIC(14,2) | Floor price — permission-gated |
| `sold_price` | NUMERIC(14,2) | Set on deal completion |
| `total_cost_cache` | NUMERIC(14,2) | DEFAULT 0 — auto-maintained by trigger |
| `status` | VARCHAR(20) | CHECK: draft/in_repair/available/reserved/sold/written_off |
| `show_on_website` | BOOLEAN | DEFAULT false — independent of status |
| `main_image_url` | TEXT | Cache updated by trigger on vehicle_documents |
| `notes` | TEXT | Internal notes |
| `created_by` | UUID | FK → employees |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | Auto-updated by trigger |
| soft delete columns | — | is_deleted, deleted_at, deleted_by, permanent_deleted_at, permanent_deleted_by |

**Key formulas:**
```
total_cost_cache = purchase_price + SUM(vehicle_costs.amount)   ← trigger-maintained
estimated_profit = asking_price − total_cost_cache              ← computed in query
profit_after_sale = sold_price − total_cost_cache               ← set on completion
```

**Triggers:**
- `trg_vehicles_updated_at` — auto-updates updated_at on every change
- `trg_vehicle_costs_recalculate` — recalculates total_cost_cache on any vehicle_costs INSERT/UPDATE/DELETE

---

### TABLE 4: `vehicle_costs`
Multi-entry cost records per vehicle. No limit on entries. Trigger auto-totals into vehicles.total_cost_cache.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `vehicle_id` | UUID | FK → vehicles CASCADE |
| `cost_date` | DATE | NOT NULL |
| `category` | VARCHAR(50) | CHECK: repair/spare_parts/paint_bodywork/service/transport/auction_import_fees/advertising/other |
| `description` | TEXT | NOT NULL |
| `amount` | NUMERIC(14,2) | CHECK: amount > 0 |
| `receipt_files` | JSONB | Array of `{url, name}` objects for receipt uploads |
| `created_by` | UUID | FK → employees |
| `created_at` | TIMESTAMPTZ | |

---

### TABLE 5: `vehicle_documents`
Photos and documents per vehicle. Stored in Supabase Storage — only URLs saved here.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `vehicle_id` | UUID | FK → vehicles CASCADE |
| `doc_type` | VARCHAR(40) | CHECK: photo/cr_copy/import_doc/repair_invoice/other |
| `file_url` | TEXT | NOT NULL — Supabase Storage URL |
| `file_name` | VARCHAR(255) | Original filename |
| `is_main_image` | BOOLEAN | DEFAULT false |
| `sort_order` | SMALLINT | DEFAULT 0 |
| `uploaded_by` | UUID | FK → employees |
| `uploaded_at` | TIMESTAMPTZ | |

**Trigger:** `trg_vehicle_documents_main_image` — when is_main_image set to true, clears all other main_image flags for that vehicle and updates vehicles.main_image_url cache.

---

### TABLE 6: `customers`
Customer profiles. Supports individual, business, dealer, and repeat buyer types.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `customer_code` | VARCHAR(20) | UNIQUE — e.g. CUS-0001 |
| `full_name` | VARCHAR(200) | NOT NULL |
| `phone_primary` | VARCHAR(30) | UNIQUE NOT NULL |
| `phone_secondary` | VARCHAR(30) | |
| `email` | VARCHAR(255) | |
| `address` | TEXT | |
| `city` | VARCHAR(100) | |
| `nic_passport` | VARCHAR(50) | UNIQUE — nullable |
| `business_name` | VARCHAR(200) | For business/dealer type |
| `business_reg_no` | VARCHAR(80) | |
| `customer_type` | VARCHAR(30) | CHECK: individual/business/dealer_trader/repeat_buyer |
| `status` | VARCHAR(20) | CHECK: active/inactive/blacklisted |
| `blacklist_reason` | TEXT | |
| `blacklisted_by` | UUID | FK → employees |
| `blacklisted_at` | TIMESTAMPTZ | |
| `created_by` | UUID | FK → employees |
| `created_at` / `updated_at` | TIMESTAMPTZ | |
| soft delete columns | — | Full 5-column set |

---

### TABLE 7: `customer_notes`
Internal staff notes on customers. Append-only in practice — no edit on notes.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `customer_id` | UUID | FK → customers CASCADE |
| `note` | TEXT | NOT NULL |
| `created_by` | UUID | FK → employees |
| `created_at` | TIMESTAMPTZ | |

---

### TABLE 8: `leads`
CRM pipeline records. Every lead must have a next_followup_date while active.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `lead_code` | VARCHAR(20) | UNIQUE — e.g. LD-2024-001 |
| `customer_id` | UUID | FK → customers, nullable (unknown walkins) |
| `customer_name` | VARCHAR(200) | Snapshot — preserved even if customer changes |
| `customer_phone` | VARCHAR(30) | Snapshot |
| `interested_vehicle_id` | UUID | FK → vehicles, nullable |
| `interested_vehicle_desc` | VARCHAR(200) | Free text if no specific vehicle |
| `source` | VARCHAR(40) | CHECK: walk_in/call/website/facebook/whatsapp/referral/other |
| `assigned_to` | UUID | FK → employees |
| `status` | VARCHAR(20) | CHECK: new/contacted/interested/test_drive/negotiation/won/lost |
| `next_followup_date` | DATE | Required on all active leads |
| `next_followup_note` | TEXT | |
| `lost_reason` | VARCHAR(60) | CHECK: price_too_high/competitor/not_interested/financing_rejected/other |
| `lost_note` | TEXT | |
| `won_deal_id` | UUID | FK → deals (forward ref — added via ALTER after deals created) |
| `created_by` | UUID | FK → employees |
| `created_at` / `updated_at` | TIMESTAMPTZ | |
| soft delete columns | — | Full 5-column set |

**Partial index for overdue follow-ups:**
```sql
WHERE status NOT IN ('won','lost') AND is_deleted = false
```

---

### TABLE 9: `lead_follow_ups`
Immutable history log of every follow-up action on a lead.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `lead_id` | UUID | FK → leads CASCADE |
| `follow_up_date` | DATE | NOT NULL |
| `status_at_time` | VARCHAR(20) | Pipeline stage snapshot at time of follow-up |
| `notes` | TEXT | NOT NULL |
| `next_followup_date` | DATE | What was set for the next follow-up |
| `created_by` | UUID | FK → employees |
| `created_at` | TIMESTAMPTZ | |

---

### TABLE 10: `deals`
The financial core of the system. Every vehicle sale is a deal. All money flows through here.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `deal_code` | VARCHAR(20) | UNIQUE — e.g. DL-2024-001 |
| `vehicle_id` | UUID | FK → vehicles |
| `customer_id` | UUID | FK → customers |
| `salesperson_id` | UUID | FK → employees |
| `deal_date` | DATE | NOT NULL |
| `status` | VARCHAR(20) | CHECK: draft/reserved/active/completed/cancelled |
| `selling_price` | NUMERIC(14,2) | Agreed final price |
| `discount_amount` | NUMERIC(14,2) | DEFAULT 0 |
| `discount_approved_by` | UUID | FK → employees |
| `payment_type` | VARCHAR(20) | CHECK: cash/finance/mixed |
| `total_paid_cache` | NUMERIC(14,2) | DEFAULT 0 — auto-maintained by trigger |
| `payment_status` | VARCHAR(20) | CHECK: unpaid/partially_paid/fully_paid — auto-calculated |
| `reservation_amount` | NUMERIC(14,2) | |
| `reservation_date` | DATE | |
| `reservation_expiry` | DATE | Display only — no auto-expiry |
| `cancellation_reason` | TEXT | |
| `refund_amount` | NUMERIC(14,2) | |
| `completion_override_by` | UUID | FK → employees — Admin override logged to audit |
| `notes` | TEXT | |
| `created_by` | UUID | FK → employees |
| `created_at` / `updated_at` / `completed_at` | TIMESTAMPTZ | |
| soft delete columns | — | Full 5-column set |

**Completion rule:** `payment_status = fully_paid` required before deal can be marked Completed. Admin override available — logged to audit_logs.

**On completion trigger (API-side):** vehicle.status → `sold`, vehicle.sold_price set, commission auto-created.

**Trigger:** `trg_deals_create_delivery` — auto-creates a deal_delivery row on every new deal.

---

### TABLE 11: `deal_payments`
Multi-entry payment records per deal. Trigger auto-recalculates deal totals on every INSERT.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `deal_id` | UUID | FK → deals CASCADE |
| `payment_date` | DATE | NOT NULL |
| `amount` | NUMERIC(14,2) | CHECK: amount > 0 |
| `method` | VARCHAR(40) | CHECK: cash/bank_transfer/cheque/finance_disbursement |
| `reference_number` | VARCHAR(100) | Bank ref, cheque number |
| `notes` | TEXT | |
| `is_finance_disbursement` | BOOLEAN | DEFAULT false — flags loan disbursement entries |
| `created_by` | UUID | FK → employees |
| `created_at` | TIMESTAMPTZ | |

**Trigger:** `trg_deal_payments_update_cache` — on every INSERT, recalculates:
```
total_paid_cache = SUM(deal_payments.amount) for this deal
payment_status   = unpaid / partially_paid / fully_paid
```

---

### TABLE 12: `deal_finance`
One row per deal (if financed). Tracks bank/finance company details and loan progression.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `deal_id` | UUID | UNIQUE FK → deals CASCADE |
| `provider_type` | VARCHAR(30) | CHECK: bank/finance_company |
| `provider_name` | VARCHAR(200) | Institution name |
| `branch` | VARCHAR(200) | |
| `officer_name` / `officer_contact` | VARCHAR | Contact at provider |
| `application_date` | DATE | |
| `loan_status` | VARCHAR(20) | CHECK: not_started/submitted/approved/rejected/disbursed |
| `selling_price_ref` | NUMERIC(14,2) | Deal price snapshot |
| `customer_down_payment` | NUMERIC(14,2) | DEFAULT 0 |
| `loan_amount_requested` | NUMERIC(14,2) | Applied for |
| `loan_amount_approved` | NUMERIC(14,2) | Bank approved |
| `loan_amount_disbursed` | NUMERIC(14,2) | Actually released — triggers payment entry |
| `disbursement_date` | DATE | |
| `notes` | TEXT | |
| `created_by` | UUID | FK → employees |
| `created_at` / `updated_at` | TIMESTAMPTZ | |

> When loan is disbursed: disbursed amount is added as a `finance_disbursement` entry in deal_payments automatically.

---

### TABLE 13: `deal_trade_ins`
One row per deal (if customer trades in a vehicle). Trade-in vehicle optionally linked back to inventory.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `deal_id` | UUID | UNIQUE FK → deals CASCADE |
| `make` / `model` | VARCHAR(100) | Trade-in vehicle info |
| `year` | SMALLINT | |
| `mileage` | INTEGER | |
| `registration_number` | VARCHAR(30) | |
| `condition_notes` | TEXT | |
| `trade_in_value` | NUMERIC(14,2) | NOT NULL — agreed value |
| `added_to_inventory_id` | UUID | FK → vehicles — links to new inventory entry |
| `created_by` | UUID | FK → employees |
| `created_at` | TIMESTAMPTZ | |

---

### TABLE 14: `deal_delivery`
One row per deal (auto-created by trigger). Tracks handover checklist and delivery status.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `deal_id` | UUID | UNIQUE FK → deals CASCADE |
| `delivery_date` | DATE | Planned or actual |
| `delivery_status` | VARCHAR(20) | CHECK: pending/delivered |
| `check_payment_received` | BOOLEAN | DEFAULT false |
| `check_agreement_signed` | BOOLEAN | DEFAULT false |
| `check_docs_handed_over` | BOOLEAN | DEFAULT false |
| `check_vehicle_handed_over` | BOOLEAN | DEFAULT false |
| `delivery_notes` | TEXT | |
| `updated_by` | UUID | FK → employees |
| `updated_at` | TIMESTAMPTZ | Auto-updated by trigger |

---

### TABLE 15: `commissions`
One row per completed deal. Auto-calculated on deal completion. Supports manual override by Admin.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `deal_id` | UUID | UNIQUE FK → deals CASCADE |
| `employee_id` | UUID | FK → employees (salesperson) |
| `commission_type` | VARCHAR(20) | CHECK: fixed/percent_price/percent_profit |
| `commission_rate` | NUMERIC(10,4) | Percentage rate used |
| `fixed_value` | NUMERIC(14,2) | If fixed type |
| `base_amount` | NUMERIC(14,2) | Selling price or profit — basis for calculation |
| `calculated_amount` | NUMERIC(14,2) | System-calculated amount |
| `manual_override_amount` | NUMERIC(14,2) | Admin override — logged to audit |
| `override_by` | UUID | FK → employees |
| `final_amount` | NUMERIC(14,2) | calculated_amount OR manual_override_amount |
| `status` | VARCHAR(10) | CHECK: unpaid/paid |
| `paid_at` | TIMESTAMPTZ | |
| `paid_by` | UUID | FK → employees |
| `notes` | TEXT | |
| `created_at` | TIMESTAMPTZ | |

> Commission only created when `deal.status = COMPLETED`. Never created for cancelled or active deals.

---

### TABLE 16: `expenses`
General and vehicle-linked business expenses. Optionally linked to a vehicle for per-vehicle cost reporting.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `expense_date` | DATE | NOT NULL |
| `category` | VARCHAR(60) | CHECK: rent/utilities/salaries/marketing/office/vehicle_repair/transport/other |
| `description` | TEXT | NOT NULL |
| `amount` | NUMERIC(14,2) | CHECK: amount > 0 |
| `vehicle_id` | UUID | FK → vehicles, nullable — links to specific vehicle |
| `reference` | VARCHAR(200) | Invoice or reference number |
| `notes` | TEXT | |
| `created_by` | UUID | FK → employees |
| `created_at` / `updated_at` | TIMESTAMPTZ | |
| soft delete columns | — | Full 5-column set |

---

### TABLE 17: `audit_logs` ⚠️ APPEND ONLY
Tamper-proof, immutable record of every important action. No UPDATE or DELETE ever runs on this table.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `timestamp` | TIMESTAMPTZ | DEFAULT now() |
| `actor_type` | VARCHAR(10) | CHECK: user/system |
| `actor_id` | UUID | employees.id — no FK (allows purged employees to remain in log) |
| `actor_name` | VARCHAR(200) | **Snapshot** — preserved even if employee name changes later |
| `module` | VARCHAR(30) | CHECK: auth/inventory/crm/sales/customers/employees/commission/expenses/reports/trash/backup |
| `action` | VARCHAR(40) | CHECK: create/update/delete/restore/perm_delete/export/login/login_fail/logout/mfa_setup/password_change/backup_start/backup_success/backup_fail/backup_download/purge/mark_paid/override/blacklist/complete_deal/cancel_deal |
| `entity_type` | VARCHAR(40) | vehicle/lead/deal/payment/customer/employee/commission/etc. |
| `entity_id` | UUID | ID of the affected record |
| `message` | TEXT | Human-readable summary |
| `metadata` | JSONB | Before/after changes, additional context |
| `ip_address` | INET | Client IP |
| `prev_hash` | VARCHAR(64) | Optional — hash of previous entry for chain integrity |
| `row_hash` | VARCHAR(64) | Optional — hash of this entry |

**RLS enforcement:**
```sql
-- App role: INSERT + SELECT only
-- No UPDATE policy = impossible to update
-- No DELETE policy = impossible to delete
```

**Example metadata for a price change:**
```json
{
  "changes": {
    "asking_price": { "from": 9500000, "to": 9200000 },
    "status": { "from": "available", "to": "reserved" }
  }
}
```

---

### TABLE 18: `backup_records`
Automated backup history. Supports daily, weekly, and manual backup types.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `backup_type` | VARCHAR(10) | CHECK: daily/weekly/manual |
| `status` | VARCHAR(10) | CHECK: running/success/failed |
| `triggered_by` | TEXT | DEFAULT system — or employee name for manual |
| `started_at` | TIMESTAMPTZ | NOT NULL |
| `completed_at` | TIMESTAMPTZ | |
| `file_name` | TEXT | Backup archive filename |
| `file_path` | TEXT | Storage path |
| `db_file_url` | TEXT | Legacy — superseded by file_path |
| `files_archive_url` | TEXT | |
| `db_size_bytes` | BIGINT | Database snapshot size |
| `files_size_bytes` | BIGINT | File archive size |
| `tables_included` | TEXT[] | Array of table names backed up |
| `row_counts` | JSONB | Row count per table at backup time |
| `duration_ms` | INTEGER | Backup duration in milliseconds |
| `error_message` | TEXT | Populated on failure |
| `retention_delete_scheduled` | BOOLEAN | DEFAULT false — marks for retention cleanup |
| `created_at` | TIMESTAMPTZ | NOT NULL |

**Retention:** 14 daily backups, 8 weekly backups. Daily job auto-purges older records.

---

### TABLE 19: `contact_submissions`
Public website contact form submissions. Service role only — no anon/authenticated direct access.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `name` | VARCHAR(200) | NOT NULL |
| `phone` | VARCHAR(30) | NOT NULL |
| `email` | VARCHAR(255) | |
| `subject` | VARCHAR(100) | NOT NULL |
| `message` | TEXT | NOT NULL |
| `vehicle_ref` | VARCHAR(50) | Stock ID or description of enquired vehicle |
| `status` | VARCHAR(20) | CHECK: new/read/replied/archived |
| `ip_address` | INET | |
| `created_at` | TIMESTAMPTZ | NOT NULL |
| `read_at` | TIMESTAMPTZ | |
| `read_by` | UUID | FK → employees |

---

## Triggers Summary

| Trigger | Table | Event | Action |
|---------|-------|-------|--------|
| `trg_employees_updated_at` | employees | BEFORE UPDATE | Sets updated_at = now() |
| `trg_vehicles_updated_at` | vehicles | BEFORE UPDATE | Sets updated_at = now() |
| `trg_customers_updated_at` | customers | BEFORE UPDATE | Sets updated_at = now() |
| `trg_leads_updated_at` | leads | BEFORE UPDATE | Sets updated_at = now() |
| `trg_deals_updated_at` | deals | BEFORE UPDATE | Sets updated_at = now() |
| `trg_deal_finance_updated_at` | deal_finance | BEFORE UPDATE | Sets updated_at = now() |
| `trg_deal_delivery_updated_at` | deal_delivery | BEFORE UPDATE | Sets updated_at = now() |
| `trg_expenses_updated_at` | expenses | BEFORE UPDATE | Sets updated_at = now() |
| `trg_employee_permissions_updated_at` | employee_permissions | BEFORE UPDATE | Sets updated_at = now() |
| `trg_vehicle_costs_recalculate` | vehicle_costs | AFTER INSERT/UPDATE/DELETE | Recalculates vehicles.total_cost_cache |
| `trg_deal_payments_update_cache` | deal_payments | AFTER INSERT | Recalculates deals.total_paid_cache + payment_status |
| `trg_employees_create_permissions` | employees | AFTER INSERT | Auto-creates employee_permissions row |
| `trg_deals_create_delivery` | deals | AFTER INSERT | Auto-creates deal_delivery row |
| `trg_vehicle_documents_main_image` | vehicle_documents | AFTER INSERT/UPDATE | Enforces single main image per vehicle, updates vehicles.main_image_url |

---

## RLS Policy Summary

| Role | Access |
|------|--------|
| **service_role** (Express API) | Bypasses all RLS — full access |
| **anon** | Read available vehicles + photos for public website only |
| **authenticated** | Self-read on own employee + permissions row only |
| All other tables | Explicit DENY ALL for anon + authenticated |

---

## Migration History

| Migration | Description |
|-----------|-------------|
| 001 | Initial schema — 18 tables in dependency order |
| 002 | Performance indexes — 50+ indexes across all tables |
| 003 | Row Level Security policies |
| 004 | Database triggers — 6 trigger functions |
| 005 | contact_submissions table + vehicle notes column |
| 006 | Added written_off to vehicles.status constraint |
| 007 | Upgraded backup_records — manual backup type + new columns |
| 008 | Fixed all Supabase linter warnings — RLS initplan, search_path, 33 FK indexes |
| 009 | Granular CRUD permission flags on employee_permissions |
