# **Architecture Definition Document**

## **CatatNiaga — Offline-First Android POS for Indonesian UMKM & Retail**

**Document version:** 1.5

**Date:** 2026-08-10

**Status:** Baseline for AI-agent implementation (Rebranded to CatatNiaga with Full Feature Set)

**Scope:** Mobile application architecture, local data architecture, customer tier management, dynamic POS pricing engine, stock loss & shrinkage accounting, operational expenses & custom cost tracking (salaries, utilities, dues, rent), hardware integration, offline backup, persistent print queueing, hybrid logical clock synchronization, and 24-month automated database archival.

## **1\. Purpose**

This document defines the target architecture for CatatNiaga, an offline-first Android point-of-sale application for Indonesian micro, small, and medium businesses (UMKM/Ritel). The architecture preserves core functional capabilities—sales transactions, inventory, transaction records, paper and digital receipts, reports, expenses, Excel export, cash-drawer support, and offline operation—while materially improving data integrity, customer finance management, customer tier pricing, stock loss tracking, flexible operational expense accounting (salaries, utilities, dues, custom categories), hardware resilience, and disaster recovery features.

This is an original product architecture. It does not describe or authorize copying source code, branding, interface assets, database schema, package identity, or proprietary behavior of any third-party software.

## **2\. Architectural Drivers**

### **2.1 Business drivers**

1. **Reliable checkout without internet:** Cashiers must complete sales in remote, low-connectivity, or completely offline environments.  
2. **Flexible customer tier pricing:** Support custom customer categories (e.g., *Normal*, *Warung*, *Langganan*) with configurable percentage discounts automatically applied during POS checkout.  
3. **Accurate Stock Loss Accounting:** Calculate financial losses from expired goods (*kadaluwarsa*), damaged packaging/handling (*barang rusak/pecah*), unaccounted shrinkage/theft (*barang hilang/stok opname*), and owner personal draw (*diambil pemilik*) using COGS (Harga Pokok Pembelian / HPP) valuation.  
4. **Comprehensive Store Expense Tracking:** Record routine and custom operational expenses (e.g., *Gaji Pegawai*, *Listrik/Token PLN*, *Air PDAM*, *Iuran Kebersihan/Keamanan*, *Sewa Tempat*, *Maintenance*) with shift cash drawer sync, image proof attachments, and custom category creation.  
5. **Direct APK distribution:** The application installs directly as an APK and does not depend on Google Play Services for core operation.  
6. **Affordable deployment:** Small businesses can use ordinary Android phones/tablets and common Bluetooth thermal printers.  
7. **Financial trust:** Sales, payments, inventory movements, stock write-offs, store expenses, customer debt, customer tier discounts, and wallet balances must be auditable, deterministic, and recoverable.  
8. **Indonesian product bootstrap:** The system may import publicly available Indonesian product metadata using the api-product-indonesia service, while treating its documented random price values as non-authoritative suggestions only.

### **2.2 Technical drivers**

1. **Offline-first consistency:** Local SQLite is the single source of truth during disconnected operation.  
2. **Deterministic pricing & ledger behavior:** Money, stock, customer tier discounts (![][image1]), stock loss write-offs, store expenses, wallet, debt, and report calculations must be reproducible across time and device restarts using physical and logical sequence tracking.  
3. **Hardware resilience:** Printing, cash drawer, camera, barcode scanner, and file export can fail independently without corrupting a completed sale. Print jobs survive device power loss via persistent queueing.  
4. **Safe distribution:** APK signature verification, integrity checks, and offline activation/licensing must not require Play Integrity or Play Store entitlements.  
5. **Recoverability:** Manual and scheduled encrypted backups must support local storage and optional Google Drive integration.

## **3\. System Context**

### **3.1 Actors**

| Actor | Description |
| :---- | :---- |
| Owner | Configures business, users, customer types & default discounts, expense categories, stock write-off approvals, backup policies, security, exports, and sensitive financial reports. |
| Manager | Runs operations, manages customer tiers, audits stocktakes, logs/approves store expenses, registers stock loss/damage records, approves restricted transactions, and reviews inventory valuation. |
| Cashier | Completes sales, assigns customers to carts, accepts payments, records petty cash expenses from drawer, logs initial damaged/expired item reports, opens/closes shift, and reprints receipts. |
| Customer | Purchases items/services, belongs to a specific Customer Type (e.g., *Normal*, *Warung*, *Langganan*), may maintain receivables, and may have an advance wallet balance. |
| External product API | Provides optional product metadata by barcode or name. |
| Google Drive | Optional encrypted backup destination; not required for core operation. |
| Bluetooth printer | Prints receipts and may open a cash drawer via ESC/POS commands. |
| Barcode scanner | Camera or keyboard-wedge external scanner. |

### **3.2 Context diagram**

                        \+----------------------+  
                        |   Google Drive       |  
                        |   optional backups   |  
                        \+----------^-----------+  
                                   | encrypted archive  
Cashier/Owner/Manager              |  
          |                        |  
          v                        |  
