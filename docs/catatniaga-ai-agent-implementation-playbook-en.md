# CatatNiaga AI Agent Implementation Playbook

**Document:** AI Agent Implementation Playbook & Guardrails  
**Version:** 1.0  
**Date:** 2026-08-11  
**Product:** CatatNiaga — Offline-First Android POS for Indonesian SMEs  
**Status:** Execution guide for AI coding agents

---

## 1. Document Purpose

This document translates the CatatNiaga PRD and Architecture Definition Document into operational instructions for an AI coding agent. The agent must work incrementally, produce small verifiable changes, and stop whenever it encounters a specification conflict, data risk, or decision that requires human approval.

This document does **not** replace the PRD or ADD. It is an execution-control layer that defines the implementation sequence, scope boundaries, phase gates, guardrails, Definition of Done, and recovery procedures.

### 1.1 Source-of-truth hierarchy

When sources differ, use this priority order:

1. Explicit decisions from the project owner in the active conversation or issue.
2. This document for execution order, guardrails, and acceptance gates.
3. `prd-2.md` for product requirements, user stories, business scenarios, Drift schema, DAOs, and the initial roadmap.
4. `architecture-definition-document.md` for architecture principles, module boundaries, security, backup, synchronization, hardware, and long-term operations.
5. Official documentation for the selected libraries and framework APIs.
6. Agent assumptions may only be used for minor implementation details and must be recorded in `docs/DECISIONS.md`.

If two high-priority sources conflict, **do not silently choose one**. Create a `SPEC_CONFLICT` record containing the conflict location, impact, resolution options, and recommendation; then stop the affected phase.

### 1.2 MVP scope

The MVP must prioritize the local workflows that deliver business value:

- Login/PIN and basic roles.
- Product, variant, price, cost, stock, category, and barcode management.
- Customer types and tier discounts.
- Shifts and POS checkout.
- Cash, QRIS, bank transfer, customer debt, and wallet payments according to the approved implementation scope.
- Stock movements, stock loss, expenses, and cash-drawer synchronization.
- Stocktake with automatic shrinkage records.
- Receipt documents and a persistent print queue.
- Basic sales, stock-loss, expense, and profit-and-loss reporting.

Features such as multi-device synchronization, Google Drive, external catalog import, 24-month archival, complex refunds/voids, or additional hardware integrations must not block the MVP. They belong in later phases after the local foundation and invariants have passed their gates.

---

## 2. Mandatory AI Agent Contract

Before writing code, the agent must read:

- the available PRD and ADD;
- the repository structure and `pubspec.yaml`;
- repository instructions such as `README`, `CONTRIBUTING`, and CI rules;
- the active issue or task;
- the output and status of the previous phase.

The agent must not start implementation based only on a prompt fragment before inspecting the repository context.

### 2.1 Required report before every task

Before changing files, the agent must write a short plan:

```text
TASK: <task title>
PHASE: <phase>
SCOPE: <what will be implemented>
OUT_OF_SCOPE: <what is intentionally excluded>
FILES_EXPECTED: <files likely to change>
INVARIANTS: <relevant business and architecture rules>
TESTS: <tests to create or run>
RISKS: <risks and mitigations>
BLOCKERS: <questions or specification conflicts>
```

If `BLOCKERS` contains a conflict affecting data, money, authorization, or migration, the agent must stop and request a decision.

### 2.2 Required report after every task

```text
DONE: <summary of changes>
FILES_CHANGED: <file list>
TESTS_RUN: <commands and results>
INVARIANTS_VERIFIED: <verified invariants>
MIGRATION_IMPACT: <none or explanation>
KNOWN_LIMITATIONS: <remaining limitations>
NEXT_RECOMMENDED_TASK: <next task>
```

The agent must not declare a task complete merely because the code compiles. Tests, invariant checks, and change review are also required.

---

## 3. Non-Negotiable Guardrails

### 3.1 Data and money

