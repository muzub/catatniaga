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

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACsAAAAaCAYAAAAue6XIAAAC3ElEQVR4Xu2W3YuMURzHn6cdWm9pMcbO2/PMi4YhZC6UUvJS64JkJUVRCjcSN+wmaUuuXKwLbjYvW2g3tSmSvWCjUPwFXCmRW8kFtXy+O+fZjqMZm10zN/Otb+f8zvec8/ud33NeHs9roYX6iMfj88Mw3JzNZvcGQbCSpja1JxKJealUKu10bw4IrkxwL+FXgh2iPA0HqY+iraL+iHKbO66hqFQqswjkHPxOYGfT6fQcW89kMpvQvsAPTc2sAiVb1wjkB+x2dYEFtKM9FFV39YYB58cJ4idlD6bv6hHQb9Gv121vGPjcRQL4CN8lk8mMq9ugz0BT9yvZuqCswouu5iKfzy/UlnHbGwJdTwQ5BsebmrGpgCA74Xv4iWDzrv4/gb+tYmSXy+XZdb+aFazY6eo2zCHc6Lb/I2LadgS73bLvYu/8rZcNxA46vf5bsNyri9GvFwqFpa42E8jlcomg+hDppawNOlyG42Suy9UM/KD6ik3ev9TXs9D7lDtk6+Ax/pjOQLFYjFPvRz9BEGvoMwBvRw8J2jLsq7DPPERH4Aj8zJgbGhP5+QOIAZ3eMskzTWRresXQztN+yjP3r7JrtkQ35RBNbTqccojdTv2gXjvNaRboUz8jqk6fw/hci/ZCvjWnFop9ZdJxPdA51GD4jUlvYh6C/djP4RbPeihYwCItSllAO6A2yl7R6MvRu+CoMq2x6quAqMeks5hd2MOyRcbeg3siH1OBVi3sFpmw4Jk/LRc4TGkhBJEPq8/wiLIb6cpiYO5tsx91PUb7UcHdYdx+GTxES7CfWvrMgmAqTP6gVCot0F6k/grnG/RFkGPKmrKnvlEW4TrG7VNQ6k/7auyjZss81lzaQmg5x930YA7UMJOfhJdw8Eal9qLJpDI14VTBUh9UtjUOvYT9BPYxx4ps9bdzjHpPWPuQTxs6WB2Uvk61fspNu2/VJ8CnnutZW0pbR6xlt9BCCw3GL03vr9SqkquiAAAAAElFTkSuQmCC>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEcAAAAaCAYAAADloEE2AAADxUlEQVR4Xu2Yf2iNURjH39umJr8SM/br3Lutlh/5Z0mK8iNSDI20mvKHWpplpYU/pCQlIpb8oVjUUJbUWtsfK7TSovyFlchoJUryB4VmPo/3vOvsce+7917aGvdb396953nOOc/5nuc8573zvCyyyGIcESstLa2H67QhImIlJSWF9K+Ox+PbYFzatNOkBAtbYoz5CJ+Ul5fP0/YQ5CDITvgcPa7CWsbYI+PAB1akiUKM6ZcTRwu8CDfRlqOdxkKMjmfhDzjMgHu1QzKUlZXNQpSb+D/UIoiNsXpgf1FRUbFrGyfISTgI7xNDghjm8GyDl6qqqqZo55RggMV06oB1IpAsVgbTfi7s4rskYxKJhNF2AeNstII3alumKCgomBZlccRVxbzveK502spoey1xub5h+JU1dNxVXFw8lWe3ZI+8a0cHsivn8BviuV4bA9gAP8NOAsrT9nTABlQyzi14vaKiIl/bNfA7IULABUFbZWXlDN57iavVi1IPEaSCDreDTHF2u0+yQ/sLqE+rsH+FHSKotgdwxLmXn58/XdsjIKgZd+GFwsLCEu2QDLIR+HdqcSQGiQU+IrbZbp9kkMmPuVniDCC1p9bxDZCL7UaIfQSIuAy/LxmIk0OftbAXnh7riGs4a0glzqj2pMAhAdv15LTVmBTZQ6YU0T4A38oZdm0axq9hw7DNi5LGiILgW2VeeITjM1M7RIEs3AqQuTgs7mg8yc1ki60EKALVuLY0jorUpVYTIcOkwIoPvo/p0yRFV/ukA+pTAWO91CJEFsf4WdOZ6pvGBjusRXDECc0GO/6gifDdhM8aOCAbFVbDoiKVCKnafwOLPGRCrlg3e6RIB+28L4QfYE/IDssNeBwOUXe2aGMyqOw5mOmRspC62K5FcMTplZvL7TACqfoEcEfST9tcSKE2/rXeHeyoI1q/9LfvV+AL/PZ7vjDb4XdZpLyPHnVMBHXnkcmgGAewmy+buDBoY91zeX8KW1zfUcDYCL8xwJsw4vNexDEqeyQbgsXzbMa23PNvvsO875OgsDV5GXyqOxh1jZuwY5AE9hNl0K139hNE1rTC9R2BLVb9dtHp0P2mkWK72fg78wk2G//3VB/BPMS2yPrlWv4JYsS8lLG74GWY0A6pYPxb9xXx1BPXbv5+Bhu89LM5fTBhHlzN5DvYlQ08TzJ5nTXLzktAkRczFmQseEqyQttSQY4lcVULMz2ifwUEfsYexwZ4DXHOe+OxS5MBCHLA2GNYGvKD9L8EmTLf+D/4TkzQvymyyCKLLLL4F/AT6eI3MiQQdA4AAAAASUVORK5CYII=>