\+----------------------+   HTTP    \+----------------------+  
| Android CatatNiaga   \+-----------\>| External Product API |  
| Application          |            | barcode/name lookup  |  
\+--+---------+---------+            \+----------------------+  
   |         |  
   |         \+-----------------------------+  
   |                                       |  
   v                                       v  
\+--------------------+              \+----------------------+  
| Local SQLite DB    |              | Local Storage/SAF    |  
| source of truth    |              | exports & backups    |  
\+--------------------+              \+----------------------+  
   |  
   \+--\> Bluetooth ESC/POS printer/cash drawer (via persistent queue)  
   \+--\> Camera / external barcode scanner  
   \+--\> Android share sheet / PDF or image receipts

## **4\. Architecture Principles**

1. **Local source of truth:** All finalized operational records are committed first to local SQLite.  
2. **Append-only financial ledgers:** Inventory movements, stock write-off loss entries, store expense logs, customer wallet/debt entries, and audit events are never silently overwritten.  
3. **Immutable pricing & cost snapshots:** When a sale, expense, or stock write-off is committed, unit pricing, unit cost price (![][image2]), and expense amount are snapshot-copied into historical ledger records to ensure Net Profit reports remain accurate regardless of future master data changes.  
4. **Atomic transactions:** Sales, customer tier discount calculations, stock deductions, stock write-off loss generation, store expense cash payouts, wallet redemption, shift cash events, print queue enqueuing, and audit records succeed or fail together.  
5. **Deterministic sequence order:** Every ledger entry incorporates a Hybrid Logical Clock (HLC) ![][image3] to guarantee total ordering independent of system clock skew.  
6. **Hardware decoupling & persistence:** Hardware operations execute asynchronously from financial transactions using disk-backed queues.  
7. **Adapter isolation:** Android hardware and cloud services are accessed strictly through replaceable interfaces.  
8. **Least privilege:** Owner, manager, cashier, and staff capabilities are enforced in domain services.  
9. **Privacy by default:** Customer PII (phone/email) is masked in generic exports unless full export is explicitly authorized.

## **5\. Solution Overview**

### **5.1 Deployment model**

* **Primary target:** Android APK distributed directly by the developer/business.  
* **Minimum recommended SDK:** Android 8.0 (API 26), subject to hardware vendor capabilities.  
* **Target form factors:** Phones and small tablets, portrait-first POS layout, landscape optional for tablets.  
* **Persistence:** SQLite in app-private storage, configured with Write-Ahead Logging (WAL).  
* **Optional cloud:** Encrypted backup transport and catalog bootstrap only.

### **5.2 Recommended implementation stack**

| Layer | Recommended choice | Rationale |
| :---- | :---- | :---- |
| Mobile framework | Flutter | Single codebase, stable Android APK builds, native FFI capabilities for SQLite. |
| Local DB | SQLite via Drift (with SQLCipher option) | Strongly typed schema, automatic compile-time migrations, fast native query compilation. |
| Background work | WorkManager via native platform channels | Reliable execution for persistent print queues, scheduled backups, and background exports. |
| Dependency injection | GetIt / Riverpod containers | Explicit module lifecycle management, scoping, and test mocks. |
| State management | Riverpod / Bloc | Predictable immutable state streams and use-case orchestration. |
| Printer bridge | Native Android ESC/POS adapter | Direct Bluetooth SPP/BLE socket lifecycle control without third-party layer fragility. |
| Secure storage | Android Keystore | Hardware-backed key protection for backup encryption keys and credentials. |
| Reports | Local SQL aggregation \+ Isolate worker | Prevents UI thread jank during multi-year report generation, loss analysis, and expense categorization. |

## **6\. Logical Architecture**

### **6.1 Layer model**

Presentation  
  Screens, widgets, customer type settings, expense entry UI, stock loss entry UI, receipt previews  
        |  
Application  
  Use cases, pricing engine, expense workflows, stock write-off workflows, queue management  
        |  
Domain  
  Entities, expense rules, stock loss rules, customer tier discount rules, invariants, HLC generators  
        |  
Data  
  Repositories, SQLite DAOs, migrations, loss/expense projections, archival engine  
        |  
Infrastructure  
  Android Keystore, WorkManager, native Bluetooth printer adapter, SAF, Drive API

### **6.2 Module catalog**

| Module | Responsibility |
| :---- | :---- |
| app\_shell | Navigation, theme, localization, feature flags, diagnostics. |
| identity\_access | Owner setup, PIN/biometric, users, roles, permissions, approvals. |
| catalog | Products, services, variants, bundles, categories, barcode/SKU, expiry tracking. |
| inventory | Stock ledger, receiving, adjustments, stocktake, low-stock alerts, stock loss & shrinkage tracking. |
| pos | Cart, pricing engine (base price \- customer tier discount \- manual promo), checkout, held carts. |
| payments | Tender methods, split payment, cash change, manual confirmations. |
| transactions | Sale lifecycle, refunds, voids, receipts, transaction history. |
| customer\_finance | Customers, customer types/tiers configuration, receivables, customer wallet, loyalty. |
| expenses\_shifts | Expense categories (default & custom), expense logging, cash drawer payout integration, shift open/close, variance. |
| reporting | Report definitions, filters, sales by customer type, stock loss reports, expense breakdown reports, profit & loss statements. |
| receipts\_hardware | Receipt rendering (showing customer tier discount), ESC/POS printer, persistent queue. |
| backup\_sync | Encrypted local backup, scheduling, Google Drive connector, HLC sync engine. |
| import\_export | CSV/XLSX/JSON exports, catalog imports, external API bootstrap. |
| maintenance | Seed data (default customer types, loss reasons, & expense categories), double-confirmed reset, integrity checks. |