1. All Indonesian Rupiah amounts must be stored as `int`/`BIGINT` in Rupiah units, never as `double`, `float`, or formatted strings.
2. `double` may only be used temporarily for percentage calculations when absolutely necessary; the final result must be explicitly rounded and tested.
3. Prices, costs, discounts, quantities, totals, and balances must have boundary tests: zero, large values, 0% discount, and rounding cases.
4. Stock loss must never be calculated using the selling price. Valuation must use the HPP/cost-price snapshot.
5. Finalized sales, payments, stock movements, stock-loss records, expenses, customer ledgers, and audit events are append-only. Corrections must use an auditable reversal or adjustment rather than silently modifying history.
6. Every historical transaction must store the snapshots needed for future reporting: product/variant name, unit price, unit HPP, discount, and other relevant values at commit time.
7. No checkout operation may reduce stock without a consistent sale, or record a sale without a consistent stock movement.

### 3.2 Database and transactions

1. Local SQLite is the source of truth while offline.
2. SQL queries and Drift DAO access are forbidden inside Widgets/Pages.
3. UI code may communicate only through the appropriate provider, use case, repository, or service.
4. Checkout, stock loss, cash-drawer expenses, and stocktake must be atomic database transactions.
5. Every schema change must have a migration test from the previous database version.
6. Do not remove historical columns or tables without an approved migration, backup, and explicit sign-off.
7. Operations that may be retried after a crash must use an idempotency key or unique constraint.
8. Database startup must verify the baseline PRAGMAs: WAL, synchronous NORMAL, foreign keys ON, busy timeout, incremental vacuum, and cache size.

### 3.3 Offline-first and hardware

1. Sales must never wait for a Bluetooth printer, camera, scanner, network, or Google Drive.
2. A print job must be inserted into the persistent queue in the same transaction as the sale.
3. Printer failure must not cancel a sale that has already been committed.
4. Hardware and cloud integrations must use interfaces/adapters that can be mocked in tests.
5. The following core features must work without connectivity: customer selection, discount calculation, checkout, inventory, expenses, shifts, debt/wallet, receipt generation, and local backup.

### 3.4 Authorization and privacy

1. Permissions must be enforced in the domain/application layer, not only by hiding UI buttons.
2. Cashiers must not access database reset, PII export, sensitive backup/restore, or owner profit-and-loss reports.
3. Expenses exceeding drawer balance or the policy threshold must require Manager/Owner approval as defined by policy.
4. Stock loss requiring approval must not become approved merely because the UI submitted an approved status.
5. Generic exports must mask PII. Full exports are restricted to authorized roles and must create an audit event.
6. PINs must never be stored in plaintext. Use hashing and secure storage according to the security design.
7. Reset, restore, PII export, and data deletion operations must be double-confirmed, audited, and inaccessible to ordinary UI tests without an approval mock.

### 3.5 Code quality

1. Do not add a dependency without a documented reason, risk/size assessment, and task approval.
2. Do not use `dynamic` in financial domain calculations without a documented justification.
3. Do not disable analyzers, tests, lint, or security checks to make a build pass.
4. Do not combine a large refactor with a small feature task.
5. Each task must have one primary objective and a reviewable diff.
6. Do not create placeholders that appear complete. If an adapter is unavailable, use an explicit interface and `UnsupportedOperation` with a clear test.
7. Do not copy third-party branding, code, assets, schema, or proprietary behavior. The implementation must be original to CatatNiaga.

---

## 4. Implementation Phases and Gates

Each phase has outputs, scope boundaries, and a completion gate. The agent must not advance while the active phase gate is failing.

### Phase 0 — Reconnaissance and Baseline

**Goal:** Understand the repository and establish a buildable baseline.

**Work:**

- Inspect Flutter/Dart versions, package structure, build flavors, lint, analyzer, and test runner.
- Establish the minimum Clean Architecture structure: presentation, application, domain, data, and infrastructure.
- Create `docs/DECISIONS.md`, `docs/KNOWN_CONFLICTS.md`, and `docs/IMPLEMENTATION_STATUS.md`.
- Verify that a debug build and empty test suite can run.
- Establish UUID, clock, error model, result type, and dependency-injection patterns.