[image4]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEcAAAAaCAYAAADloEE2AAADwElEQVR4Xu2Yf2iNURjH39umCEnMZbt3595N3Yb8c9NSlBIRk0ZaTflDLc2itMZfSlr4g1h+FM2iFmVJzdrKapi05l+sRH60EiX5g0Izn8d73tu5x/3x3pmtcb/17d17nuc87/N8z/Oe8945Th555DGBCJSWltbBtbbBJwLhcLiY+VWRSGQrjMiY7TQlQWHLlFIf4ePy8vIFtj0DChBkB3yGHldgDTF2Sxz4UIs0WQjw+EryaIHn4SbGCmynbAgw8RT8AUcJuMd2SIWysrI5iHId/0FbBLERqxcOlZSUhEzbBEHehCZ4jxyi5DCPazu8GI/Hp9nOaUGApUzqhLUikBQrwWw/E7r4bumYaDSqbLuAOBu04A22zQR+04X2+J+AvOI89x3XVcZYGWOvJS/TNxN+dQ0Td4ZCoRlce6R75N52NCCrchq/Ea7rbKMHneBn2JWpeJ30HXxOZFsUvyBeswgBF3ljsVhsNvf9PK/N8bMfIshiJtz0kjJWe0C6w/YXsD+txv4Vdoqgtt2DIc7doqKiWbbdQoAOXI5vN2yFUdvBL2QhZEFscSQHyQU+Ire55pxUkA3riNklRgDZe2oMXw+F2K5lsCeAiCvw++JTnAQQKcacG0L527Zng1FDOnGSxlMChyjssFuZsWqVpnvolBLGX8G38jqYNhvK3cNGYbvjp40t6PxaWYT7sNLxGUMK1wKMXRyKOxxJcTLpzXZAuQJVm7YcXhXZl9qUjw7LBl3sOb8i0W1B/F/YIvgWR7mr0pXum0YKksJsEQxxMnaDjj+scv9uSgnpbp59gXgPyC1i202kEyHd+G/gQQdVhiPW7B7ZpL1x7ivgB9gbDAZnmnMMyAl4FI6w72yxjblAioBnYZ+frtGQfbHDFsEQp19OLnNCAsXFxWHEuSXtZ9tMyEat3GO9xzuVDNGGZL6+vwyf47fPcYXZBr9z3yT3yVH9QbmddwnellPMyTGOXnxZxApvjLrnc/8Etpi+ScDYAL8R4E0m4vNeua9WUvdIN3jFc230VpTrIe73SlLY9ju5f6oHRAg1Dse5/kQZNvc7/QkiNa00fRPQm9WQLjoXmt80stluVu7KfIKNyv09NUAyg9iWaL9CzWwQYeU3UJ9yX6H0+0EOUO6p+5J86oi/i7+fwnonxy4cEyLuZ/8aHr6dVVnP9TgPr9VmKVgSyrr6+Gxk7jH7k2I8oDfyKuHfiO8bFHlSv4718CrinHEmYpWmAhDkgNKvYWmGH6T/JeiUhcr9wdc8Sf+myCOPPPLI41/AT+DjNdrI2jUDAAAAAElFTkSuQmCC>

[image5]: <data:image/png;base64,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>

[image6]: <data:image/png;base64,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>