## **7\. Data Architecture**

### **7.1 Database selection & PRAGMA configuration**

SQLite is the primary local database. To guarantee maximum concurrency and transaction speed, the following pragmas are enforced on startup:

PRAGMA journal\_mode \= WAL;  
PRAGMA synchronous \= NORMAL;  
PRAGMA foreign\_keys \= ON;  
PRAGMA busy\_timeout \= 5000;  
PRAGMA auto\_vacuum \= INCREMENTAL;  
PRAGMA cache\_size \= \-16000;

### **7.2 Data consistency model**

| Data class | Consistency rule |
| :---- | :---- |
| Customer Types | Configurable master table; soft-deletable (is\_active). Default types seeded on bootstrap. |
| Expense Categories | Configurable master table; supports system defaults & user custom categories. Soft-deletable. |
| Operational Expenses | Append-only financial ledger. Cash drawer payouts auto-adjust active shift balance. |
| Finalized sales/payments | Immutable; customer tier discount percentage is captured as a permanent snapshot. |
| Inventory & Stock Loss | Append-only stock movement ledger plus explicit stock\_loss\_records entries. |
| Customer debt/wallet | Append-only customer ledger plus rebuildable cached balance. |
| Print queue | Persistent, transaction-enqueued job queue with status state machine. |
| Audit events | Append-only, HLC-sequenced, and protected from ordinary deletion. |

### **7.3 Core schema domains**

business\_identity: businesses, outlets, settings, devices  
identity\_access:   users, roles, permissions, user\_roles  
catalog:           categories, products, product\_variants, bundles, bundle\_components  
inventory:         inventory\_balances, stock\_movements, stocktakes, stocktake\_lines,  
                   stock\_loss\_records, stock\_loss\_reasons  
operations:        shifts, cash\_movements, expenses, expense\_categories  
sales:             sales, sale\_lines, sale\_payments, refunds, refund\_lines, held\_carts  
customers:         customers, customer\_types, customer\_balance\_accounts, customer\_ledger\_entries,  
                   receivables, receivable\_payments, wallet\_vouchers, suppliers  
documents:         receipt\_documents, print\_queue, export\_jobs, import\_batches, import\_batch\_items  
resilience:        backup\_policies, backup\_runs, maintenance\_actions, archive\_runs  
governance:        audit\_events, sync\_cursors, tombstones

#### **Schema specification for Customer Types (customers.customer\_types)**

CREATE TABLE customer\_types (  
    id TEXT PRIMARY KEY,                       \-- e.g., 'ct-normal', 'ct-warung', 'ct-langganan'  
    code TEXT NOT NULL UNIQUE,                 \-- Short identifier: 'NORMAL', 'WARUNG', 'LANGGANAN'  
    name TEXT NOT NULL,                        \-- Display name: 'Normal', 'Warung', 'Langganan'  
    discount\_percentage REAL NOT NULL DEFAULT 0.0, \-- Default discount % (e.g. 0.0, 5.0, 2.0)  
    is\_system\_default INTEGER NOT NULL DEFAULT 0,  
    is\_active INTEGER NOT NULL DEFAULT 1,  
    created\_at INTEGER NOT NULL,  
    updated\_at INTEGER NOT NULL  
);

\-- Seed System Customer Types on App Bootstrap  
INSERT INTO customer\_types (id, code, name, discount\_percentage, is\_system\_default, is\_active, created\_at, updated\_at) VALUES  
('ct-normal', 'NORMAL', 'Normal', 0.0, 1, 1, 1770681600000, 1770681600000),  
('ct-warung', 'WARUNG', 'Warung', 5.0, 1, 1, 1770681600000, 1770681600000),  
('ct-langganan', 'LANGGANAN', 'Langganan', 2.0, 1, 1, 1770681600000, 1770681600000);

#### **Schema specification for Stock Loss & Shrinkage Management**

\-- Stock Loss Reasons Lookup Table  
CREATE TABLE stock\_loss\_reasons (  
    id TEXT PRIMARY KEY,                       \-- e.g., 'REASON\_EXPIRED'  
    code TEXT NOT NULL UNIQUE,                 \-- Short code: 'EXPIRED', 'DAMAGED\_PEST', 'DAMAGED\_HANDLING', 'SHRINKAGE\_THEFT', 'OWNER\_CONSUMPTION'  
    display\_name\_id TEXT NOT NULL,             \-- Indonesian label: 'Kadaluwarsa', 'Hama/Tikus', 'Barang Rusak/Pecah', 'Hilang/Selisih Opname', 'Konsumsi Pribadi'  
    is\_system\_default INTEGER NOT NULL DEFAULT 1,  
    is\_active INTEGER NOT NULL DEFAULT 1,  
    created\_at INTEGER NOT NULL  
);