**Out of scope:** Major business features, cloud sync, real printer integration, and complex migrations.

**Gate:** The repository builds, analyzer runs, tests run, layer boundaries are agreed, and no unresolved blocker conflict remains.

### Phase 1 — Core Foundation, Database, and HLC

**Goal:** Provide a safe local data foundation.

**Work:**

- Set up Drift database, native connection, PRAGMAs, migration strategy, and seed framework.
- Implement the HLC utility and device/node identifier.
- Implement base errors, authorization policy, audit-event contract, and transaction helper.
- Add database integrity checks and foreign-key tests.
- Implement the minimum schema: businesses, users, shifts, customer types, loss reasons, expense categories, products, and variants.

**Gate:** Migration tests pass; all monetary values use integers; PRAGMA tests pass; HLC ordering is monotonic; seeding is idempotent; no DAO is used in UI.

### Phase 2 — Identity, Roles, Master Data, and Shifts

**Goal:** Make the application usable by the correct roles.

**Work:**

- Initial Owner setup and login/PIN.
- Owner, Manager, and Cashier roles with centralized policies.
- CRUD for products, categories, variants, customer types, expense categories, and loss reasons.
- Shift opening, active shift, expected cash, shift closing, and cash variance.
- Audit events for master-data changes and approvals.

**Gate:** The authorization matrix is tested in the application layer; cashiers cannot perform owner operations; soft deletion does not damage history; shift cash invariants are tested.

### Phase 3 — Inventory, Stock Loss, and Expenses

**Goal:** Keep inventory and operating costs auditable.

**Work:**

- Append-only stock movements.
- Stock write-off for EXPIRED, DAMAGED_HANDLING, DAMAGED_PEST, SHRINKAGE_THEFT, and OWNER_CONSUMPTION.
- HPP snapshot and total-loss calculation.
- Expense workflow, custom categories, attachment paths, approval, and cash-drawer synchronization.
- Negative-cash warning and Manager override.

**Gate:** Stock loss uses HPP; OWNER_CONSUMPTION is separated from operational loss/P&L; cash-drawer expense atomically reduces expected cash; crash and zero-stock tests pass.

### Phase 4 — POS, Pricing, Payments, and Receipt Queue

**Goal:** Complete the main sales workflow.

**Work:**

- Cart state using Riverpod.
- Customer-tier discounts: Normal 0%, Warung 5%, Langganan 2%, plus approved custom tiers.
- Snapshot customer-type name and discount percentage when the sale is committed.
- Split payments and debt/wallet rules.
- Atomic checkout: sale, lines, payments, stock movements, shift update, receipt document, print queue, and audit event.
- Held carts and idempotency protection.

**Gate:** The scenario with Minyak Goreng priced at Rp30,000 and a Warung discount produces a Rp1,500 discount and Rp28,500 total; retrying checkout does not duplicate the transaction; printer failure does not fail the sale; stock and cash match the correct payment.

### Phase 5 — Stocktake and Reporting

**Goal:** Produce inventory reconciliation and financial reports.

**Work:**

- Partial and full stocktakes.
- Variance = physical quantity - system snapshot.
- A shortage creates a negative stock movement and automatic `SHRINKAGE_THEFT` loss record.
- A surplus creates a positive stock movement without a loss record.
- Sales, COGS, expenses, stock loss, owner draw, net profit, and stock-accuracy reports.
- XLSX/PDF/export through workers without blocking the UI.

**Gate:** System stock 50, physical count 47, and HPP Rp3,100 produce shortage -3 and loss Rp9,300; owner draw does not reduce operational efficiency; reports read immutable ledgers and snapshots.

### Phase 6 — Resilience, Backup, Hardware, and Release

**Goal:** Prepare a recoverable APK distribution.

**Work:**

- Persistent printer worker and retry state machine.
- Local encrypted backup and restore.
- Optional encrypted Google Drive archive.
- SAF export/import.
- Android Keystore, integrity checks, diagnostics, archival strategy, and release build.
- Device/offline behavior tests and destructive integration tests.