[image7]: <data:image/png;base64,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>

[image8]: <data:image/png;base64,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>

[image9]: <data:image/png;base64,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>

[image10]: <data:image/png;base64,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>

[image11]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAmwAAAAuCAYAAACVmkVrAAAFaklEQVR4Xu3cXYhUZRzH8VnWyt4oqW1zdmbPmd2tZdeoaFOwDCosktqwrQtlL6QuLCoIQugFCiEkbwITUljEpYvFECMiKhAxrC7EuhGUgm4keoFAvNEgxOz3a55nO/s4I7k7RrN+P/BwzvN/nj1ve/PjnDmnVAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD+N7Is21Cr1W5P60UjIyOX9fb2LqlWqw/39fVdl463SIe2X87zfLXX08F2pnNaqOv3lJfpGAAAwHlVKpUrFdj+VDuTjkUa+8HzYl/hbpnCx/binLnS9j5QmMljX/vcp/ZdYUpb0jms0rm9pNXO0H9S7XQyDQAAoDmFh10KYN0OR9Vq9b7iWH9//02qH/GyWDfVz6a1WepQoNmi9lCxqP6I9nGqne9I+Y6kzuGTtK5zel9jK9M6AABAQwoUO8PyRYe3ZOytZsGsWf1COSRqW38U7+BZDGxdXV3XFOvtwo+QFcx2p0HUQmAbTesAAADnUCBa7mBR6DuELSj21fbEfsGCRoFNoWtAQeTHZi2db9rOMbVfG9THwz7a8rdsOt9XGl0j1Rep/o2C6tJ0DAAAYAaHtSy5o5bn+UbVdpRCaAuBbaw4xzRvjYLHlrQ+G9r+KbWpBvWf1I6k9UjHsDetnY+Od31aS5XL5Ru1zy/SeqSxiUZN296s47m+ONd30TT2e7EW6r7G5wS5f8Mve/jOY1oHAADzlELDNoWHR5JaTe1EtVq9LfTPqr1cnNPT03OD/u6QlpViPejU/MXNWjrZVD+uti8pd6h2RsfxeFKfpvFNaW2uFIZWqB1I67Oh49vq69eg/suFhs2oVqvd4VCZ1gEAwPzkQHQyfWTp5pChdsKT1N/udbV3wosJ/k3bsVY+znMo0zZPq21Qt0Nh5lWtHy+Ftypd86dEvN/BwcFrfYepUqn0aN5aP3Z00/pzGr9Lx6hF9pWDjZbbfNxqz2j8Ti2Xe2Nafqr+/VoeVXvP29fy9VL9Me8etfGw3znxb+8czLS9B/zY2SFY6x+qlsc54Q3dCR+PP6vSW//sx0a1Z1Xfof7bqg9q/XPP13JrWI5p7GNfD63fo/XH/L/ytcnC4+UsCdoAAGCeUxAYdfBw4HA/fIdt+rdureDQ4mCmfW0O/XXaX83rqk06xIRAMuSQFuZ8Ge4G+k3TSdccdMLYWq3/7MDjEBTGFmveCq+H/pSDlZafua+xA2qL4ngraN+5g5jaSm/bfXk6juv479b+T3rd5+c5WQidGuvX+m9ZPXgOZeFxrZbfK+zdErdR+LtNvja+bvHaAQCAS4gfhSoEHHZw0vJgOt4qIZw8r/28WwovHKi/33f1QlBbqLHdof6RA08IXf58hu+Wjef1O2h+DLsrhJev1V7wHbj4uFfbWuLthTtfB/U3a7L6Y8xV0wfTQgq5t2rbO9W+LTxyPqxjGNW+X3Pf66X6OexX7eYQvsYcOH1ealNZ/doc8vl5bql+Z9B3BX13cq+Dofs612VaPjp9AAAA4NKgMHCv2vr0Exyt5KCSJb9P0z4nBwYGrgjdju7u7qu9Ui6Xr9Ki07/r8l0m/d2D8TMgDpjx74tvwuYzv+v292PXOO5HjLF2ETiITZSS7ef/vKzgR79Lfdzxt3vpJ03ieadj4TqUhoeHL4+10sU7DwAAgJl810lB5w21oXQs8t0rzXuzyYsQbSF8t22dzvOJdAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA/kt/AfR8IQ1bYKvrAAAAAElFTkSuQmCC>

[image12]: <data:image/png;base64,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>

[image13]: <data:image/png;base64,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>