\-- Seed System Stock Loss Reasons  
INSERT INTO stock\_loss\_reasons (id, code, display\_name\_id, is\_system\_default, is\_active, created\_at) VALUES  
('slr-expired', 'EXPIRED', 'Kadaluwarsa / Tanggal Lewat', 1, 1, 1770681600000),  
('slr-damaged-handling', 'DAMAGED\_HANDLING', 'Barang Rusak / Pecah / Bocor', 1, 1, 1770681600000),  
('slr-damaged-pest', 'DAMAGED\_PEST', 'Barang Digigit Hama / Tikus', 1, 1, 1770681600000),  
('slr-shrinkage-theft', 'SHRINKAGE\_THEFT', 'Barang Hilang / Selisih Stok Opname', 1, 1, 1770681600000),  
('slr-owner-draw', 'OWNER\_CONSUMPTION', 'Konsumsi Pribadi / Diambil Pemilik', 1, 1, 1770681600000),  
('slr-supplier-reject', 'SUPPLIER\_REJECT', 'Retur Supplier Ditolak', 1, 1, 1770681600000);

\-- Stock Loss Records Table  
CREATE TABLE stock\_loss\_records (  
    id TEXT PRIMARY KEY,                       \-- UUID  
    product\_variant\_id TEXT NOT NULL,  
    stock\_loss\_reason\_id TEXT NOT NULL,  
    quantity INTEGER NOT NULL,                 \-- Positive quantity lost  
    cost\_price\_snapshot INT NOT NULL,          \-- Cost price (COGS) at time of write-off in IDR minor units  
    total\_loss\_amount INT NOT NULL,            \-- quantity \* cost\_price\_snapshot in IDR minor units  
    expiry\_date INTEGER,                       \-- Optional expiry date (UTC epoch ms) if applicable  
    notes TEXT,                                \-- Detailed explanation (e.g. "Susu UHT kemasan kembung")  
    recorded\_by\_user\_id TEXT NOT NULL,         \-- Cashier or Manager who logged the issue  
    approved\_by\_user\_id TEXT,                  \-- Manager or Owner who approved write-off  
    status TEXT NOT NULL DEFAULT 'APPROVED',   \-- 'DRAFT', 'PENDING\_APPROVAL', 'APPROVED', 'REJECTED'  
    hlc\_physical INTEGER NOT NULL,             \-- HLC Timestamp physical component  
    hlc\_logical INTEGER NOT NULL,              \-- HLC Timestamp logical component  
    hlc\_node TEXT NOT NULL,                    \-- Node UUID  
    created\_at INTEGER NOT NULL,               \-- UTC Epoch milliseconds  
    FOREIGN KEY(product\_variant\_id) REFERENCES product\_variants(id),  
    FOREIGN KEY(stock\_loss\_reason\_id) REFERENCES stock\_loss\_reasons(id),  
    FOREIGN KEY(recorded\_by\_user\_id) REFERENCES users(id),  
    FOREIGN KEY(approved\_by\_user\_id) REFERENCES users(id)  
);

CREATE INDEX idx\_stock\_loss\_variant\_date ON stock\_loss\_records(product\_variant\_id, created\_at);  
CREATE INDEX idx\_stock\_loss\_reason ON stock\_loss\_records(stock\_loss\_reason\_id, created\_at);

#### **Schema specification for Store Expense Categories & Operational Expenses (operations)**

\-- Store Expense Categories Table (Supports System Defaults \+ User Custom Categories)  
CREATE TABLE expense\_categories (  
    id TEXT PRIMARY KEY,                       \-- e.g., 'ec-salary', 'ec-electricity', 'ec-custom-xxx'  
    code TEXT NOT NULL UNIQUE,                 \-- Short code: 'SALARY', 'ELECTRICITY', 'WATER', 'DUES', 'RENT', 'CUSTOM'  
    name TEXT NOT NULL,                        \-- Display Name: 'Gaji Pegawai', 'Listrik / PLN', 'Air / PDAM', 'Iuran Kebersihan/Keamanan'  
    is\_system\_default INTEGER NOT NULL DEFAULT 0, \-- 1 for pre-seeded, 0 for user created  
    is\_active INTEGER NOT NULL DEFAULT 1,  
    created\_at INTEGER NOT NULL,  
    updated\_at INTEGER NOT NULL  
);