**Gate:** A backup can be verified and restored into a test database; the print queue survives restart; restore cannot overwrite the active database without confirmation; release builds expose no secrets; APK smoke tests pass.

### Phase 7 — Optional Multi-Device Sync and External Catalog

Start this phase only after the MVP is stable. HLC merge, conflict handling, tombstones, sync cursors, external product API, and catalog bootstrap must remain separate from the core transaction path. External API prices must never become the selling price or HPP without user confirmation.

---

## 5. Required Implementation Patterns

### 5.1 Feature workflow

Every feature must follow this order:

1. Domain rule and invariant.
2. Entity/value object/error.
3. Repository interface.
4. Drift table/migration/DAO.
5. Use case/application service.
6. Provider/state model.
7. UI.
8. Unit, integration, authorization, failure, and offline tests.
9. Decision and status documentation.

Do not build the UI first and then place business rules inside widget callbacks.

### 5.2 Atomic transaction checklist

Before merging a transactional operation, the agent must answer:

- Which rows must succeed or fail together?
- What is the idempotency key?
- What happens if the process dies after the first row?
- Are stock, balances, ledgers, and audit events consistent?
- Is hardware or network called inside the transaction? If yes, this is incorrect and must be moved to a queue.
- Is retry safe?
- Does the history contain enough snapshots for future reports?

### 5.3 Required state machines

Use explicit states for risky workflows:

- Expense: `DRAFT -> PENDING_APPROVAL -> APPROVED`, or `REJECTED/VOIDED`.
- Stock loss: `DRAFT -> PENDING_APPROVAL -> APPROVED`, or `REJECTED`.
- Print job: `PENDING -> PRINTING -> COMPLETED`, or `FAILED`, with a retry limit.
- Stocktake: `DRAFT -> COMPLETED`, or `CANCELLED`.
- Sale: `DRAFT -> COMMITTED`; failure must not leave a half-committed state.

Transitions may only be performed by use cases that validate the actor, policy, and current state.

---

## 6. Definition of Done

A task is complete only when all applicable criteria are satisfied:

- Code is placed in the correct layer.
- Important business rules are not in UI code.
- The analyzer/linter has no new errors.
- Domain unit tests pass.
- Database integration tests pass.
- Authorization tests pass.
- Offline/failure-path tests exist for hardware- or network-dependent operations.
- Migration tests exist when the schema changes.
- Sensitive operations create audit events.
- Amounts and rounding are tested.
- Status and decision documentation is updated.
- The diff has been checked for secrets, sensitive debug logs, and unrelated changes.

### 6.1 Minimum test matrix

| Area | Minimum scenarios |
|---|---|
| Pricing | 0%, 2%, 5%, custom tier, rounding, customer changed before commit |
| Checkout | success, crash mid-transaction, retry, insufficient stock, held cart |
| Payment | cash, split, debt, wallet, change, invalid total |
| Inventory | sale, receiving, write-off, zero stock, negative-stock alert |
| Stocktake | shortage, surplus, zero variance, partial stocktake, duplicate completion |
| Expense | cash drawer, bank, owner pocket, over-limit, approval/rejection |
| Authorization | cashier/manager/owner for every restricted use case |
| Printer | offline, timeout, retry, max retry, restart persistence |
| Backup | create, corrupt archive, wrong password, restore confirmation |
| Reports | date range, HPP snapshot, owner-draw exclusion, empty dataset |

---

## 7. Stop Conditions and Escalation

The agent must stop and request a human decision when:

- Two definitions differ for a table, status, formula, or role permission.
- A destructive migration is required.
- The change affects money, HPP, debt, wallet, net profit, inventory, or stock calculation.
- The agent cannot determine whether an operation must be append-only or mutable.
- A secret, credential, signing key, OAuth token, or production access is required.
- A test fails because the specification is unclear rather than because of a local implementation bug.
- Someone requests disabling a guardrail, removing audit, bypassing approval, or storing PII/secrets insecurely.
- A new dependency has licensing, native-permission, or supply-chain risks that have not been approved.

Escalation format:

