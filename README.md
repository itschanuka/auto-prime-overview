<div align="center">

<img src="https://img.shields.io/badge/Status-Live%20Demo%20Available-0ea5e9?style=for-the-badge" />
<img src="https://img.shields.io/badge/Modules-13-6366f1?style=for-the-badge" />
<img src="https://img.shields.io/badge/Stack-Next.js%20%7C%20TypeScript%20%7C%20Node.js%20%7C%20PostgreSQL-0f172a?style=for-the-badge" />
<img src="https://img.shields.io/badge/Build-6%20Months-f59e0b?style=for-the-badge" />

# 🚗 Auto Prime — Dealership Management System

**A production-grade, full-stack business operating system built for vehicle dealerships.**  
Replaces spreadsheets, WhatsApp threads, and manual processes with a structured, role-controlled platform.

[**🔴 Live Demo →**](https://dms-web-chi.vercel.app) &nbsp;&nbsp; [**📬 Request Source Code Access →**](#-source-code-access)

---

</div>

## 📌 What This Is

Auto Prime is a **complete dealership management system** — not a UI demo, not a CRUD app. It covers every real business operation a vehicle dealership runs daily:

- Tracking stock from purchase → repair → listing → reservation → sale
- Managing leads through a full CRM pipeline from first inquiry to deal
- Running deals with multi-entry payments, bank finance tracking, and trade-in handling
- Automated commission calculation per salesperson when a deal closes
- Full reporting suite with PDF / Excel / CSV / DOCX exports
- Immutable audit log with before/after change tracking on every action
- 3-layer soft-delete protection with 365-day auto-purge
- Automated daily and weekly encrypted backups

> **Built in 6 months. Designed and developed solo — architecture, database, API, frontend, deployment, and documentation.**

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    PUBLIC WEBSITE                        │
│         Next.js SSR · Public Inventory · Contact        │
└───────────────────┬─────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────┐
│              AUTHENTICATION LAYER                        │
│    Supabase Auth · MFA (Google Authenticator) · RBAC    │
│         Invitation-only · No public registration        │
└───────────────────┬─────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────┐
│               ADMIN DASHBOARD                           │
│                                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │Inventory │ │   CRM    │ │  Sales   │ │Customers │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │Employees │ │Commission│ │ Reports  │ │  Trash   │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│  │  Backup  │ │Audit Log │ │Dashboard │               │
│  └──────────┘ └──────────┘ └──────────┘               │
└───────────────────┬─────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌───────────────┐     ┌─────────────────┐
│  Node.js API  │     │ Supabase Storage │
│  Express.js   │     │  Photos · Docs  │
│  TypeScript   │     └─────────────────┘
└───────┬───────┘
        ▼
┌───────────────┐
│   Supabase    │
│  PostgreSQL   │
│  + RLS Rules  │
└───────────────┘
```

---

## ⚙️ Tech Stack

| Layer | Technology | Detail |
|-------|-----------|--------|
| **Frontend** | Next.js + TypeScript | SSR/SSG, protected routes via middleware |
| **Backend** | Node.js + Express (TypeScript) | REST API, business logic, report generation |
| **Database** | Supabase PostgreSQL | Row Level Security, schema migrations |
| **Auth** | Supabase Auth | MFA via Google Authenticator (TOTP), no public signup |
| **File Storage** | Supabase Storage | Vehicle photos, customer docs, agreements |
| **Frontend Hosting** | Vercel | Automatic CI/CD on push |
| **Backend Hosting** | Render | Always-on API with environment isolation |
| **Exports** | PDF · Excel · CSV · DOCX | Every report supports all 4 formats |

---

## 🗂️ The 13 Modules

### Module 1 — Public Website
The customer-facing side of the business. Built as a separate Next.js presentation layer, fully decoupled from the admin system.

| Page | Contents |
|------|----------|
| Home | Hero, featured vehicles, CTA, dealership highlights |
| Inventory | Live available stock — searchable, filterable, card/list view |
| About | Dealership story, team, trust signals |
| Contact | Contact form, phone, address, WhatsApp link, map embed |

**Key rule:** Login is never linked from the public site. Admin access is invitation-only via a private link.

---

### Module 2 — Authentication & Access Control
Zero-trust authentication. No user can self-register. All accounts created by Admin only.

**Login flow (every login, no exceptions):**
```
Email → Password → Google Authenticator TOTP code → Access granted
```

**First login mandatory setup:**
```
Change temporary password → Set up Google Authenticator → Update profile → System access unlocked
```

**Permission system (stored in DB, not role-hardcoded):**

| Permission Flag | What It Controls | Who Can Grant |
|-----------------|-----------------|---------------|
| `VIEW_PROFIT` | Profit visibility in inventory and deals | Admin only |
| `EDIT_PRICE` | Modify asking/minimum price | Admin only |
| `DELETE_RECORDS` | Soft-delete across all modules | Admin / Manager |
| `VIEW_REPORTS` | Access the reports section | Admin / Manager |
| `MANAGE_EMPLOYEES` | Create, edit, deactivate employees | Admin only |
| `AUDIT_VIEW` | Read the audit log | Admin only |
| `BACKUP_DOWNLOAD` | Download backup files | Admin only (can delegate) |
| `APPROVE_DISCOUNT` | Approve deal discounts | Admin / Manager |
| `CANCEL_DEAL` | Cancel an active deal | Admin / Manager |
| `BLACKLIST_CUSTOMER` | Flag a customer as blacklisted | Admin only |

---

### Module 3 — Inventory Management
Complete vehicle lifecycle management from acquisition to sale.

**Vehicle Status Flow:**
```
Draft → In Repair → Available → Reserved → Sold
```

**Cost tracking per vehicle (multi-entry, auto-totalling):**

| Cost Category | Tracked Per Entry |
|--------------|------------------|
| Purchase Price | Base acquisition cost |
| Repair / Parts | Individual repair entries |
| Paint & Bodywork | Body shop work |
| Transport | Movement/delivery costs |
| Auction / Import Fees | Fees at purchase stage |
| Advertising | Per-vehicle marketing spend |

```
Total Cost = Purchase Price + Σ All Cost Entries
Estimated Profit = Asking Price − Total Cost  [live, shown in list]
Profit After Sale = Sold Price − Total Cost   [set on deal completion]
```

**Stock aging system (auto-categorized):**

| Bucket | Definition | Action |
|--------|-----------|--------|
| 0–30 days | Fresh stock | None |
| 31–60 days | Standard aging | Monitor |
| 61–90 days | Attention needed | Flag |
| 90+ days | **Dead stock** | Flagged on dashboard — capital tied up |

**Website visibility toggle:** Each vehicle has an independent public visibility toggle separate from its status. Available ≠ automatically visible.

---

### Module 4 — CRM (Lead Management)
Full lead pipeline from first contact to deal or loss. No lead allowed to exist without a next follow-up date.

**Pipeline stages:**
```
New → Contacted → Interested → Test Drive → Negotiation → Won / Lost
```

**Follow-up enforcement:**
- Every active lead must have a `next_followup_date` set
- Overdue follow-ups flagged in red on dashboard and CRM view
- Full follow-up history: date, notes, outcome — per lead

**Lost lead tracking:**
```
Lost Reason: Price too high / Went to competitor / Not interested / Financing rejected / Other
→ Feeds into Lost Reason Report for pattern analysis
```

---

### Module 5 — Sales & Deal Management
The financial core. Every money movement in the business flows through a Deal.

**Deal status flow:**
```
Draft → Reserved → Active → Completed → (or) Cancelled
```

**Payment system (multi-entry, auto-calculating):**
```
Total Selling Price
− Σ Payment Entries (Cash / Bank Transfer / Cheque / Finance Disbursement)
= Remaining Balance

Payment Status: Unpaid / Partially Paid / Fully Paid  ← auto-calculated
```

**Bank finance tracking:**

| Stage | Status |
|-------|--------|
| Not started | Not Started |
| Application submitted | Submitted |
| Bank approved | Approved |
| Bank rejected | Rejected |
| Amount released | Disbursed → auto-added as payment entry |

**Trade-in handling:**
```
Agreed Trade-in Value is recorded on the deal
Final Payable = Selling Price − Trade-in Value
Trade-in vehicle → manually added to Inventory after handover
```

**Profit calculation (deal level):**
```
Gross Profit = Selling Price − Total Vehicle Cost (from Inventory)
Net Profit   = Gross Profit − Commission − Any Extra Deal Costs
```

**Completion rules:**
- Deal can only be marked Completed when `payment_status = Fully Paid`
- Admin override available — logged to Audit Log
- On completion: vehicle status → `Sold`, sold price saved in Inventory

---

### Module 6 — Customer Management

**Customer types:** Individual / Business / Dealer / Repeat Buyer

**Under each customer profile:**
- Full deal history (all linked deals)
- Vehicle history (all vehicles purchased)
- Payment history (all payments across deals)
- Internal staff notes (with author + timestamp)
- Blacklist status with reason (Admin-only action, logged to Audit)

---

### Module 7 — Employee Management

**Employee profile includes:**
- Role assignment (Admin / Manager / Salesperson / Accountant)
- Permission flags (overrides above role defaults)
- Commission structure (per employee): Fixed / % of Selling Price / % of Profit
- Performance stats: leads assigned, deals closed, conversion rate, total revenue
- Account status: Active / Deactivated (deactivated = cannot log in)

---

### Module 8 — Commission Tracking
Auto-calculated commission on every completed deal. No payroll, no bank integration — tracking only.

```
Commission triggers ONLY when deal.status = COMPLETED
Commission type per employee: Fixed Amount / % of Selling Price / % of Profit
Admin can manually override any commission amount → override logged in Audit Log
```

**Commission status flow:**
```
UNPAID → [Admin clicks Mark Paid] → PAID (paid_at + paid_by recorded)
```

---

### Module 9 — Reports & Analytics

**7 report categories, every report exports in 4 formats: PDF · Excel · CSV · DOCX**

| Category | Reports Included |
|----------|-----------------|
| **Inventory** | Current Stock, Aging/Dead Stock, Profit Potential |
| **CRM** | Lead Source, Pipeline Status, Follow-up Discipline, Lost Reason |
| **Sales** | Sales Summary, Pending Balance, Finance/Loan, Profit (Admin only) |
| **Customers** | Repeat Customers, Purchase History, Top Customers |
| **Employees** | Salesperson Performance, Activity Summary |
| **Commission** | Summary, Unpaid Report, Monthly Breakdown |
| **Expenses** | Expense Summary, Vehicle Expenses, Monthly Trend |

**Analytics dashboard (owner view):**
- KPI cards: Available Stock, Dead Stock, Leads, Conversion Rate, Revenue, Net Profit, Outstanding Balances, Unpaid Commissions
- Charts: Revenue by month, Leads by source, Inventory aging distribution, Top/bottom profit per deal

---

### Module 10 — 3-Layer Trash & Soft Delete
Prevents accidental permanent data loss. Nothing is hard-deleted immediately.

```
Layer 1 — Soft Delete
  User deletes a record
  → is_deleted = true, deleted_at, deleted_by recorded
  → Record disappears from all normal views
  → Moves to Trash Bin

Layer 2 — Admin Permanent Delete
  Admin reviews Trash
  → Restore (returns to original state, exact status preserved)
  → Permanently Delete (moves to Junk Folder, not yet gone)

Layer 3 — 365-Day Auto Purge
  Scheduled backend job runs daily
  → Records in Junk where permanent_deleted_at > 365 days
  → Hard DELETE from database
  → Only true data removal in the system
```

**All deletable tables carry these DB columns:**
```sql
is_deleted            BOOLEAN DEFAULT false
deleted_at            TIMESTAMP NULL
deleted_by            UUID NULL
permanent_deleted_at  TIMESTAMP NULL
permanent_deleted_by  UUID NULL
```

---

### Module 11 — Backup System
Automated, scheduled, no manual action required.

| Schedule | Time | Retention |
|----------|------|-----------|
| Daily Backup | 2:00 AM every day | Last 14 daily backups |
| Weekly Backup | 3:00 AM every Sunday | Last 8 weekly backups |

**What gets backed up:**
- Full database snapshot (all tables, all modules, audit logs, trash metadata)
- All files (vehicle photos, customer documents, sales agreements)

> Critical rule: File backup is mandatory. Database-only restore = incomplete restore.  
> Backups stored in a separate storage location from the main server.

**Access:** Admin only (+ any user explicitly granted `BACKUP_DOWNLOAD` permission)

---

### Module 12 — Audit Log (Immutable)
Tamper-proof, append-only record of every data change in the system.

**Core rules:**
```
APPEND ONLY — no UPDATE or DELETE ever runs on audit_logs table
DB-level permission: app role has INSERT + SELECT only
Even Admin cannot modify or delete audit entries
```

**Every log entry captures:**
```json
{
  "timestamp": "2025-01-15T14:32:00Z",
  "actor_name": "Kamal Perera",
  "module": "sales",
  "action": "update",
  "entity_type": "deal",
  "entity_id": "deal_abc123",
  "message": "Selling price updated on Deal #1042",
  "metadata": {
    "changes": {
      "selling_price": { "from": 9500000, "to": 9200000 }
    }
  }
}
```

**What is logged (by module):** Vehicle edits, price changes, status transitions, every payment entry, every loan status change, deal completion/cancellation, permission changes, login events, every deletion/restore, every backup event, every report export/download.

**What is NOT logged:** Page views, read actions — audit log stays clean and useful, not a noise feed.

---

### Module 13 — Main Dashboard
The landing screen after login. Real-time KPI summary across every system.

| KPI Card | Data |
|----------|------|
| Available Stock | Total vehicles ready to sell |
| Dead Stock (90+ days) | Capital tied up — flagged prominently |
| Leads This Month | Total CRM leads created |
| Conversion Rate | Won leads ÷ Total leads (%) |
| Sales Revenue (Month) | Sum of all completed deal values |
| Net Profit (Month) | Revenue minus costs and commissions |
| Outstanding Balances | Total unpaid across all active deals |
| Unpaid Commissions | Total commission owed to staff |
| Today's Follow-ups | CRM leads due for follow-up today |
| Overdue Follow-ups | Missed follow-ups — flagged |
| Finance Pending | Deals awaiting loan disbursement |

---

## 🛡️ Security Architecture

| Concern | Implementation |
|---------|---------------|
| Authentication | Email + Password + Google Authenticator TOTP (all 3 required) |
| Authorization | DB-stored permission flags, checked server-side on every request |
| Row Level Security | Supabase RLS enabled — DB-side data protection |
| Audit Trail | Immutable append-only log, DB-level INSERT-only permission |
| Data Deletion | 3-layer soft delete — no accidental permanent loss |
| Secrets | All credentials in environment variables — never in codebase |
| File Storage | Files stored in Supabase Storage, only URLs in DB — no binary in Postgres |

---

## 🗄️ Database Design Highlights

- Schema versioning via migrations — tables never created by clicking
- Soft delete columns on all deletable tables
- Commission table with manual override + audit trail
- Audit log table with INSERT-only DB permissions
- Backup history table tracking size, status, and timestamps
- Permission flags stored as DB records, not hardcoded role logic
- All file references stored as URLs/paths — binary data never in Postgres

---

## 🔗 Live Demo

**Demo URL:** [https://dms-web-chi.vercel.app](https://dms-web-chi.vercel.app)

The live demo is a fully functional deployment of the system with seeded dealership data. All 13 modules are accessible using the demo credentials available on the login screen.

> Note: Some destructive Admin actions (backup download, permanent delete) are restricted in the demo environment.

---

## 📬 Source Code Access

The source code for this project is **private** to protect the business logic and architecture that took 6 months to design and build.

If you're a **recruiter, hiring manager, or potential collaborator** who wants to review the actual codebase:

**→ Email:** itschanuka@gmail.com  
**→ LinkedIn:** [linkedin.com/in/chanuka-keerthisingha](https://linkedin.com/in/chanuka-keerthisingha)

Please include your name, company, and what you're evaluating for. I'll grant private repo access within 24 hours.

---

## 👨‍💻 Built By

**Chanuka Keerthisingha** — Full Stack Engineer  
Sri Lanka · Available for remote roles globally

[![Portfolio](https://img.shields.io/badge/Portfolio-itschanuka.vercel.app-0f172a?style=flat-square&logo=vercel&logoColor=white)](https://itschanuka.vercel.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-chanuka--keerthisingha-0a66c2?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/chanuka-keerthisingha)
[![Email](https://img.shields.io/badge/Email-itschanuka%40gmail.com-ea4335?style=flat-square&logo=gmail&logoColor=white)](mailto:itschanuka@gmail.com)

---

<div align="center">
<sub>This repository contains product documentation and architecture overview only. Source code is available on request.</sub>
</div>