\-- Seed System Default Expense Categories  
INSERT INTO expense\_categories (id, code, name, is\_system\_default, is\_active, created\_at, updated\_at) VALUES  
('ec-salary', 'SALARY', 'Gaji Pegawai / Staf', 1, 1, 1770681600000, 1770681600000),  
('ec-electricity', 'ELECTRICITY', 'Listrik / Token PLN', 1, 1, 1770681600000, 1770681600000),  
('ec-water', 'WATER', 'Air / PDAM', 1, 1, 1770681600000, 1770681600000),  
('ec-dues', 'DUES', 'Iuran Kebersihan / Keamanan / RT', 1, 1, 1770681600000, 1770681600000),  
('ec-rent', 'RENT', 'Sewa Tempat / Kios / Lahan', 1, 1, 1770681600000, 1770681600000),  
('ec-maintenance', 'MAINTENANCE', 'Perawatan / Perbaikan Peralatan', 1, 1, 1770681600000, 1770681600000),  
('ec-transport', 'TRANSPORT', 'Bensin / Transportasi / Kurir', 1, 1, 1770681600000, 1770681600000),  
('ec-other', 'OTHER\_OPERATIONAL', 'Pengeluaran Operasional Lainnya', 1, 1, 1770681600000, 1770681600000);

\-- Operational Expenses Table  
CREATE TABLE expenses (  
    id TEXT PRIMARY KEY,                       \-- UUID  
    expense\_category\_id TEXT NOT NULL,  
    amount INT NOT NULL,                       \-- Expense amount in IDR minor units (e.g. 150000\)  
    payment\_source TEXT NOT NULL,              \-- 'CASH\_DRAWER' (Petty Cash), 'BANK\_TRANSFER', 'OWNER\_POCKET'  
    shift\_id TEXT,                             \-- Nullable; linked if paid directly from current cashier cash drawer  
    description TEXT NOT NULL,                 \-- e.g., "Gaji mingguan Budi \+ bonus shift"  
    receipt\_image\_path TEXT,                   \-- Local image path for scanned receipt/nota proof  
    expense\_date INTEGER NOT NULL,             \-- Effective business expense timestamp (UTC Epoch ms)  
    recorded\_by\_user\_id TEXT NOT NULL,         \-- Cashier/Manager who logged expense  
    approved\_by\_user\_id TEXT,                  \-- Approving manager/owner if required  
    status TEXT NOT NULL DEFAULT 'APPROVED',   \-- 'DRAFT', 'APPROVED', 'VOIDED'  
    hlc\_physical INTEGER NOT NULL,  
    hlc\_logical INTEGER NOT NULL,  
    hlc\_node TEXT NOT NULL,  
    created\_at INTEGER NOT NULL,  
    updated\_at INTEGER NOT NULL,  
    FOREIGN KEY(expense\_category\_id) REFERENCES expense\_categories(id),  
    FOREIGN KEY(shift\_id) REFERENCES shifts(id),  
    FOREIGN KEY(recorded\_by\_user\_id) REFERENCES users(id),  
    FOREIGN KEY(approved\_by\_user\_id) REFERENCES users(id)  
);

CREATE INDEX idx\_expenses\_category\_date ON expenses(expense\_category\_id, expense\_date);  
CREATE INDEX idx\_expenses\_shift ON expenses(shift\_id);

#### **Detailed schema specification for documents.print\_queue**

CREATE TABLE print\_queue (  
    id TEXT PRIMARY KEY,                   \-- UUID  
    receipt\_document\_id TEXT NOT NULL,     \-- Foreign key to receipt\_documents  
    printer\_mac\_address TEXT,              \-- Target Bluetooth MAC or network IP  
    receipt\_payload BLOB NOT NULL,         \-- Pre-rendered ESC/POS command bytes  
    retry\_count INTEGER NOT NULL DEFAULT 0,  
    max\_retries INTEGER NOT NULL DEFAULT 5,  
    status TEXT NOT NULL,                  \-- 'PENDING', 'PRINTING', 'FAILED', 'COMPLETED'  
    last\_error TEXT,  
    created\_at INTEGER NOT NULL,           \-- UTC Epoch milliseconds  
    last\_attempt\_at INTEGER,               \-- UTC Epoch milliseconds  
    FOREIGN KEY(receipt\_document\_id) REFERENCES receipt\_documents(id) ON DELETE CASCADE  
);

CREATE INDEX idx\_print\_queue\_status\_created ON print\_queue(status, created\_at);

### **7.4 Real-World UMKM Field Use Cases & Loss Valuation Matrix**

In Indonesian micro/small retail operations, stock loss occurs across distinct operational scenarios. CatatNiaga evaluates all inventory losses based on **Cost Price (Harga Pokok Pembelian / HPP)**, not selling price, to isolate pure financial loss from lost margin.

![][image4]Where ![][image2] is the unit cost price snapshot from the inventory batch or product master.