```text
BLOCKED: <title>
CONFLICT_OR_RISK: <problem>
EVIDENCE: <file/section/test>
IMPACT: <data, security, UX, schedule>
OPTIONS:
A. <option and consequences>
B. <option and consequences>
RECOMMENDATION: <recommended option>
DECISION_NEEDED: <specific answer required>
```

---

## 8. Agent Prompt

Use the following text as a system/developer instruction for the coding agent:

```text
You are an AI coding agent for CatatNiaga, an offline-first Android POS application.

CORE RULES:
- Read the PRD, ADD, and repository before coding.
- Work on one task and one active phase at a time.
- Follow Clean Architecture: UI -> Provider -> Use Case -> Domain/Repository -> DAO/Infrastructure.
- Never write SQL or access a DAO from a Widget or Page.
- Store all money as integer Rupiah; never use floating point for persistence.
- All financial and inventory transactions must be atomic, idempotent, auditable, and snapshot-based.
- Never wait for printers, networks, cameras, scanners, or cloud services inside business transactions.
- Never bypass authorization, approval, audit, encryption, or backup guardrails.
- Never silently modify append-only history.
- If the specification conflicts or affects data, money, or security, STOP and escalate.

BEFORE CODING:
1. Write the TASK REPORT: scope, out-of-scope, files, invariants, tests, risks, blockers.
2. Find related implementations and tests; do not create duplicate abstractions.
3. Determine migration impact.

DURING CODING:
1. Implement the domain rule first.
2. Add tests before or alongside the implementation.
3. Keep the diff small.
4. Never disable lint or tests to make the build pass.
5. Record assumptions in docs/DECISIONS.md.

AFTER CODING:
1. Run formatting, analyzer, unit tests, and relevant integration tests.
2. Run authorization and failure-path tests.
3. Review the diff for secrets, PII, business rules in UI, and unrelated changes.
4. Write the DONE REPORT with test results, invariants, migration impact, limitations, and the next task.
5. Do not declare completion while important tests fail or blockers remain unresolved.
```

---

## 9. Change and Status Management

Keep the following three working files in the repository:

### `docs/DECISIONS.md`

Record approved decisions, date, rationale, consequences, and affected tasks. Decisions involving money, schema, permissions, and security require human review.

### `docs/KNOWN_CONFLICTS.md`

Record unresolved conflicts. Each conflict must have an ID, sources, status, impact, options, and final decision. The agent must not resolve a conflict through a hidden assumption.

### `docs/IMPLEMENTATION_STATUS.md`

Use these statuses:

- `NOT_STARTED`
- `IN_PROGRESS`
- `BLOCKED`
- `IMPLEMENTED_UNVERIFIED`
- `VERIFIED`
- `DEFERRED`

A module may be marked `VERIFIED` only after the phase gate and relevant test matrix have passed.

---

## 10. Recommended Initial Task Sequence

1. Repository reconnaissance and baseline build.
2. CI lint/analyzer/test baseline.
3. Domain primitives: IDR amount, Result/Error, Clock/HLC, UUID.
4. Drift connection, PRAGMAs, schema version, and migration tests.
5. Idempotent seed data.
6. Authorization policy and audit contract.
7. User/PIN/roles and shifts.
8. Product/category/variant CRUD.
9. Stock movement ledger.
10. Atomic stock-loss and expense workflows.
11. Customer types, customers, debt, and wallet ledger.
12. Cart pricing engine.
13. Atomic checkout.
14. Receipt document and print queue.
15. Stocktake.
16. Reporting/P&L.
17. Local backup/restore.
18. Hardware adapter and worker.
19. Release hardening.
20. Optional synchronization and external catalog.

The agent must always choose the smallest task that unblocks the next task rather than attempting to build the entire application in one large change.

---

## 11. Closing Principle

CatatNiaga implementation priority is: **data correctness, security, auditability, and offline reliability before UI completeness**. No new feature may compromise transaction atomicity, HPP accuracy, separation of owner draw from operational loss, role control, or crash recovery.

When uncertain, the agent should choose the implementation that is easiest to audit, test, and roll back—or stop and request a decision.
