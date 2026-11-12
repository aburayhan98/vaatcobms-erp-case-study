# vaatcobms-erp-case-study
VaatcoBMS project case study and REDME file
> This document has two parts:
>
> **Part I** is the project `README.md`, copied verbatim from the VaatcoBMS repository. Its links are relative to that repository (`Docs/…`, `AGENTS.md`), so they do not resolve here.
>
> **Part II** is an engineering case study: how the practices that top-tier software teams apply show up, concretely, in this codebase — including the numbers, the file paths, the invariants, and the mistakes.

Source repository: <https://github.com/aburayhan98/VaatcoBMS>

---

# Part I — VaatcoBMS Project README

# VaatcoBMS

A comprehensive, enterprise-grade Business Management System (BMS) and Manufacturing ERP built for **VAATCO Bangladesh Limited** — a veterinary and aqua medicine manufacturing and distribution company.

VaatcoBMS is developed with **Clean Architecture** on **ASP.NET Core 8** (C# 12) and **PostgreSQL**, orchestrating end-to-end operations across **5-Level Field Sales Hierarchy & Invoicing (CRM)**, **Value-Anchored Inventory & Manufacturing (MRP / BOM / Bulk Dispensing Tolerance / COGS)**, **Decoupled Sales Order Dispatch & SARD 4.1 Delivery Chalans**, **Carton Packing & Packaging Deduction**, **3-Act Batch-Wise Returns, QA Quarantine, Commercial Credit Notes & Return-to-Vendor (RTV)**, a full **Double-Entry General Ledger Engine**, **Company Debt & Loan Management**, **5-Tab Accounting Reports Hub**, **Fixed Assets & Commercial Leases**, **Smart GPS Attendance, Two-Stage Leave & Payroll Automation**, **Monthly Expense Top Sheets (TA/DA)**, **Dual-Gate Sales Incentives & Leaderboards**, a hardened **Employee Self-Service (ESS) Portal**, and **Production Background Workers**.

---

## Key Modules & Capabilities

```
+-----------------------------------------------------------------------------------------------------------------------------------+
|                                                             VaatcoBMS                                                             |
+-----------------------+------------------------+-------------------------+--------------------------+-----------------------------+
| Sales, CRM & Returns  | Inventory & MRP        | Accounting & Finance    | HR, Payroll & Attendance | Security, RBAC & Operations |
+-----------------------+------------------------+-------------------------+--------------------------+-----------------------------+
| * 5-Level Hierarchy   | * Flat RM/PM/FG Master | * Double-Entry GL (COA) | * Employee 360 Master    | * Refresh Token Rotation    |
| * Field Workspace     | * Value-Anchored Stock | * Single Choke-Pt Post  | * Smart GPS Attendance   | * SecurityStamp Invalidation|
| * Invoice & Batch Trc | * Bulk Scale Tolerance | * Company Debt & Loans  | * LocationIQ Geocoding   | * Subtree Hierarchy RBAC    |
| * FIFO AR Allocations | * Multi-Item Purchases | * 5-Tab Accounting Hub  | * 2-Stage Leave & Missed | * Module Permission Matrix  |
| * Credit Control Gate | * Hybrid BOM & Prod    | * Regular Expense & EMS | * Attendance Auto-Pay    | * PostgreSQL RowVersion Trg |
| * SARD 4.1 Chalans    | * FEFO FG Allocation   | * Fixed Asset Depr.     | * Multi-Tier Salary Pay  | * Nightly AR Self-Healing   |
| * Cash Commissions    | * Carton Packing (5310)| * Lease Rent Sub-Ledger | * TA/DA Expense TopSheet | * HR & Payroll Auto Worker  |
| * 3-Act Batch Returns | * Dynamic Batch COGS   | * Period Lock & Cutover | * Dual-Gate Incentives   | * RFC 7807 + Rate Limiting  |
| * Credit Notes & RTV  | * Waste & Sample Logs  | * Dapper Financial Rpts | * Hardened ESS Portal    | * Startup Fail-Fast Migrate |
+-----------------------+------------------------+-------------------------+--------------------------+-----------------------------+
```

### 1. Field Sales Hierarchy, Invoicing & Receivables (CRM)
- **5-Level Sales Hierarchy & Effective-Dated Assignments** — Configurable 5-tier organizational tree (`ZSM` → `RSM` → `ASM` → `TSM` → `TO/BDO`) with self-referencing territory hierarchy (`ParentTerritoryId`, `LevelId`), effective-dated officer/manager assignments (`SalesAssignment`), TSM dual personal/team working-line support, and subtree-scoped RBAC (`SalesTeamScope`).
- **Role-Based Field Workspace** — Dedicated landing workspace for field officers and managers featuring live receivable summaries, order-to-cash pipeline tabs, commercial product catalogue (restricting raw manufacturing cost masters to back-office), and privacy-safe company-wide leaderboards.
- **Invoice Lifecycle & Batch Traceability** — Complete state machine (`Draft` → `Approved` → `Paid` / `Cancelled`) with automated VAT, trade discounts, officer/manager historical snapshot attribution, and frozen per-line FEFO batch capture (`InvoiceItemBatch`) at every dispatch point.
- **QuestPDF Multi-Copy Invoicing** — Server-side downloadable PDF invoices supporting Customer, Office (with printed `BATCH` column), and Delivery copies with Bangladeshi currency-in-words conversion (`Taka ... and ... Paisa only`).
- **Single-Formula Dealer Due & FIFO Payment Allocation** — Canonical dealer due calculation owned by `CustomerDueCalculator` (`OpeningDue + approved invoices − approved payments − legacy returns − commissions − credit notes`). `PaymentAllocationService` dynamically allocates Cash, Bank, Mobile Banking (MFS), Salary Adjustments, and Discount Adjustments in strict FIFO sequence.
- **Credit Control & Customer Portal** — Pre-invoice validation (`ICreditControlService`) enforcing credit limits and credit term windows (`CreditDays`, `CreditStartDate`, `CreditEndDate`), plus cryptographically secure single-use invitation tokens for the **Customer Self-Service Portal**.
- **Cash Commission (Prompt-Payment Incentive)** — Configurable invoice selection window and commission rate with optional VAT withholding; posts `Dr Sales Discounts Allowed / Cr Customer AR`, updates customer due balance, and supports admin mirror-reversals.

### 2. Value-Anchored Inventory, Manufacturing (MRP / BOM / COGS) & Tolerance
- **Value-Anchored Inventory Invariant** — `MaterialInventory.CarryingValue` is the authoritative source of truth anchored to General Ledger control accounts (`1300` Raw Material, `1400` Packaging Material, `1500` Finished Goods). Unit rates (`CarryingValue ÷ CurrentStock`) are display-only and never stored; every stock movement moves `CarryingValue` by the exact GL amount.
- **Bulk Material Dispensing-Scale Tolerance Engine** — Handles physical dispensing-scale variance on bulk RM/PM (`MaterialToleranceAdjustment` & `TolerancePolicySetting`) with configurable tolerance thresholds, automatic in-band posting, manager approval gates for out-of-band variances, and immutable vendor AP ledger preservation.
- **Multi-Line Material Purchasing** — Unified purchase engine (`MaterialPurchase` & `MaterialPurchaseItem`) for Raw and Packaging Materials with subtotal/tax/freight calculation, lot & expiry tracking, approval workflows, mirror-reversal audit trails, and clickable links to vendor AP ledger accounts.
- **Append-Only Inventory Ledger & Concurrency Safety Net** — Chronological `InventoryTransaction` audit trail with running balance tracking, expired batch blocking, and database-enforced `RowVersion` optimistic concurrency tokens (returning RFC 7807 HTTP 409 Conflict on race conditions).
- **Hybrid Bill of Materials (BOM) & Production Depletion** — Dynamic recipe scaling with high-precision component quantities, two-step `ProductionLog` confirmation, automatic RM/PM stock and carrying-value depletion, FG batch creation, and dynamic batch **COGS** calculation.
- **FEFO Finished Goods Allocation** — `FinishedGoodsInventory` with batch numbers, manufacturing/expiry dates, and automated **FEFO** (First-Expired, First-Out) stock allocation via `FgStockAllocator`.
- **Waste & Sample Routing** — Physical stock adjustment, multi-material waste write-off previews, and sample distribution workflows with mandatory reason logging and automatic GL postings.

### 3. Decoupled Sales Order Dispatch, Carton Packing & SARD 4.1 Chalans
- **Decoupled 4-Stage Lifecycle** — Separation between Order Booking (`SalesOrder`), Physical Dispatch (Delivery Order / Chalan with FEFO batch lock), Financial Invoicing (`IsConvertedToInvoice`), and Payment Settlement.
- **Read-Only Field Role Policy After Booking** — Officers and Managers retain stock-availability hints during order booking but are strictly read-only post-booking (no packing cost visibility, customer-copy printing only).
- **Carton Packing Management** — Physical packing carton allocation (`SalesOrderPackingLine`) with stock readiness indicators, automatic packaging stock deduction on dispatch, and automatic posting to `Account 5310 Carton Packing Expense` upon invoice approval.
- **SARD 4.1 Compliant Chalans** — QuestPDF delivery chalans featuring 4-party authorization signature blocks (Customer, Preparer, Delivery Person, Authorized Signatory).

### 4. 3-Act Batch-Wise Returns, Commercial Credit Notes & Return-to-Vendor (RTV)
- **Act 1 — Gate Intake & Batch Lookup (`ReturnConsignment`)** — Batch-first return intake capturing returned quantities against original dispatched FEFO batches (`BatchLookupService`), with LIFO auto-split fallback and gate batch correction.
- **Act 2 — QA Quarantine & Multi-Disposition (`ReturnItemDisposition`)** — Technical inspection routing returned stock across restock, decanting/reprocessing, or scrap/write-off with database `CHECK` constraints (`CK_ReturnItemDisposition_RecoveryXor`) and batch hold/recall enforcement.
- **Act 3 — LIFO Waterfall & Commercial Credit Notes (`CreditNote`)** — Automated LIFO multi-invoice valuation waterfall (`BatchWaterfallService`) generating immutable commercial credit notes (`Dr Sales Returns / Cr Customer AR` + inventory recovery legs) that reduce dealer due balances.
- **Return-to-Vendor (`VendorDebitNote`)** — Commercial vendor debit notes for defective or short-supplied RM/PM lots with XOR material check constraints and AP sub-ledger integration.

### 5. Double-Entry Accounting, Company Loans & 5-Tab Reports Hub
- **5-Tier Chart of Accounts (COA)** — Hierarchical account tree (`Assets`, `Liabilities`, `Equity`, `Revenue`, `Expenses`) with automated runtime sub-account provisioning (`1200-C{id}` Customer AR, `2000-V{id}` Vendor AP, `2320-E{id}` Employee Expense Payable, `2330` Sales Incentive Payable, `4100-D{id}` Customer Discounts, `5310` Carton Packing Expense).
- **Single Choke-Point Append-Only Ledger** — Every journal entry passes through `IJournalPostingService` (enforcing balanced Dr/Cr, ≥ 2 lines, active accounts, and open accounting periods). `AppDbContext.SaveChanges` blocks any mutation or deletion of posted journals; all corrections are mirror reversals (`ReversedEntryId`).
- **Company Debt & Loan Management (`CompanyLoan`)** — Full lifecycle management for **Bank EMI Loans**, **Fixed Company Bullet Loans**, and **Personal/Director Loans** with running interest accruals (`LoanInterestAccrual`), rate revisions (`LoanRateRevision`), disbursements/repayments (`LoanPaymentDisbursement`), historical cutover onboarding, and guarded contract voiding/reversals.
- **5-Tab Accounting Reports Hub (`/ReportsHub/AccountingHub`)** — Centralized Dapper-powered financial hub with PDF & CSV exports across 5 views:
  1. **Received Report** (Cash/Bank/MFS inflows)
  2. **Payment Report** (Operational, vendor, salary & loan outflows)
  3. **Loan & Advance Report** (Corporate borrowings & employee advances)
  4. **Receivable Balance-Only Report** (Dealer AR balances)
  5. **Payable Balance-Only Report** (Vendor AP & obligation balances)
- **Regular Expense Register & Employee Reimbursements (`EXP-*` / `EMS-*`)** — Day-by-day operating expense register with Finance-maintained category/sub-category → GL mappings, *Spent By* employee attribution, on-demand `2320-E{id}` out-of-pocket payable tracking, GL-derived Employee Dues settlement (`EMS-{yyyy}-{seq}`), mirror reversals, and printable QuestPDF vouchers.
- **Fixed Assets, Commercial Leases & Period Locking** — Straight-line fixed asset depreciation (`FixedAsset`), commercial lease & factory rent accruals (`LeaseAgreement`), guided COA opening balances (`OpeningBalance`), and fiscal period lock gates (`AccountingPeriod`).
- **Core Financial Statements (Dapper CQRS)** — Trial Balance, Multi-Tier Income Statement (P&L), Balance Sheet with Retained Earnings rollup, Account Ledgers, Gross Profit analysis, Debtor Aging, and Itemized Product-Wise Customer & Vendor Ledgers.

### 6. Human Resources (HR), Smart GPS Attendance, Payroll & Dual-Gate Incentives
- **Employee 360 Master & Linked Identity** — Executive profiles with independent NID (10/13/17 digits) & TIN (12 digits) validation, salary templates, breakdown settings, and a database-enforced 1:1 `LinkedUserId` filtered unique index (`IX_Employees_LinkedUserId`).
- **Smart GPS Attendance & Reverse Geocoding (`AttendanceRecord`)** — Mobile-friendly self-service GPS check-in and check-out with **LocationIQ reverse geocoding** (with strict timeout & non-blocking fallback), direct Google Maps coordinate links, daily work report check-out comments, admin overrides, and configurable `AttendancePolicySetting` (grace minutes, standard hours, weekly off-days, auto-checkout cutoff).
- **Two-Stage Hierarchy Approvals (Leave & Missed Check-In)** — Multi-stage approval pipeline resolved via `IHrHierarchyApprovalResolver` (`Pending` → `Forwarded` by immediate supervisor → `Approved`/`Rejected` by HR/Admin) for both `LeaveRequest` and `MissedCheckInRequest`, with live pending-approval badges and admin-only leave policies (`IsAdminOnly`).
- **Attendance-Driven Payroll Engine (`SalarySheet`)** — Monthly salary sheet generation factoring in attendance/leave records, proration, bonuses, deductions, multi-tier partial installment payments (`SalaryPayment`), 1-click cascade GL reversals, department-filtered print views, and compact single-page A4 QuestPDF payslips.
- **Monthly Expense Top Sheets (TA/DA Engine)** — Field conveyance and TA/DA submission lifecycle (`Draft` → `Submitted` → `Verified` → `Approved` → `Partially/Fully Disbursed`), sharing the unified `IEmployeeObligation` contract with Payroll.
- **Dual-Gate Sales Incentives & Leaderboard (`SalesIncentive`)** — Monthly eligibility requiring **both** Sales Actual/Target and Collection Actual/Target to reach the policy threshold (default 99%). Freezes immutable monthly snapshots (`OfficerMonthlyAchievement`), issues numbered grants (`INC-{yyyyMM}-{####}`, `Dr 6100 / Cr 2330`), and renders a company-wide performance leaderboard with service-level bonus privacy for non-admin roles.
- **Employee Self-Service (ESS) Portal & HR Analytics** — Hardened self-service portal (`/Ess`) for GPS attendance, leave applications, TA/DA expense sheets, and self-scoped payslip PDF downloads, paired with an **ApexCharts** HR Analytics Dashboard.

### 7. Security, Identity & Production Background Workers
- **JWT + HttpOnly Cookies & Refresh Token Rotation (RTR)** — Algorithm-pinned (`HS256`) multi-key JWT authentication with SHA-256 hashed refresh token rotation, token-family reuse detection, and persistent "Remember Me" session policies.
- **Instant Session Invalidation & Rate Limiting** — 60-second cached `SecurityStamp` validation revoking active sessions on role/password/approval changes, paired with ASP.NET Core IP rate limiting (`auth-login`, `auth-sensitive`, and global partitions).
- **`DueBalanceReconciliationWorker`** — Nightly background service (02:00 AM) auditing all customer `DueBalance` fields against the canonical `CustomerDueCalculator` ledger formula and self-healing any drift.
- **`HrPayrollAndAttendanceAutomationHostedService`** — Background automation worker executing nightly auto-checkouts for open attendance logs and scheduled monthly attendance-driven payroll accruals.

---

## Tech Stack

| Layer / Concern | Technology | Details |
|---|---|---|
| **Backend Framework** | ASP.NET Core 8 (C# 12) | Hybrid MVC + REST API in Clean Architecture |
| **ORM & Persistence** | Entity Framework Core 8 + Npgsql | Code-First Migrations, **84 Domain Tables**, `dbo` schema, `trg_set_rowversion` triggers |
| **High-Performance Read CQRS** | Dapper | Parameterized read-side queries for Accounting, Hub Reports, Ledgers & Territory Analytics |
| **Database** | PostgreSQL (Prod/Dev) & SQL Server LocalDB (Integration Tests) | Startup fail-fast migration gate (`StartupDatabaseMigrator`) |
| **Authentication & Security** | JWT Bearer + `HttpOnly` Cookies | Multi-Key Rotation, Refresh Token Rotation (RTR), `SecurityStamp` cache, RBAC + Subtree Scope |
| **Reverse Geocoding** | LocationIQ HTTP Client | Resilient GPS coordinate → address resolution with non-blocking fallback |
| **PDF & Excel Generation** | QuestPDF & NPOI | Multi-copy invoices, SARD 4.1 chalans, EXP/EMS vouchers, payslips, financial statements & Excel exports |
| **Mapping & Validation** | Mapster & FluentValidation | Compile/runtime DTO mapping and automatic request validation |
| **Logging & Diagnostics** | Serilog + HealthChecks | Structured rolling logs, RFC 7807 `ProblemDetails`, and `/health` container endpoint |
| **Frontend UI** | Razor Views + Bootstrap 5.3.3 | Custom design system (`vcard`, `vtable`, `vmodal`), ApexCharts, responsive 767.98px stacked mobile cards, zero build step |
| **Testing Suite** | xUnit + Moq + FluentAssertions | Split fast Unit Tests (`VaatcoBMS.Tests`) & real DB Integration Tests (`VaatcoBMS.IntegrationTests`) |

---

## Project Architecture

The solution strictly adheres to **Clean Architecture** across 4 core projects and 2 test projects:

```
VaatcoBMS.sln
+-- VaatcoBMS.Domain/                  # Pure C# domain model (84 entities, enums, domain rules, repository interfaces)
|   +-- Common/                        # FgStockAllocator (FEFO logic), PagedResult, QueryParams
|   +-- Entities/                      # Core, Accounting/, HR/, Inventory/, Sales/ entities
|   +-- Enums/                         # Strongly-typed domain state machines & source module discriminators
|   +-- Interfaces/                    # Repository contracts & IUnitOfWork
|
+-- VaatcoBMS.Application/             # Framework-independent use-case orchestration & business rules
|   +-- Common/                        # CustomerDueCalculator (single canonical dealer-due formula)
|   +-- DTOs/                          # Request/response contracts across Accounting, HR, Inventory, Returns, Sales
|   +-- Interfaces/                    # Application service & query contracts
|   +-- Mappings/                      # Mapster TypeAdapter configurations
|   +-- Services/                      # Domain services (Accounting, HR, Inventory, Returns, Sales)
|   +-- Validators/                    # FluentValidation rules
|
+-- VaatcoBMS.Infrastructure/          # Data access, external integrations, and background automation
|   +-- Migrations/                    # EF Core Npgsql Code-First migrations
|   +-- Persistence/                   # AppDbContext, EntityTypeConfigurations, StartupDatabaseMigrator
|   +-- Repositories/                  # EF Core repositories + UnitOfWork
|   +-- Services/                      # Dapper Query Services, QuestPDF & NPOI Exporters, LocationIQ Geocoding,
|   |                                  # HrPayrollAndAttendanceAutomationHostedService, Email & Auth services
|   +-- Utility/                       # TokenBuilder (JWT), HashService (BCrypt)
|
+-- VaatcoBMS.Web/                     # ASP.NET Core MVC + REST API Presentation Layer
|   +-- BackgroundWorkers/             # DueBalanceReconciliationWorker
|   +-- Controllers/                   # 46 MVC Controllers + 75 REST API Controllers (/api/*)
|   +-- Middleware/                    # ExceptionHandlingMiddleware (RFC 7807), RequestLoggingMiddleware
|   +-- Security/                      # SecurityStampCache, Portal & Hierarchy authorization filters
|   +-- Views/                         # Modular Razor Views + Shared Design System
|
+-- VaatcoBMS.Tests/                   # Fast isolated Unit Test suite (~10s execution)
+-- VaatcoBMS.IntegrationTests/        # End-to-end Database Integration Tests (~30s execution)
```

---

## Getting Started

### Prerequisites
- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- **PostgreSQL 15+** (for running the application)
- **SQL Server LocalDB** (optional, used by `VaatcoBMS.IntegrationTests`)

### Setup & Installation

1. **Clone the Repository**
   ```bash
   git clone https://github.com/aburayhan98/VaatcoBMS.git
   cd VaatcoBMS
   ```

2. **Configure User Secrets (Development)**
   See [`Docs/operations/Deployment_Preflight.md`](Docs/operations/Deployment_Preflight.md) for the full pre-flight checklist.
   ```powershell
   cd VaatcoBMS.Web
   dotnet user-secrets set "ConnectionStrings:VaatcoBMSDB" "Host=localhost;Port=5432;Database=VaatcoBMS;Username=postgres;Password=your-password;Search Path=dbo,public"
   dotnet user-secrets set "JwtSettings:Secret" "your-super-secret-key-at-least-32-characters-long"
   dotnet user-secrets set "JwtSettings:Issuer" "VaatcoBMS_API"
   dotnet user-secrets set "JwtSettings:Audience" "VaatcoBMS_Client"
   dotnet user-secrets set "SmtpSettings:Server" "smtp-relay.brevo.com"
   dotnet user-secrets set "SmtpSettings:Port" "587"
   dotnet user-secrets set "SmtpSettings:SenderEmail" "admin@vaatco.com"
   dotnet user-secrets set "SmtpSettings:SenderName" "VAATCO BMS"
   cd ..
   ```

3. **Build & Run the Application**
   Pending EF Core migrations are applied automatically on startup via `StartupDatabaseMigrator` (fail-fast gate).
   ```bash
   dotnet build VaatcoBMS.sln
   dotnet run --project VaatcoBMS.Web
   ```
   - **Web App UI:** `https://localhost:7169`
   - **Swagger API UI:** `https://localhost:7169/swagger`
   - **Health Check:** `https://localhost:7169/health`

---

## Test Suite & Verification

```bash
# 1. Run fast unit tests (~10s)
dotnet test VaatcoBMS.Tests

# 2. Run database integration tests (~30s, requires SQL Server LocalDB)
dotnet test VaatcoBMS.IntegrationTests

# 3. Filter a specific test suite while iterating
dotnet test VaatcoBMS.Tests --filter "FullyQualifiedName~CompanyLoan"
```

---

## Core Engineering Invariants

1. **Immutable Vendor & General Ledger** — A posted purchase, bill, journal, loan event, or AP entry is never edited in place. All corrections are **mirror reversals** (`ReversedEntryId`).
2. **Value-Anchored Inventory (`CarryingValue`)** — `MaterialInventory.CarryingValue` is the source of truth; `Σ CarryingValue` across items always equals the GL inventory control balances (`1300` RM, `1400` PM, `1500` FG) across `MaterialPurchaseService`, `ProductionService`, `SalesOrderService`, `WasteService`, `SampleService`, and `MaterialToleranceService`.
3. **Single Canonical Dealer Due Formula** — Owned exclusively by `Application/Common/CustomerDueCalculator`. Every screen, aging report, territory rollup, and nightly reconciliation worker calls the same formula.
4. **Database-Generated `RowVersion` Concurrency** — Every concurrency-guarded table in PostgreSQL carries a `trg_set_rowversion` trigger (`BEFORE INSERT OR UPDATE` → `int8send(txid_current())`) paired with store-generated EF Core token configuration.

---

## Project Documentation Index

Comprehensive architectural plans, implementation tickets, SQL reconciliation scripts, and operational SOPs live in [`Docs/`](Docs/README.md):

| Domain Area | Directory | Key Highlights |
|---|---|---|
| **Master Documentation Index** | [`Docs/README.md`](Docs/README.md) | Complete index of all specifications, runbooks, and ticket plans |
| **Engineering Rules & Schema** | [`AGENTS.md`](AGENTS.md) · [`Docs/Database-Schema-ER.md`](Docs/Database-Schema-ER.md) | Architectural guardrails, accounting invariants, Mermaid ER diagrams & DBML |
| **Accounting, Loans & Hub** | [`Docs/accounting/`](Docs/accounting/) | [Accounting Reports Hub](Docs/accounting/Accounting_Reports_Hub_Implementation_Tickets.md), [Company Debt & Loans](Docs/accounting/Company_Debt_And_Loan_Management_Specification.md), [Regular Expenses](Docs/accounting/Regular_Expense_Implementation_Tickets.md), [Opening Balance Runbook](Docs/accounting/OpeningBalance_Runbook.md) |
| **Receivables & Credit** | [`Docs/receivables/`](Docs/receivables/) | [Receivables Plan](Docs/receivables/Receivables_Plan.md), [Rollout Runbook](Docs/receivables/Receivables_Rollout_Runbook.md), [Dealer Due SQL Reconciliation](Docs/receivables/Dealer_Due_Reconciliation.sql) |
| **Sales, Hierarchy & CRM** | [`Docs/sales/`](Docs/sales/) | [5-Level Hierarchy & RBAC](Docs/sales/Sales_Hierarchy_RBAC_Targets_Leaderboard_Implementation_Tickets.md), [Territory Reports](Docs/sales/Territory_Report_Implementation_Tickets.md), [Cash Commissions](Docs/sales/Cash_Commission_Implementation_Tickets.md), [DO Read-Only Policy](Docs/sales/DeliveryOrder_ReadOnly_Role_Policy.md) |
| **Inventory, MRP & Tolerance** | [`Docs/inventory/`](Docs/inventory/) | [Tolerance Handoff Brief](Docs/inventory/Tolerance_Programme_Handoff.md), [Bulk Dispensing Tolerance Spec](Docs/inventory/Bulk_Material_Tolerance_And_Accounting_Workflow.md), [Delivery Orders & Cartons](Docs/inventory/DeliveryOrder_PackingCarton_Implementation_Tickets.md), [SARD 4.1 Spec](Docs/inventory/VAATCO_SARD_mod.md) |
| **Returns, Credit Notes & RTV** | [`Docs/returns/`](Docs/returns/) | [Returns Architecture v2.0](Docs/returns/VaatcoBMS_Returns_And_Credit_Notes_Architecture_Specification_v2.0.md), [Batch-Wise Returns](Docs/returns/BatchWise_Returns_Implementation_Tickets.md), [Returns Handoff](Docs/returns/Returns_Programme_Handoff.md) |
| **HR, Payroll, Attendance & ESS** | [`Docs/hr/`](Docs/hr/) | [Employee 360 & Top Sheets](Docs/hr/HR_Payroll_EmployeeManagement_Implementation_Tickets.md), [Payroll Engine](Docs/hr/Payroll_Engine_Implementation_Tickets.md), [Sales Incentives](Docs/hr/Sales_Incentive_Implementation_Tickets.md), [ESS Linked Identity](Docs/hr/ESS_Linked_Identity_Handoff.md) |
| **Operations, Security & Migration** | [`Docs/operations/`](Docs/operations/) · [`Docs/security/`](Docs/security/) · [`Docs/migration/`](Docs/migration/) | [Deployment Preflight](Docs/operations/Deployment_Preflight.md), [Error Correction SOPs](Docs/operations/Operational_Error_Correction_SOPs.md), [Security & Key Rotation](Docs/security/SECURITY.md), [PostgreSQL Migration](Docs/migration/PostgreSQL_Migration_Implementation_Tickets.md) |

---

## Author & Maintainer

Built for **VAATCO Bangladesh Limited** by [@aburayhan98](https://github.com/aburayhan98).

---
---

# Part II — Engineering Case Study

## How this was measured

`AGENTS.md` notes that `VaatcoBMS.Tests/InfrastructureTests/**` is outside the documented baseline and has never run successfully. Running the full suite confirms exactly that: **2108 pass, 1 fails, and the only failure is inside that excluded folder.** Excluding it, the suite is green. That detail matters for this case study — see Practice 6.

| Dimension | Measured |
| :--- | :--- |
| Commits | 1,001 |
| First → latest commit | 2026-04-05 → 2026-11-11 |
| Contributors | 3 |
| Production C# files (excl. migrations) | 1,121 |
| Production LOC (excl. migrations) | ~89,024 |
| Test files / test LOC | 253 / ~56,607 |
| Unit tests passing (documented baseline) | 2,108 |
| Domain entity files / enum files | 89 / 63 |
| `DbSet`s on `AppDbContext` | 86 |
| EF Core migrations | 23 net (50 files incl. snapshot & designer) |
| MVC controllers | 54 |
| REST API controllers | 70 |
| Razor views | 149 |
| Documentation markdown files | 127 |
| Remote branches | 22 |

The ratio worth pausing on: **~56.6k lines of test code against ~89k lines of production code.** That is a 0.64 test-to-production ratio with 2,108 assertions executing in about 15 seconds. That did not happen by asking for tests at the end.

---

## The domain problem

VAATCO Bangladesh manufactures and distributes veterinary and aqua medicine. That single sentence generates most of the constraints in this system:

- **Batch and expiry control is regulatory**, so every dispatched unit must be traceable to the batch it came from. This is why FEFO allocation and `InvoiceItemBatch` exist.
- **A veterinary medicine manufacturer in Bangladesh has a field sales force**, so the system needs a 5-level hierarchy, territory-scoped visibility, and commissions — but the officers must never see manufacturing cost.
- **Dispensing is physical.** Bulk raw material is weighed on a scale that rarely matches the system quantity. Moisture, tare, cling, and spillage are normal.
- **The vendor ledger is a legal record.** If a scale reading disagrees with a purchase, the *purchase must not change*.

That last constraint is where most ERP projects quietly fail, and where this one has its best story.

---

## Practice 1 — Write the invariant down before writing the feature

Junior engineering treats correctness as "the tests pass." Senior engineering treats correctness as **a small set of statements that must never be false**, and then enforces them at every entry point.

VaatcoBMS declares four. They are in `AGENTS.md`, in `README.md`, and — critically — enforced in code:

1. **The vendor and purchase ledger is immutable.** A purchase, bill, PO, GRN, or AP entry is never edited to absorb a discrepancy. Corrections are *mirror reversals*, never compensating adjustments.
2. **Inventory value is anchored, not recomputed.** `MaterialInventory.CarryingValue` is the source of truth. A derived rate (`CarryingValue ÷ CurrentStock`) is display-only and **never stored**.
3. **Σ `CarryingValue` must equal the GL inventory control balance** (`1300` RM, `1400` PM, `1500` FG) after *every* operation, including intermediate states.
4. **Dealer due has exactly one formula**, owned by `CustomerDueCalculator`.

What makes this a mature practice rather than a slogan is that each invariant names the code that can violate it. Invariant 2 lists `MaterialPurchaseService`, `ProductionService`, `SalesOrderService`, `WasteService`, `SampleService` — because those are the only paths that move stock. Invariant 3 is stated as an equation that can be checked with one SQL query.

### Transferable lesson

An invariant is not documentation. It is a **list of every code path that can break it**, plus the query that proves it hasn't. If you cannot name the violating paths, you have a wish, not an invariant.

---

## Practice 2 — One formula, one owner, and a named incident

The comment at the top of `Application/Common/CustomerDueCalculator.cs` is the most instructive line of documentation in the repository:

```
/// Single owner of the dealer-due formula. Every reader — the nightly due rebuild, the
/// Territory Sales ClosingDue, and the assigned-customer as-of view — must agree on this,
/// or the screens drift apart again (the ৳13,412.50 vs ৳9,000.00 dealer incident).
```

Top-tier teams do three things here that weaker teams skip:

1. **They collapse duplicate logic into one pure function.** `ComputeDue` takes six decimals and returns one. It has no `DbContext`, no `DateTime.Now`, no `IHttpContextAccessor`. It is trivially testable.
2. **They distinguish arithmetic from policy.** The XML comment explicitly says: *"Callers own their date cutoffs and status filters — this type only owns the arithmetic and the condition rules that define which credits count."* That separation is why the same function serves a live screen, an as-of view, and a nightly worker.
3. **They record the incident that caused the abstraction.** The ৳13,412.50 vs ৳9,000.00 discrepancy is in the source code, in the README, and in `AGENTS.md`. A future maintainer who is tempted to "just compute it here, it's faster" has to argue with a real number that reached a real dealer.

### Transferable lesson

Two implementations of a financial formula are not redundancy — they are a scheduled defect. And the exception message or code comment that says *why* an abstraction exists outlives any amount of tribal knowledge.

---

## Practice 3 — Make the wrong thing impossible, not discouraged

Most codebases enforce accounting rules with code review. This one enforces them in `SaveChanges`, where no controller, service, or future developer can route around them.

`AppDbContext.EnforceJournalImmutability()` walks the change tracker on every save, and buffers two specific violations:

```csharp
if (entry.State == EntityState.Deleted)
    throw new InvalidOperationException(
        $"Journal entries are immutable — cannot delete {entry.Entity.GetType().Name}. " +
        "Create a reversal entry instead.");
```

and for modifications, it compares original against current value across every property, allowing only a narrow allowlist:

```csharp
var allowedProps = new HashSet<string> {
    nameof(JournalEntry.IsReversed),
    nameof(JournalEntry.UpdatedAt),
    nameof(JournalEntry.UpdatedBy) };
```

That allowlist is the detail that separates a real implementation from a gesture. The team discovered that a reversal must be able to flip `IsReversed` on the original entry — so they permitted lifecycle metadata while keeping **financial amounts and journal lines 100% immutable**. They did not abandon the rule when it collided with reality; they carved out the smallest possible exception and commented why.

The same philosophy governs posting. Every journal passes through a single choke point, `IJournalPostingService`, which validates balance, line count, active accounts, and — before anything touches the change tracker — period locks:

```csharp
// Period Locking Guard — absolute hard check before touching anything else
if (isPeriodLocked)
    throw new InvalidOperationException(
        $"Cannot post transaction. The accounting period for {entry.EntryDate:MMMM yyyy} is locked and sealed for audit.");
```

Note the ordering comment: *"Everything is validated BEFORE the entry touches the change tracker."* Validating after attaching to the tracker leaves a half-built entity in memory if the validation throws — a subtle source of corrupted subsequent operations.

### Transferable lesson

If a rule matters, put it where bypassing it requires editing infrastructure, not where bypassing it requires forgetting a review comment. "Impossible" beats "forbidden."

---

## Practice 4 — Respect the database you actually have

VaatcoBMS migrates from SQL Server to PostgreSQL. SQL Server has `rowversion`. PostgreSQL does not. The naive fix — seed a GUID in application code — produces a bug that only appears on the *second save in the same unit of work*.

The team's fix is a trigger, in `20260930125237_RowVersion_TriggerEmulation.cs`:

```sql
CREATE OR REPLACE FUNCTION dbo.set_rowversion()
RETURNS trigger LANGUAGE plpgsql AS $function$
BEGIN
    NEW."RowVersion" := int8send(txid_current());
    RETURN NEW;
END;
$function$;
```

`txid_current()` is the 8-byte transaction id — **one value per transaction, exactly like SQL Server**. Then, for each of 20 token tables, the migration applies a default, drops any existing trigger, and attaches `trg_set_rowversion` `BEFORE INSERT OR UPDATE`.

The migration's XML comment is a small masterclass in post-incident writing. It documents the *pairing* requirement:

> `AppDbContext` declares the token store-generated on Npgsql so EF reads the trigger's value back via `RETURNING`; without that pairing the change tracker keeps the app-seeded value and the next save in the same unit of work fails its `WHERE "RowVersion" = @original` predicate ("Stock changed by another user" incident, 2026-09-30).

Three things are happening: the emulation matches the *semantics* of the original (one per transaction, not one per row), the trigger and the EF configuration are documented as an inseparable pair, and the incident that taught them is dated and named.

`AGENTS.md` then generalises the rule so nobody re-learns it:

> Never hand-seed or compare it in application code, and never configure a new `byte[]` token without that pairing — otherwise the second save in a unit of work matches zero rows.

### Transferable lesson

When you emulate a platform feature, match its **semantics**, not its shape. An 8-byte value that changes per row is a different feature from an 8-byte value that changes per transaction. And write the incident into the migration, because migrations are what the next person reads.

---

## Practice 5 — Fail closed, and let Finance own the numbers

The bulk-dispensing tolerance programme is the clearest example of senior judgement in the codebase, because the interesting part is what happens when the answer is *unknown*.

The policy has three gates. Measurement basis: measured units (kg/L/ml) get a percentage; counted units (`Pcs`) get **0%**, because a difference on a counted item is a miscount, not a tolerance. Quantity gate: a percentage. Value gate: a currency cap, seeded at ৳500.

The design decisions that matter:

- **A cap of `0` means nothing is auto-absorbed.** A missing policy row also fails closed. The system does not guess.
- **An out-of-policy reading moves *nothing*** — no quantity, no carrying value, no ledger row — until a Manager approves. It is held as a *proposal*.
- **While held, the material is frozen** for production, waste, sample, and dispatch packing. A `PendingToleranceGuard` refuses it by name at every consumption path.
- **One open reading per material**, because two proposals on the same stock would double-count.
- **The recorder can never settle their own reading** — with a documented, dated amendment when the owner overrode that rule. The exception is recorded, not silently applied.

And critically, `AGENTS.md` pins the ownership boundary in a way that prevents an entire category of future bug:

> Both the percentage and the cap are Finance-owned and edited in the app (Admin/SuperAdmin), **never in `appsettings.json`**.

An engineer's instinct is to put a tunable number in config. Here that instinct is explicitly forbidden, because a tolerance threshold is an *accounting policy*, and accounting policies require an audit trail (`TOLERANCE_POLICY_CHANGED`), a permission model, and instant application without a restart. Config files have none of those.

### Transferable lesson

"Fail closed" is a decision you make in advance, and it is almost always right for money. The corollary: classify every tunable number. A performance knob belongs in config; a financial policy belongs in the database with an audit trail and a permission gate.

---

## Practice 6 — Be honest about the test baseline

This is the practice most teams get wrong, and it's the one I'd hold up first in a review.

Running the full suite produces **2,108 passing and 1 failing**. The failure is `TerritorySalesReportQueryServiceTests.GetSummaryAsync_ClosingDue_ShouldIncludeAllCreditLegs_LikeTheDealerLedger`, which expects `8900M` and gets `8600M`.

A weak team does one of two things: deletes the failing test, or "fixes" it by changing the expected value. VaatcoBMS does neither. `AGENTS.md` **declares the exclusion explicitly**:

> `VaatcoBMS.Tests/InfrastructureTests/**` is excluded from the documented baseline and has never run successfully; do not read its failure as a regression, and do not "fix" it as a side task.

That is a mature move: the baseline is *defined*, so a red result outside it cannot masquerade as a new regression, and a green result inside it cannot be faked. The project also documents its real verification command:

```bash
dotnet test VaatcoBMS.Tests --filter "FullyQualifiedName!~InfrastructureTests"
```

Separately, the tolerance handoff brief documents its own test posture as *"922/922 unit · 63/63 integration"* — a snapshot from that programme, not a claim about today's build. The distinction between "the last time someone verified this area" and "the current state of the whole suite" is one most documentation blurs.

### Transferable lesson

A test baseline is a claim, and claims need scope. "The suite is green" is meaningless without saying *which* suite. Documenting a known-failing folder is stronger engineering than hiding it, because it converts an unknown into a bounded, non-alarming fact.

---

## Practice 7 — Documentation as an engineering artifact, not a courtesy

127 markdown files in `Docs/`, organised by domain, with a master index. But the count is not the signal. The *kinds* of documents are:

- **Ticket plans with stable IDs** (`ARH-00`…`ARH-12`, `LN-01`…`LN-25`, `RTB-01`…`RTB-12`, `INC-D1`…`INC-DOC1`, `ARR-01`…`ARR-16`). Stable IDs mean a commit can cite a ticket, and a reviewer can check scope against the plan instead of against memory.
- **Locked Decisions** (`D1`–`D10`) with **revision history**. Decision D7 was amended on 2026-09-14, and the amendment is visible rather than overwritten. D5 was amended too. You can see that the rule changed and why.
- **Session handoff briefs** (`Tolerance_Programme_Handoff.md`, `Returns_Programme_Handoff.md`, `ESS_Linked_Identity_Handoff.md`). These exist because a fresh session — human or AI — otherwise re-derives, and re-deriving re-introduces errors. The brief opens with *"Read this first in a new session"* and states the live database state, not the intended state.
- **SQL scripts with a defined lifecycle**: dry-run → guarded transactional apply → verification → residual report. `BatchWise_Returns_Backfill.sql` and `Tolerance_Historical_Reclassification.sql` both follow it. A data migration that can't be dry-run and can't be verified is not a migration, it's a hope.
- **Runbooks and SOPs** for the unglamorous parts: `Deployment_Preflight.md`, `OpeningBalance_Runbook.md`, `Operational_Error_Correction_SOPs.md`.

The handoff brief also contains a section titled **"Guardrails — what a fresh agent gets wrong"**, listing the exact mistakes that were made before: touching the vendor ledger, reintroducing a recorder ≠ approver check, moving policy values back into `appsettings.json`. That is institutional memory with a file path.

### Transferable lesson

Document the decision and the *reversal of* the decision. A plan that only reflects the current state teaches nothing; a plan with Rev 1–9 shows a maintainer which paths were already tried and abandoned.

---

## Practice 8 — Delete the old path, and treat its 404 as correct

The tolerance programme replaced an "adjust stock" screen that changed quantity *and* value with no policy, no approval, and no audit trail.

The new code did not deprecate it. It deleted it, and then wrote the deletion into the handoff brief:

> **Deleted on purpose:** `AdjustStockAsync` / `AdjustMaterialAsync`, `POST /stock/adjust`, both DTOs + validator, `StockAdjustmentTests`, `Views/InventoryDashboard/Adjust.cshtml` + its sidebar link. **A 404 there is correct, not a bug.**

Three things are noteworthy. The deletion covered **the service, the endpoint, the DTOs, the validators, the tests, the view, and the navigation link** — the whole vertical slice, so no orphan can be re-wired later. The removal is **documented as intentional**, because an undocumented deletion looks identical to an accident. And it is called out precisely because the first instinct of a returning developer seeing a 404 is to file a bug.

This pairs with the `AGENTS.md` line: *"Don't use feature flags or backwards-compatibility shims when you can just change the code."* Two code paths that both write inventory is not backwards compatibility, it is two behaviours pretending to be one.

### Transferable lesson

A legacy write path for money is a live defect, not a convenience. Delete the full slice and leave a note where the next person will look for it.

---

## The defect that mattered

The most useful artefact for anyone learning from this codebase is not a success. It's a bug, documented in `Tolerance_Programme_Handoff.md` §4:

> `MaterialPurchaseService` **never increased `CarryingValue`**, so every drain subtracted from a value that was never filled — on a fresh purchase the GL would have posted **৳0 COGS**. Fixed in all four purchase paths and proved by an end-to-end integration fact (buy 100 kg @ ৳50 → carry ৳5,000 → consume 50 kg while typing an absurd rate of 999 → still valued at ৳50, carry ৳2,500, `StockIn` unchanged at 100). Found by reading the service, **not** by a failing test — the pre-existing tests asserted the old behaviour.

Four lessons in one paragraph, and every one of them is the kind of thing that only comes from doing the work:

1. **A green test suite can encode a bug.** The pre-existing tests asserted the *old* behaviour, so they passed while the value anchor was empty. Passing tests prove consistency with intent — and the intent was wrong.
2. **The bug was found by reading code, not by running it.** Code review of the *consumer* (`ProductionService` draining carrying value) against the *producer* (`MaterialPurchaseService` never filling it) is what exposed it. Invariants make that review possible: if you know `Σ CarryingValue = GL`, you look for the path that doesn't add.
3. **The reproduction is specified in business terms, not code terms.** "Buy 100 kg @ ৳50, consume while typing rate 999, still valued at ৳50" is a test a non-programmer can read and confirm.
4. **The fix was verified across all four purchase paths** — new row, existing row, edit-revert, edit-apply — because a fix in one path leaves three ways to reproduce the bug.

The `AGENTS.md` rule that came out of it is a single sentence with a wide blast radius:

> **Any path that moves RM/PM quantity must move `CarryingValue` by exactly what the GL moves.** Check `MaterialPurchaseService`, `ProductionService`, `SalesOrderService`, `WasteService`, `SampleService` whenever touching stock flows — a missed one silently breaks the identity.

---

## What the architecture actually buys

It is easy to write "Clean Architecture" in a README. Here is what is concrete in this solution:

**Dependency direction is enforced by project references.** `VaatcoBMS.Application` cannot read `IConfiguration`, because it does not reference it. `AGENTS.md` states the rule plainly — configuration enters as primitives or plain option objects. That is why `CustomerDueCalculator` is a pure static method and why the tolerance gate takes a `ToleranceMeasureBasis` enum rather than reaching for a settings object. The constraint is structural, so it does not decay.

**Domain holds no framework types.** 89 entity files and 63 enum files with no EF Core attributes leaking into business rules. The state machines (`InvoiceStatus`, `ExpenseTopSheetStatus`, `SalarySheetStatus`, `ToleranceResolution`) are enums, so an invalid state is unrepresentable rather than merely unhandled.

**Reads and writes are separated by need, not by fashion.** EF Core for transactional writes with change tracking; Dapper for the 5-tab reports hub, ledgers, and territory analytics where a projection with six joins should not materialise an object graph. This is CQRS in the useful sense — a read model shaped for the screen — not event sourcing cosplay.

**Database constraints back up application logic.** XOR check constraints like `CK_ReturnItemDisposition_RecoveryXor` mean a return line cannot be simultaneously restocked and scrapped. Filtered unique indexes like `IX_Employees_LinkedUserId` mean one login maps to one employee even if two requests race.

**Authority is expressed once and reused.** `IJournalPostingService` is the only door to the ledger. `FgStockAllocator` is the only FEFO implementation. `CustomerDueCalculator` is the only due formula. `PendingToleranceGuard` is the only freeze check. Every one of these replaces what would otherwise be N copies drifting apart.

---

## Deliberate anti-patterns

A case study is more credible when it names what was rejected. From `AGENTS.md` and the tolerance spec's §12 (rejected alternatives):

| Anti-pattern | Why it was refused here |
| :--- | :--- |
| Editing a posted purchase to absorb a variance | The vendor ledger is a legal record. Corrections are mirror reversals. |
| Storing a derived unit rate | A stored rate goes stale the moment quantity moves. Store `CarryingValue`, derive the rate for display. |
| Compensating adjustments instead of reversals | A compensating entry makes the audit trail unreadable; a mirror reversal restores the original state provably. |
| Putting tolerance thresholds in `appsettings.json` | A financial policy needs an audit trail, permissions, and instant application. Config files have none. |
| Seeding `RowVersion` in application code | The database owns it; hand-seeding produced the "Stock changed by another user" incident. |
| Keeping the old `AdjustStockAsync` behind a flag | Two inventory write paths is two behaviours pretending to be one. |
| Two implementations of dealer due | The ৳13,412.50 vs ৳9,000.00 incident. |
| "Fixing" a known-failing excluded test | Changes the baseline silently. The exclusion is documented instead. |

---

## Where to look, if you are reviewing this

If you have limited time, these files carry the most signal per line:

| File | Why |
| :--- | :--- |
| `AGENTS.md` | Four invariants, the `RowVersion` pairing rule, and the blast-radius list for stock flows. Start here. |
| `VaatcoBMS.Application/Common/CustomerDueCalculator.cs` | The single-owner pattern, a pure function, and the incident that motivated it — in 45 lines. |
| `VaatcoBMS.Infrastructure/Persistence/AppDbContext.cs` | `EnforceJournalImmutability()` — an architectural rule enforced where it cannot be bypassed. |
| `VaatcoBMS.Infrastructure/Migrations/20260930125237_RowVersion_TriggerEmulation.cs` | Platform-semantics emulation with the incident documented in the migration. |
| `Docs/inventory/Tolerance_Programme_Handoff.md` | Handoff briefs, fail-closed design, locked decisions with amendments, and an honest §7 "what remains". |
| `VaatcoBMS.Application/Services/Accounting/JournalPostingService.cs` | The single choke point, validation ordering, and period locking. |

---

## Honest limitations

A case study that only lists strengths is marketing. These are open, and they are recorded in the repository itself:

- **UAT U1–U16 for the tolerance programme has not been executed by a human.** Four screens have never been opened by the owner. Pre-flight P1–P9 and a two-week parallel run precede sign-off.
- **The packing drain has no automated test.** `SyncPackingCarryingValueAsync` is verified by build and by inspection of the symmetric drain/restore; neither test project has a chalan-with-packing-lines fixture. UAT U15/U16 cover it manually. The handoff explicitly says *do not claim it is proven.*
- **One known-failing test folder** (`InfrastructureTests/**`), documented and excluded rather than fixed or deleted.
- **Three raw materials are measured in `Pcs`** and therefore get a 0% gate — every difference queues for a human. That is correct and inconvenient, and the fix is a per-material override, not a code change.
- **A material (`Yucca`) carries 5 kg at ৳0** because its only purchase was reversed. The arithmetic is right; the report looks wrong. It needs a Finance decision, not an engineering one.
- **The suite runs on SQL Server LocalDB for integration tests** while production is PostgreSQL. The Npgsql-specific behaviour is covered by the trigger migration and by manual verification, not by an equivalent automated harness.
- **105 remote branches.** Twenty-two are listed as active in the remote head; the rest are historical. Branch hygiene is the least disciplined part of an otherwise rigorous process.

---

## The transferable checklist

Distilled from the above — what a top-tier engineer does differently on a financial system:

1. **Name the invariants, then name every path that can break them.** An invariant with no violating-path list is a wish.
2. **One formula, one owner, one pure function.** Two implementations of a money calculation is a scheduled defect.
3. **Enforce rules at the narrowest choke point that cannot be bypassed** — `SaveChanges`, not code review.
4. **Make impossible states unrepresentable** with enums, XOR check constraints, and filtered unique indexes.
5. **Fail closed on money.** Unknown policy, zero cap, and missing row must all mean "hold", never "guess".
6. **Classify every tunable number.** Performance knob → config. Financial policy → database + audit trail + permission gate.
7. **Match platform semantics, not shapes,** when emulating a feature across databases.
8. **Delete the legacy write path completely** — service, endpoint, DTO, validator, test, view, link — and document the 404.
9. **Document the reversal, not just the decision.** Revision histories and amended locked decisions are how a maintainer learns what was already tried.
10. **Scope every test claim.** "Green" means nothing without "which suite", and a documented known-failing folder is stronger than a hidden one.
11. **Write the incident into the source.** The number that reached a customer is the most persuasive comment you will ever write.
12. **State what remains, in the repository.** A handoff brief with an honest "what is not done" list is worth more than a feature list.

---

## Attribution

The system described in Part I was built for **VAATCO Bangladesh Limited** by [@aburayhan98](https://github.com/aburayhan98).

 Figures were measured on 2026-11-11. Where the project's own documentation and a live measurement disagree, the measurement is used and the discrepancy is noted.