\+----------------------------------------------------------------------------------------------------+  
|                                    REAL-WORLD FIELD USE CASE MATRIX                                |  
\+------------------------+------------------------------------+--------------------------------------+  
| Field Scenario         | Real-World Root Cause              | CatatNiaga System Action & Accounting|  
\+------------------------+------------------------------------+--------------------------------------+  
| 1\. Kadaluwarsa         | Milk/snack/bread expires on shelf; | Reason: 'EXPIRED'                    |  
|    (Expired Goods)     | distributor return window passed   | Stock: Decremented by Qty            |  
|                        | (e.g., non-returnable bread/dairy).| Loss: Recorded as Stock Loss Expense.|  
\+------------------------+------------------------------------+--------------------------------------+  
| 2\. Rusak / Pecah /     | Glass syrup bottle dropped; sauce  | Reason: 'DAMAGED\_HANDLING'           |  
|    Bocor (Damaged)     | pouch punctured; oil container     | Stock: Decremented by Qty            |  
|                        | leaking during handling/display.   | Loss: Recorded as Stock Loss Expense.|  
\+------------------------+------------------------------------+--------------------------------------+  
| 3\. Hama / Tikus        | Rats chew instant noodle packets,  | Reason: 'DAMAGED\_PEST'               |  
|    (Pest Damage)       | rice sacks, or snack bags          | Stock: Decremented by Qty            |  
|                        | overnight in storehouse.           | Loss: Isolated for Pest Risk Control.|  
\+------------------------+------------------------------------+--------------------------------------+  
| 4\. Hilang / Shoplift   | Unaccounted stock shrinkage        | Reason: 'SHRINKAGE\_THEFT'            |  
|    (Stock Shrinkage)   | revealed during physical stocktake | Stock: Auto-adjusted during Stocktake|  
|                        | (e.g., 3 cigarette packs missing). | Loss: Recorded as Shrinkage Loss.    |  
\+------------------------+------------------------------------+--------------------------------------+  
| 5\. Konsumsi Pribadi    | Store owner takes galon aqua,      | Reason: 'OWNER\_CONSUMPTION'          |  
|    (Owner Draw)        | cigarettes, or coffee for personal | Stock: Decremented by Qty            |  
|                        | or home use without payment.       | Loss: Categorized as Owner Equity    |  
|                        |                                    | Draw (Prive), NOT store shrinkage.   |  
\+------------------------+------------------------------------+--------------------------------------+

### **7.5 Financial Net Profit & Cash Flow Formula Integration**

To provide true business net profit accuracy for store owners, CatatNiaga incorporates Store Operational Expenses and Stock Write-offs into the Profit & Loss statement as follows:

#### **![][image5]![][image6]![][image7]![][image8]Shift Cash Drawer Balancing Integration**

When an expense is marked with payment\_source \= 'CASH\_DRAWER', the amount is immediately deducted from the active cashier shift's cash balance:

![][image9]*Note: Items classified under OWNER\_CONSUMPTION (Konsumsi Pribadi) are segregated from operational loss/expense reports and accounted for under Owner Equity Draw (Prive) so store operational efficiency isn't artificially penalized.*

### **7.6 Real-World UMKM Expense Management Matrix**

