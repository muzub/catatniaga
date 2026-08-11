# **CatatNiaga — Unified Master Technical Specification Suite & Product Requirements Document (PRD)**

**Document Version:** 3.0 (Unified Master Specification with Complete PRD)

**Date:** 2026-08-10

**Target System:** CatatNiaga (Offline-First Android POS for Indonesian UMKM)

**Target Tech Stack:** Flutter 3.x (Dart 3.x), Drift SQLite DB (drift), Riverpod, WorkManager, SQLCipher

## **Table of Contents**

1. [Product Requirements Document (PRD) & Screen Flow Spec](#bookmark=id.lwr9stq9rjr7)  
2. [Architecture Definition Document (ADD)](#bookmark=id.971a76uckp42)  
3. [AI Agent Steering Directives & Rules (AGENT\_INSTRUCTIONS.md)](#bookmark=id.dz8sbx1egkm7)  
4. [Complete Database Schema & Drift DDL Definitions](#bookmark=id.q4vzxxmy9jyj)  
5. [Atomic Data Access Objects (DAOs)](#bookmark=id.q44uk7xrk46t)  
6. [Stock Opname (Stocktake) & Reporting Engine Specification](#bookmark=id.x5krxiy4g1bx)  
7. [Implementation Roadmap & Test Suite Blueprint](#bookmark=id.kxbwfiz1xexr)

## **1\. Product Requirements Document (PRD) & Screen Flow Spec**

### **1.1 Product Vision & Target Audience**

CatatNiaga adalah aplikasi Point of Sale (POS) Android *offline-first* yang dirancang khusus untuk Ritel dan UMKM Indonesia (Warung Kelontong, Minimarket, Toko Pakaian, Counter HP, Kedai, dan Apotek). Aplikasi ini membantu pemilik usaha mencatat transaksi kasir secara cepat tanpa tergantung koneksi internet, mengelola diskon bertingkat (*Customer Tiers*), menghitung kerugian barang (*Stock Loss/Shrinkage*), mencatat beban operasional toko (*Operational Expenses*), melakukan audit stok fisik (*Stock Opname*), serta menghasilkan laporan Laba Rugi Bersih (*Net Profit*) secara otomatis.

### **1.2 Persona & Authorization Matrix**

| Role | Primary Responsibilities | Authorization Limits |
| :---- | :---- | :---- |
| **Owner** | Pemilik bisnis; memantau performa keuangan, mengelola master data, membuat kategori biaya custom, mengatur kebijakan diskon/backup. | Akses penuh seluruh fitur, Laporan Laba Rugi, Export PII, Backup/Restore, Reset Data. |
| **Manager** | Pengelola toko harian; mengaudit stok opname, menyetujui *write-off* kerugian stok & biaya operasional yang melebihi batas toleransi kasir. | Akses operasional, stok opname, input/approval kerugian stok & pengeluaran, tanpa akses reset/export PII. |
| **Cashier** | Kasir toko; melayani transaksi penjualan, memilih pelanggan, menerima pembayaran, mencatat *petty cash* kecil dari laci. | Transaksi POS, cetak ulang struk, buka/tutup shift, catat *draft* pengeluaran/kerugian kecil (\< limit policy). |

### **1.3 Key Feature Modules & Business Requirements**

#### **Module A: POS Checkout & Dynamic Customer Tiers**

* **Dynamic Customer Tier Discounting:**  
  * Tipe pelanggan default: Normal (Diskon 0%), Warung (Diskon 5%), Langganan (Diskon 2%).  
  * Saat kasir memilih pelanggan di keranjang belanja, diskon persentase tipe pelanggan otomatis memotong harga item/subtotal.  
  * Snapshot nama tipe pelanggan dan persentase diskon disimpan secara permanen pada header transaksi sales.  
* **Payment Split & Wallet/Debt Management:**  
  * Menerima metode pembayaran: Cash, QRIS, Transfer Bank, Hutang Pelanggan (*Customer Debt*), dan Saldo Wallet (*Customer Wallet*).  
  * Kembalian tunai dapat dialokasikan langsung menjadi Top-Up Saldo Wallet pelanggan untuk transaksi luring berikutnya.

#### **Module B: Stock Loss & Shrinkage Engine**

* **COGS Valuation:** Kerugian barang yang dihapus (*write-off*) dihitung berdasarkan Harga Pokok Pembelian (![][image1] / HPP), bukan harga jual.  
* **Field Loss Reasons:**  
  1. EXPIRED: Makanan/minuman kadaluwarsa.  
  2. DAMAGED\_HANDLING: Kemasan pecah, penyok, atau bocor saat penataan.  
  3. DAMAGED\_PEST: Barang rusak digigit hama/tikus di gudang.  
  4. SHRINKAGE\_THEFT: Barang hilang/selisih minus hasil audit stok opname.  
  5. OWNER\_CONSUMPTION: Barang diambil pemilik untuk konsumsi pribadi. Dikelompokkan sebagai Penarikan Modal (*Prive*), TIDAK mengurangi efisiensi operasional toko.

#### **Module C: Operational Expenses & Shift Cash Sync**

* **Kategori Pengeluaran Toko:**  
  * Default: Gaji Pegawai, Listrik / PLN, Air / PDAM, Iuran Kebersihan/Keamanan, Sewa Tempat, Perawatan, Transportasi, Lainnya.  
  * Mendukung pembuatan Kategori Pengeluaran Custom oleh Owner/Manager (contoh: "Beli Plastik Pembungkus").  
* **Cash Drawer Reconciliation:**  
  * Pengeluaran yang bersumber dari CASH\_DRAWER secara otomatis mengurangi saldo kasir aktif pada shift berjalan.

#### **Module D: Physical Stock Opname (Stocktake)**

* Melakukan audit fisik persediaan barang secara parsial maupun menyeluruh.  
* Menghitung selisih (![][image2]).  
  * **Selisih Minus (![][image3]):** Otomatis membuat pergerakan stok minus dan merekam *Stock Loss Record* dengan alasan SHRINKAGE\_THEFT.  
  * **Selisih Plus (![][image4]):** Otomatis menambah stok sistem tanpa mencatat kerugian.

### **1.4 Detailed User Stories (Gherkin Format)**

Feature: Customer Tier Discount Application in POS Cart  
  Scenario: Cashier selects a Warung tier customer during checkout  
    Given the cashier has added "Minyak Goreng 2L" priced at Rp 30000 to cart  
    And customer "Pak Rudi" belongs to "Warung" tier with 5.0% discount  
    When the cashier assigns customer "Pak Rudi" to the active cart  
    Then the calculated item discount should be Rp 1500  
    And the item final total should display Rp 28500  
    And the snapshot discount percent "5.0" and tier name "Warung" should be attached to the sale draft

Feature: Operational Expense Recording with Cash Drawer Sync  
  Scenario: Cashier pays local electricity token from cash drawer  
    Given the current active shift has opening cash of Rp 200000  
    And no prior cash sales or expenses have occurred in this shift  
    When the cashier logs an expense of Rp 50000 under category "Listrik / Token PLN"  
    And sets payment source to "CASH\_DRAWER"  
    Then the active shift expected cash balance should immediately update to Rp 150000  
    And an expense record should be created with status "APPROVED"

Feature: Stock Opname Reconciliation with Auto Shrinkage Loss  
  Scenario: Physical stock count reveals missing inventory  
    Given product "Indomie Goreng" has system stock of 50 pcs with HPP of Rp 3100  
    When the manager completes a stock opname with physical count of 47 pcs  
    Then the product stock quantity should update to 47 pcs  
    And a negative stock movement of \-3 pcs should be inserted  
    And an auto stock loss record with reason "SHRINKAGE\_THEFT" and loss amount Rp 9300 should be generated

### **1.5 Edge Case & Error Handling Matrix**

| Edge Case Scenario | System Action & Resilience Protocol |
| :---- | :---- |
| **App crash or power loss during checkout commit** | Transaction runs within SQLite transaction block. Rollback occurs automatically. Draft cart is preserved in held\_carts table. Idempotency key prevents double deduction. |
| **Bluetooth printer offline during payment** | Sale completes successfully in SQLite. Print job is spooled into print\_queue table with status PENDING. Toast notification alerts cashier, and background worker retries automatically upon reconnection. |
| **Expense amount exceeds physical cash drawer balance** | System presents warning dialog indicating negative cash balance. Requires Manager PIN override before transaction can be saved. |
| **Stock Loss recorded for item with 0 stock** | System permits logging loss for audit purposes, updates stock balance to negative, and flags a "Stock Discrepancy Alert" on Manager dashboard. |
| **System physical clock changed manually offline** | Hybrid Logical Clock (HLC) maintains monotonic event sequence ordering (![][image5]). Time jump is logged as security audit event. |

### **1.6 Screen Flow Architecture**

\[ Login Screen / PIN Entry \]  
          |  
          v  
\[ Main Navigation Shell \]  
   ├──\> \[ POS Checkout Screen \]  
   │      ├── Customer Selector Modal (Applies Tier Discount)  
   │      ├── Cart Item Discount / Quantity Editor  
   │      ├── Payment Modal (Cash / QRIS / Debt / Wallet / Change to Wallet)  
   │      └── Receipt Preview & Async Print Spooler  
   │  
   ├──\> \[ Product Catalog & Inventory \]  
   │      ├── Product / Variant Master List & Barcode Scanner  
   │      ├── Stock Write-Off Screen (Expired / Damaged / Pest / Owner Draw)  
   │      └── Stock Opname Screen (Physical Count Entry \-\> Auto Loss Log)  
   │  
   ├──\> \[ Operations & Expenses \]  
   │      ├── Active Shift Cash Drawer Summary  
   │      ├── Log Operational Expense Form (PLN, Water, Salary, Custom)  
   │      └── Manage Expense Categories Modal  
   │  
   ├──\> \[ Customer Management \]  
   │      ├── Customer List & Tier Config (Normal, Warung, Langganan)  
   │      └── Debt & Wallet Ledger History  
   │  
   └──\> \[ Reports & Financials (Owner Only) \]  
          ├── Profit & Loss (P\&L) Statement  
          ├── Stock Loss / Shrinkage Report  
          └── Excel (XLSX) Export Engine & Encrypted Backup Manager

## **2\. Architecture Definition Document (ADD)**

### **2.1 Architectural Drivers & Invariants**

* **Offline-First Authority:** SQLite lokal adalah sumber kebenaran tunggal (*single source of truth*).  
* **IDR Currency Integrity:** Seluruh nominal Rupiah disimpan sebagai 64-bit Integer (int / BIGINT) dalam unit terkecil (minor unit, Rp 15.000 \= 15000). **Dilarang keras** menggunakan double atau float untuk menyimpan nilai uang.  
* **Append-Only Ledgers:** Transaksi penjualan, mutasi stok, kerugian stok, biaya operasional, dan buku besar pelanggan bersifat *append-only*.  
* **Deterministic Ordering:** Menggunakan Hybrid Logical Clock (HLC) ![][image6] untuk mengurutkan kejadian secara absolut di seluruh perangkat luring.

### **2.2 System Context & Layered Clean Architecture**

Presentation (Widgets, Notifiers) \-\> Application (Use Cases) \-\> Domain (Entities) \<- Data (Drift DAOs) \<- Infrastructure (Bluetooth, Keystore)

### **2.3 Real-World UMKM Field Scenarios & P\&L Integration**

## **![][image7]![][image8]![][image9]3\. AI Agent Steering Directives & Rules (AGENT\_INSTRUCTIONS.md)**

### **3.1 Core Architectural Invariants**

1. **Clean Architecture Boundaries:**  
   * **DILARANG KERAS** menulis query SQL, Drift DAO, atau akses basis data langsung di dalam Flutter Widgets/Pages.  
   * Seluruh manajemen state UI **WAJIB** menggunakan Riverpod (NotifierProvider, AsyncNotifierProvider, atau StreamProvider).  
2. **Currency & Math Rules:**  
   * Mata uang Rupiah disimpan sebagai int (contoh: Rp 15.000 \= 15000).  
   * Perhitungan diskon persentase menggunakan double hanya saat kalkulasi sementara, kemudian di-round secara eksplisit ke int:  
     ![][image10]  
3. **Price & Cost Snapshots:**  
   * Saat transaksi committed, nilai unit\_price, unit\_cost\_price (HPP), dan discount\_amount **WAJIB** di-snapshot langsung ke dalam baris histori (sale\_lines, stock\_loss\_records, stocktake\_lines).  
4. **Hardware Decoupling:**  
   * Eksekusi transaksi POS tidak boleh menunggu respon printer Bluetooth. Cetak struk dilakukan secara *asynchronous* dengan memasukkan payload ESC/POS ke tabel print\_queue dalam transaksi SQLite yang sama.  
5. **PRAGMA Enforcement:**  
   * Setiap koneksi SQLite yang dibuka WAJIB mengeksekusi PRAGMA journal\_mode \= WAL, synchronous \= NORMAL, foreign\_keys \= ON, busy\_timeout \= 5000, auto\_vacuum \= INCREMENTAL, dan cache\_size \= \-16000.

## **4\. Complete Database Schema & Drift DDL Definitions**

### **4.1 Native Connection Setup (lib/data/local/connection/native\_connection.dart)**

import 'dart:io';  
import 'package:drift/drift.dart';  
import 'package:drift/native.dart';  
import 'package:path\_provider/path\_provider.dart';  
import 'package:path/path.dart' as p;

QueryExecutor openCatatNiagaConnection({bool isEncrypted \= false, String? password}) {  
  return LazyDatabase(() async {  
    final dbFolder \= await getApplicationDocumentsDirectory();  
    final file \= File(p.join(dbFolder.path, 'catatniaga\_main.sqlite'));

    return NativeDatabase.createInBackground(  
      file,  
      setup: (rawDb) {  
        rawDb.execute('PRAGMA journal\_mode \= WAL;');  
        rawDb.execute('PRAGMA synchronous \= NORMAL;');  
        rawDb.execute('PRAGMA foreign\_keys \= ON;');  
        rawDb.execute('PRAGMA busy\_timeout \= 5000;');  
        rawDb.execute('PRAGMA auto\_vacuum \= INCREMENTAL;');  
        rawDb.execute('PRAGMA cache\_size \= \-16000;'); // 16MB Cache  
      },  
    );  
  });  
}

### **4.2 Complete Master DDL Table Definitions (lib/data/local/tables/master\_tables.dart)**

import 'package:drift/drift.dart';

// \==========================================  
// 1\. BUSINESS IDENTITY & SHIFT MANAGEMENT  
// \==========================================

@DataClassName('BusinessEntity')  
class Businesses extends Table {  
  TextColumn get id \=\> text()(); // UUID v4  
  TextColumn get name \=\> text().withLength(min: 1, max: 100)();  
  TextColumn get ownerName \=\> text()();  
  TextColumn get phoneNumber \=\> text().nullable()();  
  TextColumn get address \=\> text().nullable()();  
  IntColumn get createdAt \=\> integer()(); // UTC Epoch ms  
  IntColumn get updatedAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('UserEntity')  
class Users extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get username \=\> text().unique()();  
  TextColumn get fullName \=\> text()();  
  TextColumn get pinHash \=\> text()();  
  TextColumn get role \=\> text()(); // 'OWNER', 'MANAGER', 'CASHIER'  
  BoolColumn get isActive \=\> boolean().withDefault(const Constant(true))();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get updatedAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('ShiftEntity')  
class Shifts extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get userId \=\> text().references(Users, \#id)();  
  IntColumn get openingCashAmount \=\> integer()(); // IDR Minor units  
  IntColumn get closingCashAmount \=\> integer().nullable()();  
  IntColumn get expectedCashAmount \=\> integer()();  
  IntColumn get totalCashSales \=\> integer().withDefault(const Constant(0))();  
  IntColumn get totalCashExpenses \=\> integer().withDefault(const Constant(0))();  
  IntColumn get totalCashDebtCollected \=\> integer().withDefault(const Constant(0))();  
  IntColumn get openedAt \=\> integer()();  
  IntColumn get closedAt \=\> integer().nullable()();  
  TextColumn get status \=\> text().withDefault(const Constant('OPEN'))(); // 'OPEN', 'CLOSED'  
  TextColumn get notes \=\> text().nullable()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

// \==========================================  
// 2\. MASTER LOOKUP TABLES (SEEDS)  
// \==========================================

@DataClassName('CustomerTypeEntity')  
class CustomerTypes extends Table {  
  TextColumn get id \=\> text()(); // e.g., 'ct-normal', 'ct-warung', 'ct-langganan'  
  TextColumn get code \=\> text().unique()(); // 'NORMAL', 'WARUNG', 'LANGGANAN'  
  TextColumn get name \=\> text()();  
  RealColumn get discountPercentage \=\> real().withDefault(const Constant(0.0))();  
  BoolColumn get isSystemDefault \=\> boolean().withDefault(const Constant(false))();  
  BoolColumn get isActive \=\> boolean().withDefault(const Constant(true))();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get updatedAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('StockLossReasonEntity')  
class StockLossReasons extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get code \=\> text().unique()(); // 'EXPIRED', 'DAMAGED\_HANDLING', 'DAMAGED\_PEST', 'SHRINKAGE\_THEFT', 'OWNER\_CONSUMPTION'  
  TextColumn get displayNameId \=\> text()();  
  BoolColumn get isSystemDefault \=\> boolean().withDefault(const Constant(true))();  
  BoolColumn get isActive \=\> boolean().withDefault(const Constant(true))();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('ExpenseCategoryEntity')  
class ExpenseCategories extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get code \=\> text().unique()(); // 'SALARY', 'ELECTRICITY', 'WATER', 'DUES', 'RENT', 'MAINTENANCE', 'TRANSPORT', 'OTHER\_OPERATIONAL'  
  TextColumn get name \=\> text()();  
  BoolColumn get isSystemDefault \=\> boolean().withDefault(const Constant(false))();  
  BoolColumn get isActive \=\> boolean().withDefault(const Constant(true))();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get updatedAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

// \==========================================  
// 3\. PRODUCT CATALOG DOMAIN  
// \==========================================

@DataClassName('CategoryEntity')  
class Categories extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get name \=\> text()();  
  TextColumn get iconName \=\> text().nullable()();  
  BoolColumn get isActive \=\> boolean().withDefault(const Constant(true))();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get updatedAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('ProductEntity')  
class Products extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get categoryId \=\> text().nullable().references(Categories, \#id)();  
  TextColumn get name \=\> text()();  
  TextColumn get description \=\> text().nullable()();  
  TextColumn get barcode \=\> text().nullable().unique()();  
  TextColumn get unitName \=\> text().withDefault(const Constant('pcs'))();  
  BoolColumn get isActive \=\> boolean().withDefault(const Constant(true))();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get updatedAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('ProductVariantEntity')  
class ProductVariants extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get productId \=\> text().references(Products, \#id)();  
  TextColumn get variantName \=\> text().withDefault(const Constant('Standard'))();  
  TextColumn get sku \=\> text().nullable().unique()();  
  IntColumn get price \=\> integer()(); // Minor units  
  IntColumn get costPrice \=\> integer()(); // Minor units (HPP)  
  IntColumn get stockQuantity \=\> integer().withDefault(const Constant(0))();  
  IntColumn get minStockThreshold \=\> integer().withDefault(const Constant(5))();  
  BoolColumn get isActive \=\> boolean().withDefault(const Constant(true))();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get updatedAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

// \==========================================  
// 4\. CUSTOMERS & LEDGERS  
// \==========================================

@DataClassName('CustomerEntity')  
class Customers extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get customerTypeId \=\> text().references(CustomerTypes, \#id)();  
  TextColumn get name \=\> text()();  
  TextColumn get phoneNumber \=\> text().nullable()();  
  IntColumn get debtBalance \=\> integer().withDefault(const Constant(0))();  
  IntColumn get walletBalance \=\> integer().withDefault(const Constant(0))();  
  BoolColumn get isActive \=\> boolean().withDefault(const Constant(true))();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get updatedAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('CustomerLedgerEntryEntity')  
class CustomerLedgerEntries extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get customerId \=\> text().references(Customers, \#id)();  
  TextColumn get transactionType \=\> text()(); // 'DEBT\_INCREASE', 'DEBT\_PAYMENT', 'WALLET\_TOPUP', 'WALLET\_REDEMPTION'  
  IntColumn get amount \=\> integer()();  
  IntColumn get runningDebtBalance \=\> integer()();  
  IntColumn get runningWalletBalance \=\> integer()();  
  TextColumn get referenceId \=\> text().nullable()();  
  TextColumn get notes \=\> text().nullable()();  
  IntColumn get hlcPhysical \=\> integer()();  
  IntColumn get hlcLogical \=\> integer()();  
  TextColumn get hlcNode \=\> text()();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

// \==========================================  
// 5\. INVENTORY MOVEMENTS & STOCK WRITE-OFF  
// \==========================================

@DataClassName('StockMovementEntity')  
class StockMovements extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get productVariantId \=\> text().references(ProductVariants, \#id)();  
  TextColumn get movementType \=\> text()(); // 'SALE', 'PURCHASE\_RECEIPT', 'STOCK\_LOSS', 'STOCKTAKE\_ADJUSTMENT'  
  IntColumn get quantityDelta \=\> integer()();  
  IntColumn get costPriceSnapshot \=\> integer()();  
  IntColumn get resultingStockQuantity \=\> integer()();  
  TextColumn get referenceId \=\> text().nullable()();  
  IntColumn get hlcPhysical \=\> integer()();  
  IntColumn get hlcLogical \=\> integer()();  
  TextColumn get hlcNode \=\> text()();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('StockLossRecordEntity')  
class StockLossRecords extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get productVariantId \=\> text().references(ProductVariants, \#id)();  
  TextColumn get stockLossReasonId \=\> text().references(StockLossReasons, \#id)();  
  IntColumn get quantity \=\> integer()();  
  IntColumn get costPriceSnapshot \=\> integer()();  
  IntColumn get totalLossAmount \=\> integer()(); // quantity \* costPriceSnapshot  
  IntColumn get expiryDate \=\> integer().nullable()();  
  TextColumn get notes \=\> text().nullable()();  
  TextColumn get recordedByUserId \=\> text().references(Users, \#id)();  
  TextColumn get approvedByUserId \=\> text().nullable().references(Users, \#id)();  
  TextColumn get status \=\> text().withDefault(const Constant('APPROVED'))();  
  IntColumn get hlcPhysical \=\> integer()();  
  IntColumn get hlcLogical \=\> integer()();  
  TextColumn get hlcNode \=\> text()();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

// \==========================================  
// 6\. OPERATIONAL EXPENSES DOMAIN  
// \==========================================

@DataClassName('ExpenseEntity')  
class Expenses extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get expenseCategoryId \=\> text().references(ExpenseCategories, \#id)();  
  IntColumn get amount \=\> integer()(); // IDR Minor units  
  TextColumn get paymentSource \=\> text()(); // 'CASH\_DRAWER', 'BANK\_TRANSFER', 'OWNER\_POCKET'  
  TextColumn get shiftId \=\> text().nullable().references(Shifts, \#id)();  
  TextColumn get description \=\> text()();  
  TextColumn get receiptImagePath \=\> text().nullable()();  
  IntColumn get expenseDate \=\> integer()();  
  TextColumn get recordedByUserId \=\> text().references(Users, \#id)();  
  TextColumn get approvedByUserId \=\> text().nullable().references(Users, \#id)();  
  TextColumn get status \=\> text().withDefault(const Constant('APPROVED'))();  
  IntColumn get hlcPhysical \=\> integer()();  
  IntColumn get hlcLogical \=\> integer()();  
  TextColumn get hlcNode \=\> text()();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get updatedAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

// \==========================================  
// 7\. SALES & POS CHECKOUT DOMAIN  
// \==========================================

@DataClassName('SaleEntity')  
class Sales extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get invoiceNumber \=\> text().unique()();  
  TextColumn get shiftId \=\> text().references(Shifts, \#id)();  
  TextColumn get customerId \=\> text().nullable().references(Customers, \#id)();  
  TextColumn get customerTypeNameSnapshot \=\> text()();  
  RealColumn get customerTypeDiscountPercentSnapshot \=\> real()();  
  IntColumn get subtotalAmount \=\> integer()();  
  IntColumn get customerTypeDiscountAmount \=\> integer()();  
  IntColumn get additionalDiscountAmount \=\> integer().withDefault(const Constant(0))();  
  IntColumn get totalAmount \=\> integer()();  
  IntColumn get totalCostAmount \=\> integer()(); // Total COGS (HPP)  
  TextColumn get paymentStatus \=\> text()(); // 'PAID', 'PARTIAL\_DEBT', 'UNPAID\_DEBT'  
  TextColumn get cashierUserId \=\> text().references(Users, \#id)();  
  IntColumn get hlcPhysical \=\> integer()();  
  IntColumn get hlcLogical \=\> integer()();  
  TextColumn get hlcNode \=\> text()();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('SaleLineEntity')  
class SaleLines extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get saleId \=\> text().references(Sales, \#id)();  
  TextColumn get productVariantId \=\> text().references(ProductVariants, \#id)();  
  TextColumn get productNameSnapshot \=\> text()();  
  TextColumn get variantNameSnapshot \=\> text()();  
  IntColumn get unitPriceSnapshot \=\> integer()();  
  IntColumn get unitCostPriceSnapshot \=\> integer()();  
  IntColumn get quantity \=\> integer()();  
  IntColumn get itemSubtotal \=\> integer()();  
  IntColumn get itemDiscountAmount \=\> integer().withDefault(const Constant(0))();  
  IntColumn get itemTotalAmount \=\> integer()();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('SalePaymentEntity')  
class SalePayments extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get saleId \=\> text().references(Sales, \#id)();  
  TextColumn get paymentMethod \=\> text()(); // 'CASH', 'QRIS', 'BANK\_TRANSFER', 'CUSTOMER\_DEBT', 'CUSTOMER\_WALLET'  
  IntColumn get amountTendered \=\> integer()();  
  IntColumn get amountApplied \=\> integer()();  
  IntColumn get changeReturned \=\> integer().withDefault(const Constant(0))();  
  IntColumn get hlcPhysical \=\> integer()();  
  IntColumn get hlcLogical \=\> integer()();  
  TextColumn get hlcNode \=\> text()();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

// \==========================================  
// 8\. STOCK OPNAME DOMAIN  
// \==========================================

@DataClassName('StocktakeEntity')  
class Stocktakes extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get stocktakeNumber \=\> text().unique()();  
  TextColumn get status \=\> text().withDefault(const Constant('COMPLETED'))(); // 'DRAFT', 'COMPLETED', 'CANCELLED'  
  TextColumn get notes \=\> text().nullable()();  
  TextColumn get performedByUserId \=\> text().references(Users, \#id)();  
  TextColumn get approvedByUserId \=\> text().nullable().references(Users, \#id)();  
  IntColumn get totalItemsAudited \=\> integer().withDefault(const Constant(0))();  
  IntColumn get totalShortageItems \=\> integer().withDefault(const Constant(0))();  
  IntColumn get totalSurplusItems \=\> integer().withDefault(const Constant(0))();  
  IntColumn get totalLossAmount \=\> integer().withDefault(const Constant(0))(); // COGS  
  IntColumn get hlcPhysical \=\> integer()();  
  IntColumn get hlcLogical \=\> integer()();  
  TextColumn get hlcNode \=\> text()();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get completedAt \=\> integer().nullable()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('StocktakeLineEntity')  
class StocktakeLines extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get stocktakeId \=\> text().references(Stocktakes, \#id, onDelete: KeyAction.cascade)();  
  TextColumn get productVariantId \=\> text().references(ProductVariants, \#id)();  
  TextColumn get productNameSnapshot \=\> text()();  
  TextColumn get variantNameSnapshot \=\> text()();  
  IntColumn get systemQuantitySnapshot \=\> integer()();  
  IntColumn get physicalQuantity \=\> integer()();  
  IntColumn get varianceQuantity \=\> integer()(); // physicalQuantity \- systemQuantitySnapshot  
  IntColumn get unitCostPriceSnapshot \=\> integer()();  
  IntColumn get varianceAmount \=\> integer()(); // varianceQuantity \* unitCostPriceSnapshot  
  TextColumn get notes \=\> text().nullable()();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

// \==========================================  
// 9\. HARDWARE PRINT QUEUE & GOVERNANCE  
// \==========================================

@DataClassName('ReceiptDocumentEntity')  
class ReceiptDocuments extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get saleId \=\> text().references(Sales, \#id)();  
  TextColumn get formattedPlainText \=\> text()();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('PrintQueueEntity')  
class PrintQueue extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get receiptDocumentId \=\> text().references(ReceiptDocuments, \#id, onDelete: KeyAction.cascade)();  
  TextColumn get printerMacAddress \=\> text().nullable()();  
  BlobColumn get receiptPayload \=\> blob()();  
  IntColumn get retryCount \=\> integer().withDefault(const Constant(0))();  
  IntColumn get maxRetries \=\> integer().withDefault(const Constant(5))();  
  TextColumn get status \=\> text().withDefault(const Constant('PENDING'))(); // 'PENDING', 'PRINTING', 'FAILED', 'COMPLETED'  
  TextColumn get lastError \=\> text().nullable()();  
  IntColumn get createdAt \=\> integer()();  
  IntColumn get lastAttemptAt \=\> integer().nullable()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

@DataClassName('AuditEventEntity')  
class AuditEvents extends Table {  
  TextColumn get id \=\> text()();  
  TextColumn get actorUserId \=\> text().references(Users, \#id)();  
  TextColumn get actionType \=\> text()();  
  TextColumn get entityName \=\> text()();  
  TextColumn get entityId \=\> text()();  
  TextColumn get payloadJson \=\> text()();  
  IntColumn get hlcPhysical \=\> integer()();  
  IntColumn get hlcLogical \=\> integer()();  
  TextColumn get hlcNode \=\> text()();  
  IntColumn get createdAt \=\> integer()();

  @override  
  Set\<Column\> get primaryKey \=\> {id};  
}

### **4.3 Database Initializer & System Seed Data (lib/data/local/catatniaga\_database.dart)**

import 'package:drift/drift.dart';  
import 'connection/native\_connection.dart';  
import 'tables/master\_tables.dart';

part 'catatniaga\_database.g.dart';

@DriftDatabase(tables: \[  
  Businesses,  
  Users,  
  Shifts,  
  CustomerTypes,  
  StockLossReasons,  
  ExpenseCategories,  
  Categories,  
  Products,  
  ProductVariants,  
  Customers,  
  CustomerLedgerEntries,  
  StockMovements,  
  StockLossRecords,  
  Expenses,  
  Sales,  
  SaleLines,  
  SalePayments,  
  Stocktakes,  
  StocktakeLines,  
  ReceiptDocuments,  
  PrintQueue,  
  AuditEvents,  
\])  
class CatatNiagaDatabase extends \_$CatatNiagaDatabase {  
  CatatNiagaDatabase(\[QueryExecutor? executor\]) : super(executor ?? openCatatNiagaConnection());

  @override  
  int get schemaVersion \=\> 1;

  @override  
  MigrationStrategy get migration \=\> MigrationStrategy(  
    onCreate: (m) async {  
      await m.createAll();  
      await seedSystemDefaultData();  
    },  
    beforeOpen: (details) async {  
      await customStatement('PRAGMA foreign\_keys \= ON;');  
    },  
  );

  /// Seeds default Customer Types, Stock Loss Reasons, and Expense Categories  
  Future\<void\> seedSystemDefaultData() async {  
    final nowMs \= DateTime.now().toUtc().millisecondsSinceEpoch;

    // 1\. Seed Default Customer Types  
    await batch((b) {  
      b.insertAll(customerTypes, \[  
        CustomerTypesCompanion.insert(  
          id: 'ct-normal',  
          code: 'NORMAL',  
          name: 'Normal',  
          discountPercentage: const Value(0.0),  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
          updatedAt: nowMs,  
        ),  
        CustomerTypesCompanion.insert(  
          id: 'ct-warung',  
          code: 'WARUNG',  
          name: 'Warung',  
          discountPercentage: const Value(5.0),  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
          updatedAt: nowMs,  
        ),  
        CustomerTypesCompanion.insert(  
          id: 'ct-langganan',  
          code: 'LANGGANAN',  
          name: 'Langganan',  
          discountPercentage: const Value(2.0),  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
          updatedAt: nowMs,  
        ),  
      \]);  
    });

    // 2\. Seed Default Stock Loss Reasons  
    await batch((b) {  
      b.insertAll(stockLossReasons, \[  
        StockLossReasonsCompanion.insert(  
          id: 'slr-expired',  
          code: 'EXPIRED',  
          displayNameId: 'Kadaluwarsa / Tanggal Lewat',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
        ),  
        StockLossReasonsCompanion.insert(  
          id: 'slr-damaged-handling',  
          code: 'DAMAGED\_HANDLING',  
          displayNameId: 'Barang Rusak / Pecah / Bocor',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
        ),  
        StockLossReasonsCompanion.insert(  
          id: 'slr-damaged-pest',  
          code: 'DAMAGED\_PEST',  
          displayNameId: 'Barang Digigit Hama / Tikus',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
        ),  
        StockLossReasonsCompanion.insert(  
          id: 'slr-shrinkage-theft',  
          code: 'SHRINKAGE\_THEFT',  
          displayNameId: 'Barang Hilang / Selisih Stok Opname',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
        ),  
        StockLossReasonsCompanion.insert(  
          id: 'slr-owner-draw',  
          code: 'OWNER\_CONSUMPTION',  
          displayNameId: 'Konsumsi Pribadi / Diambil Pemilik',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
        ),  
      \]);  
    });

    // 3\. Seed Default Expense Categories  
    await batch((b) {  
      b.insertAll(expenseCategories, \[  
        ExpenseCategoriesCompanion.insert(  
          id: 'ec-salary',  
          code: 'SALARY',  
          name: 'Gaji Pegawai / Staf',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
          updatedAt: nowMs,  
        ),  
        ExpenseCategoriesCompanion.insert(  
          id: 'ec-electricity',  
          code: 'ELECTRICITY',  
          name: 'Listrik / Token PLN',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
          updatedAt: nowMs,  
        ),  
        ExpenseCategoriesCompanion.insert(  
          id: 'ec-water',  
          code: 'WATER',  
          name: 'Air / PDAM',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
          updatedAt: nowMs,  
        ),  
        ExpenseCategoriesCompanion.insert(  
          id: 'ec-dues',  
          code: 'DUES',  
          name: 'Iuran Kebersihan / Keamanan / RT',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
          updatedAt: nowMs,  
        ),  
        ExpenseCategoriesCompanion.insert(  
          id: 'ec-rent',  
          code: 'RENT',  
          name: 'Sewa Tempat / Kios',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
          updatedAt: nowMs,  
        ),  
        ExpenseCategoriesCompanion.insert(  
          id: 'ec-other',  
          code: 'OTHER\_OPERATIONAL',  
          name: 'Pengeluaran Operasional Lainnya',  
          isSystemDefault: const Value(true),  
          createdAt: nowMs,  
          updatedAt: nowMs,  
        ),  
      \]);  
    });  
  }  
}

## **5\. Atomic Data Access Objects (DAOs)**

### **5.1 POS Sales DAO (lib/data/local/dao/sales\_dao.dart)**

import 'package:drift/drift.dart';  
import '../catatniaga\_database.dart';  
import '../tables/master\_tables.dart';

part 'sales\_dao.g.dart';

@DriftAccessor(tables: \[Sales, SaleLines, SalePayments, ProductVariants, StockMovements, Shifts, ReceiptDocuments, PrintQueue\])  
class SalesDao extends DatabaseAccessor\<CatatNiagaDatabase\> with \_$SalesDaoMixin {  
  SalesDao(CatatNiagaDatabase db) : super(db);

  /// Complete Checkout atomically inside a single SQLite transaction  
  Future\<void\> executeCheckoutTransaction({  
    required SalesCompanion sale,  
    required List\<SaleLinesCompanion\> lines,  
    required List\<SalePaymentsCompanion\> payments,  
    required List\<StockMovementsCompanion\> stockMovements,  
    required ReceiptDocumentsCompanion receiptDoc,  
    required PrintQueueCompanion printJob,  
    required int cashReceivedForShift,  
  }) async {  
    return transaction(() async {  
      // 1\. Insert Sale Header  
      await into(sales).insert(sale);

      // 2\. Insert Sale Lines  
      for (final line in lines) {  
        await into(saleLines).insert(line);  
      }

      // 3\. Insert Payment Records  
      for (final payment in payments) {  
        await into(salePayments).insert(payment);  
      }

      // 4\. Update Inventory & Record Stock Movements  
      for (final movement in stockMovements) {  
        await into(stockMovements).insert(movement);

        await (update(productVariants)..where((pv) \=\> pv.id.equals(movement.productVariantId.value)))  
            .write(ProductVariantsCompanion(  
          stockQuantity: Value(movement.resultingStockQuantity.value),  
        ));  
      }

      // 5\. Enqueue Document & ESC/POS Print Job  
      await into(receiptDocuments).insert(receiptDoc);  
      await into(printQueue).insert(printJob);

      // 6\. Update Active Shift Expected Cash Balance  
      if (cashReceivedForShift \> 0\) {  
        final shiftId \= sale.shiftId.value;  
        final currentShift \= await (select(shifts)..where((s) \=\> s.id.equals(shiftId))).getSingle();  
          
        await (update(shifts)..where((s) \=\> s.id.equals(shiftId))).write(ShiftsCompanion(  
          expectedCashAmount: Value(currentShift.expectedCashAmount \+ cashReceivedForShift),  
          totalCashSales: Value(currentShift.totalCashSales \+ cashReceivedForShift),  
        ));  
      }  
    });  
  }  
}

### **5.2 Stock Loss DAO (lib/data/local/dao/stock\_loss\_dao.dart)**

import 'package:drift/drift.dart';  
import '../catatniaga\_database.dart';  
import '../tables/master\_tables.dart';

part 'stock\_loss\_dao.g.dart';

@DriftAccessor(tables: \[StockLossRecords, StockMovements, ProductVariants\])  
class StockLossDao extends DatabaseAccessor\<CatatNiagaDatabase\> with \_$StockLossDaoMixin {  
  StockLossDao(CatatNiagaDatabase db) : super(db);

  /// Record Stock Write-Off atomically  
  Future\<void\> recordStockLossTransaction({  
    required StockLossRecordsCompanion lossRecord,  
    required StockMovementsCompanion stockMovement,  
    required int newStockBalance,  
  }) async {  
    return transaction(() async {  
      await into(stockLossRecords).insert(lossRecord);  
      await into(stockMovements).insert(stockMovement);  
      await (update(productVariants)..where((pv) \=\> pv.id.equals(lossRecord.productVariantId.value)))  
          .write(ProductVariantsCompanion(  
        stockQuantity: Value(newStockBalance),  
      ));  
    });  
  }  
}

### **5.3 Operational Expenses DAO (lib/data/local/dao/expenses\_dao.dart)**

import 'package:drift/drift.dart';  
import '../catatniaga\_database.dart';  
import '../tables/master\_tables.dart';

part 'expenses\_dao.g.dart';

@DriftAccessor(tables: \[Expenses, Shifts\])  
class ExpensesDao extends DatabaseAccessor\<CatatNiagaDatabase\> with \_$ExpensesDaoMixin {  
  ExpensesDao(CatatNiagaDatabase db) : super(db);

  /// Record Expense & sync with Cash Drawer Shift  
  Future\<void\> recordExpenseTransaction({  
    required ExpensesCompanion expense,  
    required bool isPaidFromCashDrawer,  
  }) async {  
    return transaction(() async {  
      await into(expenses).insert(expense);

      if (isPaidFromCashDrawer && expense.shiftId.value \!= null) {  
        final shiftId \= expense.shiftId.value\!;  
        final currentShift \= await (select(shifts)..where((s) \=\> s.id.equals(shiftId))).getSingle();  
        final expenseAmount \= expense.amount.value;

        await (update(shifts)..where((s) \=\> s.id.equals(shiftId))).write(ShiftsCompanion(  
          expectedCashAmount: Value(currentShift.expectedCashAmount \- expenseAmount),  
          totalCashExpenses: Value(currentShift.totalCashExpenses \+ expenseAmount),  
        ));  
      }  
    });  
  }  
}

## **6\. Stock Opname (Stocktake) & Reporting Engine Specification**

### **6.1 Core Rules & Variance Invariants**

* ![][image11]**Shortage / Selisih Minus (![][image3]):**  
  * Buat pergerakan stok minus STOCKTAKE\_ADJUSTMENT.  
  * Otomatis insert StockLossRecord dengan alasan SHRINKAGE\_THEFT ('Barang Hilang / Selisih Stok Opname').  
  * Valuasi Kerugian: ![][image12].  
* **Surplus / Selisih Plus (![][image4]):**  
  * Buat pergerakan stok plus STOCKTAKE\_ADJUSTMENT. Tidak ada record kerugian.

### **6.2 Atomic Stocktake DAO (lib/data/local/dao/stocktake\_dao.dart)**

import 'package:drift/drift.dart';  
import '../catatniaga\_database.dart';  
import '../tables/master\_tables.dart';

part 'stocktake\_dao.g.dart';

@DriftAccessor(tables: \[Stocktakes, StocktakeLines, ProductVariants, StockMovements, StockLossRecords\])  
class StocktakeDao extends DatabaseAccessor\<CatatNiagaDatabase\> with \_$StocktakeDaoMixin {  
  StocktakeDao(CatatNiagaDatabase db) : super(db);

  /// Completes a Stock Opname session atomically inside a single SQLite transaction  
  Future\<void\> executeStocktakeTransaction({  
    required StocktakesCompanion stocktake,  
    required List\<StocktakeLinesCompanion\> lines,  
    required List\<StockMovementsCompanion\> stockMovements,  
    required List\<StockLossRecordsCompanion\> autoLossRecords,  
  }) async {  
    return transaction(() async {  
      await into(stocktakes).insert(stocktake);

      for (final line in lines) {  
        await into(stocktakeLines).insert(line);

        final variantId \= line.productVariantId.value;  
        final physicalQty \= line.physicalQuantity.value;

        await (update(productVariants)..where((pv) \=\> pv.id.equals(variantId))).write(  
          ProductVariantsCompanion(  
            stockQuantity: Value(physicalQty),  
            updatedAt: Value(DateTime.now().toUtc().millisecondsSinceEpoch),  
          ),  
        );  
      }

      for (final movement in stockMovements) {  
        await into(stockMovements).insert(movement);  
      }

      for (final lossRecord in autoLossRecords) {  
        await into(stockLossRecords).insert(lossRecord);  
      }  
    });  
  }  
}

### **6.3 Stock Accuracy Rate & Analytical SQL Aggregations**

![][image13]extension StocktakeReportsExtension on CatatNiagaDatabase {  
  /// Aggregates Stock Opname statistics over a date range  
  Future\<StockOpnameSummaryReport\> getStockOpnameSummary({  
    required int startDateMs,  
    required int endDateMs,  
  }) async {  
    final query \= customSelect(  
      '''  
      SELECT   
        COUNT(id) AS total\_count,  
        COALESCE(SUM(total\_items\_audited), 0\) AS sum\_audited,  
        COALESCE(SUM(total\_shortage\_items), 0\) AS sum\_shortage,  
        COALESCE(SUM(total\_surplus\_items), 0\) AS sum\_surplus,  
        COALESCE(SUM(total\_loss\_amount), 0\) AS sum\_loss  
      FROM stocktakes  
      WHERE created\_at \>= :startDate AND created\_at \<= :endDate  
        AND status \= 'COMPLETED'  
      ''',  
      readsFrom: {stocktakes},  
      variables: \[  
        Variable.withInt(startDateMs),  
        Variable.withInt(endDateMs),  
      \],  
    );

    final row \= await query.getSingle();  
    return StockOpnameSummaryReport(  
      totalOpnameCount: row.read\<int\>('total\_count'),  
      totalItemsAudited: row.read\<int\>('sum\_audited'),  
      totalShortageQty: row.read\<int\>('sum\_shortage'),  
      totalSurplusQty: row.read\<int\>('sum\_surplus'),  
      totalFinancialLossAmount: row.read\<int\>('sum\_loss'),  
    );  
  }  
}

class StockOpnameSummaryReport {  
  final int totalOpnameCount;  
  final int totalItemsAudited;  
  final int totalShortageQty;  
  final int totalSurplusQty;  
  final int totalFinancialLossAmount;

  StockOpnameSummaryReport({  
    required this.totalOpnameCount,  
    required this.totalItemsAudited,  
    required this.totalShortageQty,  
    required this.totalSurplusQty,  
    required this.totalFinancialLossAmount,  
  });  
}

## **7\. Implementation Roadmap & Test Suite Blueprint**

### **7.1 Prompt Chaining Implementation Roadmap**

1. **Phase 1: Core Foundation & Drift Setup**  
   * Setup Flutter project structure, Riverpod providers, native connection setup with SQLite WAL PRAGMAs, and HLC clock utility.  
2. **Phase 2: Master Data & Seeders**  
   * Build CRUD for Customer Types, Expense Categories, and Stock Loss Reasons along with database seeders.  
3. **Phase 3: Inventory, Stock Loss & Expenses**  
   * Implement append-only stock movements, stock write-off loss logging, and operational expense workflows with cash drawer shift synchronization.  
4. **Phase 4: POS Engine & Checkout**  
   * Implement cart state management with dynamic customer tier discount calculations, payment split handlers, atomic checkout transaction execution, and persistent ESC/POS print spooler.  
5. **Phase 5: Stock Opname & Financial Reporting**  
   * Implement physical stock count UI, atomic stocktake execution, auto-generated shrinkage records, Profit & Loss reports, and XLSX export isolate workers.

### **7.2 Destructive Test Suite Cases (test/data/catatniaga\_integration\_test.dart)**

1. **Process Termination Test:** Kill app process mid-checkout execution and verify that SQLite rolls back cleanly without orphan rows.  
2. **Negative Cash Drawer Guard Test:** Attempt to log an expense exceeding current shift cash balance and verify requirement of Manager PIN approval.  
3. **COGS Valuation Verification:** Write off 10 expired items priced at Rp 15.000 retail with HPP of Rp 10.000, asserting total financial loss is exactly Rp 100.000.  
4. **Owner Draw Segregation Test:** Record an item under OWNER\_CONSUMPTION, verifying that stock is decremented but the loss amount is excluded from operational store expenses in P\&L reports.