\+----------------------------------------------------------------------------------------------------+  
|                                REAL-WORLD UMKM EXPENSE FIELD MATRIX                                |  
\+------------------------+------------------------------------+--------------------------------------+  
| Expense Category       | Real-World Operational Context     | CatatNiaga System Action & Accounting|  
\+------------------------+------------------------------------+--------------------------------------+  
| 1\. Gaji Pegawai        | Weekly/monthly staff payroll,      | Category: 'SALARY'                   |  
|    (Staff Salaries)    | daily cashier allowances, overtime | Source: Cash Drawer or Bank          |  
|                        | paid directly from cash drawer.    | P\&L: Deducted under Labor OPEX       |  
\+------------------------+------------------------------------+--------------------------------------+  
| 2\. Listrik / Token PLN | Purchase of electricity token for  | Category: 'ELECTRICITY'              |  
|    (Electricity)       | store lighting, freezer, beverage  | Photo: Photo receipt of PLN token    |  
|                        | cooler refrigerators.              | P\&L: Utility Expense                 |  
\+------------------------+------------------------------------+--------------------------------------+  
| 3\. Air / PDAM          | Monthly water bill for store       | Category: 'WATER'                    |  
|    (Water Utility)     | toilet, cleaning, or sanitation.   | P\&L: Utility Expense                 |  
\+------------------------+------------------------------------+--------------------------------------+  
| 4\. Iuran Kebersihan /  | Daily/monthly community dues,      | Category: 'DUES'                     |  
|    Keamanan (Dues)     | waste disposal fee, market security| Source: Cash Drawer                  |  
|                        | paid to local environment officer. | P\&L: Local Fees & Dues Expense       |  
\+------------------------+------------------------------------+--------------------------------------+  
| 5\. Sewa Tempat         | Monthly/annual kiosk, stall, or    | Category: 'RENT'                     |  
|    (Store Rent)        | shophouse rent payments.           | P\&L: Facility Rent Expense           |  
\+------------------------+------------------------------------+--------------------------------------+  
| 6\. Custom Categories   | Custom cost items (e.g. "Beli Plastik| Category: Custom user creation       |  
|    (Custom Expense)    | Pembungkus", "Perbaikan Kipas").   | P\&L: Custom Operational Category     |  
\+------------------------+------------------------------------+--------------------------------------+

## **8\. Offline and Synchronization Architecture**

### **8.1 Offline behavior**

Core features that must operate with zero connectivity:

* Customer selection and dynamic customer type discount calculation in cart.  
* Logging and approving store operational expenses (salaries, utilities, dues, custom expenses) with automatic shift cash drawer adjustment.  
* Logging and approving stock loss records (expired, damaged, lost items) with immediate inventory updates.  
* Full CRUD operations on Customer Types, Expense Categories, and Stock Loss Reasons.  
* Product lookup, barcode scan, checkout, split payments, refunds/voids under policy.  
* Stock receiving/adjustment, stocktake, and loss reconciliation.  
* Expenses, shift open/close, customer debt and wallet operations.  
* Receipt rendering, thermal printing, and backup creation.

### **8.2 Multi-device synchronization & clock skew strategy**

When multi-device offline sync is enabled, physical clock drift on low-cost Android devices creates non-deterministic event ordering if physical system timestamps (![][image10]) are used alone.

#### **Hybrid Logical Clock (HLC) Implementation**

To achieve strict total event ordering across multiple offline devices without requiring a centralized server clock, all append-only transactions generate an HLC timestamp ![][image11], where:

* ![][image12]: Highest observed physical time in UTC milliseconds.  
* ![][image13]: Logical sequence counter incremented when events occur within the same physical millisecond.  
* ![][image14]: Unique hardware/installation UUID.

When Device ![][image15] receives an event from Device ![][image16] with HLC ![][image17], Device ![][image15] updates its local HLC state ![][image18] as follows:

## **![][image19]![][image20]9\. Hardware Integration Architecture**

### **9.1 Persistent print queueing & receipt formatting**

Receipt templates render dedicated breakdowns for sales, while stock loss and expense receipts can be printed for accounting archives:

\========================================  
     CATATNIAGA — BUKTI PENGELUARAN  
\========================================  
Dokumen: EXP-20260810-0042  
Tanggal: 2026-08-10 14:15  
Petugas: Budi (Kasir)  
Shift ID: SHIFT-882  
\----------------------------------------  
Kategori: Listrik / Token PLN  
Jumlah  : Rp 100.000  
Sumber  : Kas Laci (Petty Cash)  
Catatan : Beli token PLN 100rb no stroom  
          3214-5678-9012  
\----------------------------------------  
TOTAL PENGELUARAN            Rp 100.000  
\========================================  
 Tanda Tangan Kasir / Penerima

Unprinted documents persist in SQLite print\_queue and are auto-retried by background WorkManager tasks.

### **9.2 Barcode scanner adapter**

Support camera scanning (ML Kit / ZXing) and keyboard-wedge Bluetooth/USB scanners. Normalize scanner input and deduplicate rapid scans.

### **9.3 Local storage and Android SAF**

Use Storage Access Framework for user-selected export and backup destinations. App-private storage remains available for automatic backups and temporary packages.

## **10\. External Integration Architecture**

### **10.1 Product metadata provider**

Public integration contract for api-product-indonesia metadata endpoint remain active as advisory inputs only.

### **10.2 Google Drive connector**

Google Drive is optional and uploads only encrypted backup archives (.cnbak). Handles OAuth token refresh, resumable multipart upload, and retention enforcement.

## **11\. Security Architecture**

### **11.1 Authorization matrix**

| Operation | Cashier | Manager | Owner |
| :---- | :---- | :---- | :---- |
| Process Sale with Customer Tier Discount | Yes | Yes | Yes |
| Record Petty Cash Expense (\< Policy Threshold) | Yes | Yes | Yes |
| Create / Edit Custom Expense Categories | No | Yes | Yes |
| Approve High-Value Expenses (\> Policy Limit) | No | Yes | Yes |
| Log Stock Loss Draft (Expired/Damaged Item) | Yes | Yes | Yes |
| Approve Stock Write-Off Loss (\> Policy Limit) | No | Yes | Yes |
| Create / Modify Customer Types & Discount % | No | Yes | Yes |
| View Expense & Stock Loss Impact Reports | No | No | Yes |
| View Financial Profit & Loss Reports | No | No | Yes |
| Full PII Customer Export | No | No | Yes |
| Database Backup & Restore / Factory Reset | No | No | Yes |

## **12\. Backup, Export, and Recovery Architecture**

Customer types, expense categories, expense records with attached receipt photo paths, stock loss records, stock loss reasons, and historical cost snapshots are included in all standard encrypted backup archives (.cnbak) and accounting exports (XLSX).

## **13\. Performance Targets & Database Archival Strategy**

Expense records, stock loss records, and sales records older than 24 months preserve snapshots inside historical archive database files (catatniaga\_archive\_{YYYY}.sqlite).

## **14\. Reliability and Failure Handling Matrix**

| Scenario | System Resilience Response |
| :---- | :---- |
| Expense Exceeds Current Cash Drawer Balance | System warns cashier of negative cash drawer impact; requires Manager PIN override to log drawer payout exceeding physical cash. |
| Stock Loss Logged for Zero-Stock Item | System permits logging loss for audit purposes, flags stock balance as negative, and triggers a "Stock Discrepancy Alert" for manager review. |
| Owner Consumes Goods Mid-Shift | Cashier logs under OWNER\_CONSUMPTION. Stock decrements immediately, COGS recorded as Owner Equity Draw rather than operational loss. |

## **15\. Observability and Diagnostics**

Modifications to expense categories, customer tiers, and expense approvals generate audit events in audit\_events (e.g., ACTION\_EXPENSE\_LOGGED, CATEGORY: ELECTRICITY, AMOUNT: Rp 100.000).

## **16\. Test Architecture**

### **16.1 Unit & Integration Test Requirements**

* **Expense Cash Drawer Sync Test:** Verify that logging a Rp 50.000 cash expense decrements active shift cash drawer expected balance by exactly Rp 50.000.  
* **Stock Loss Valuation Test:** Verify that writing off 10 expired units calculates financial loss strictly using ![][image21], ignoring retail selling price.  
* **Profit & Loss Integration Test:** Ensure Net Profit calculation correctly subtracts both store operational expenses and stock losses while properly segregating OWNER\_CONSUMPTION into Owner Equity Draw.  
* **Stocktake Discrepancy Test:** Perform stock opname where actual count \< system count, verifying automatic generation of SHRINKAGE\_THEFT stock loss records.

## **17\. Deployment Architecture**

### **17.1 Direct APK release pipeline**

Standard release pipeline using signed APK artifacts (.apk) with embedded SHA-256 integrity verification. Seeding of default customer types, default expense categories, and stock loss reasons occurs automatically on schema migration.

## **18\. Architecture Decisions (ADR)**

### **ADR-001: SQLite as Local System of Record**

* **Status:** Approved  
* **Decision:** Use local SQLite configured with PRAGMA journal\_mode \= WAL;.

### **ADR-008: Stock Loss & Shrinkage Accounting Valuation**

* **Status:** Approved  
* **Context:** Retail store owners frequently lose profit from expired products, rat/pest damage, broken packaging, shoplifting, and personal consumption without clear financial visibility.  
* **Decision:** Implement dedicated stock\_loss\_records table linked to reason codes (EXPIRED, DAMAGED\_HANDLING, DAMAGED\_PEST, SHRINKAGE\_THEFT, OWNER\_CONSUMPTION). Value all losses at unit cost price snapshot (![][image2]) and integrate non-owner losses into Net Profit calculations.

### **ADR-009: Operational Expense & Custom Category Engine**

* **Status:** Approved  
* **Context:** Store operations require daily recording of operational costs (staff salary, PLN electricity, PDAM water, community dues, rent) to calculate true Net Profit.  
* **Decision:** Implement flexible expense\_categories table pre-seeded with default cost types while supporting user-defined custom categories. Expenses paid from CASH\_DRAWER automatically update active shift cash reconciliation.

## **19\. Risks and Mitigations**

| Risk | Consequence | Mitigation Strategy |
| :---- | :---- | :---- |
| Cashier logs fake expenses to extract cash from drawer | Financial theft | Require manager approval for expenses over threshold (e.g., \> Rp 100.000) and require receipt photo attachment. |
| Cashier marks stolen items as "Damaged" to hide theft | Inaccurate loss categorization | Require manager PIN approval for loss write-offs exceeding threshold amount. |
| Cost price (HPP) missing for item | Loss valuation calculated as Rp 0 | Fallback to latest purchase cost or prompt user to confirm unit cost during loss registration. |

## **20\. Acceptance Criteria**

1. **Pre-Seeded Customer Types:** Default customer tiers (Normal 0%, Warung 5%, Langganan 2%) are active upon system bootstrap.  
2. **Dynamic Customer Discounts:** Selecting a customer in the POS cart automatically applies their tier discount percentage to eligible items.  
3. **Pre-Seeded Expense Categories:** Default expense categories (Gaji Pegawai, Listrik / PLN, Air / PDAM, Iuran Kebersihan/Keamanan, Sewa Tempat, Perawatan, Transportasi, Lainnya) are active upon system bootstrap.  
4. **Custom Expense Categories:** Owner/Manager can add custom expense categories (e.g. "Beli Plastik Pembungkus") which seamlessly appear in POS expense entry screens and reports.  
5. **Cash Drawer Payout Sync:** Logging an expense with payment\_source \= 'CASH\_DRAWER' immediately decrements expected cash in the active shift summary.  
6. **Pre-Seeded Stock Loss Reasons:** Default stock loss reasons (Kadaluwarsa, Barang Rusak/Pecah, Hama/Tikus, Hilang/Selisih Opname, Konsumsi Pribadi) are active upon system bootstrap.  
7. **COGS Loss Valuation:** Writing off an expired item values the financial loss strictly at unit cost price (![][image22]), preventing margin distortion.  
8. **Automatic Inventory Deduction:** Recording an approved stock loss immediately decrements available inventory balances.  
9. **Owner Draw Segregation:** Goods logged under Konsumsi Pribadi decrement inventory but are reported separately under Owner Equity Draw, not operational store loss.  
10. **Net Profit Integration:** P\&L reports compute Gross Profit, subtract total Operational Expenses and non-owner Stock Losses, and display accurate Net Operational Profit.

## **21\. References**

1. CatatNiaga Domain Driven Pricing, Inventory, Stock Loss & Expense Engine Specifications.  
2. ariph007/api-product-indonesia Open API Documentation.  
3. Hybrid Logical Clocks (Kulkarni et al., Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases).
