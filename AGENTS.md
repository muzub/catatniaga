# CatatNiaga AI Agent Instructions

## Mission
Build CatatNiaga as an offline-first Android POS for Indonesian SMEs. Prefer correctness, auditability, security, and recoverability over speed or UI completeness.

## Source priority
1. Explicit owner decisions in the active issue or conversation.
2. The English AI Agent Implementation Playbook in docs/.
3. prd.md for product behavior and business acceptance criteria.
4. architecture-definition-document.md for system architecture and operational constraints.
5. Official framework/library documentation.
6. Agent assumptions only for minor implementation details; record them in docs/DECISIONS.md.

If sources conflict on money, schema, authorization, inventory, status, or reporting, stop and create a specification-conflict issue.

## Required workflow
Before coding, report TASK, PHASE, SCOPE, OUT_OF_SCOPE, FILES_EXPECTED, INVARIANTS, TESTS, RISKS, and BLOCKERS.
After coding, report DONE, FILES_CHANGED, TESTS_RUN, INVARIANTS_VERIFIED, MIGRATION_IMPACT, KNOWN_LIMITATIONS, and NEXT_RECOMMENDED_TASK.

Work on one issue and one phase at a time. Keep diffs small. Do not combine unrelated refactors with feature work.

## Hard guardrails
- Store all Rupiah values as integer minor units; never persist floating point money.
- Stock loss is valued using HPP/cost snapshots, never selling price.
- Final sales, payments, stock movements, stock losses, expenses, customer ledgers, and audits are append-only.
- Historical records must snapshot names, prices, HPP, discounts, and other reporting inputs.
- Checkout, stock loss, cash-drawer expense, and stocktake operations must be atomic and idempotent.
- Do not put SQL, Drift DAO access, or business rules in Widgets/Pages.
- UI must use the application/domain layers and Riverpod state management.
- Printer, network, camera, scanner, and cloud operations must never block a financial transaction.
- Printer jobs must be persistent and transaction-enqueued.
- Permissions must be enforced outside the UI. Cashier, Manager, and Owner limits must be tested.
- PINs and secrets must never be stored in plaintext.
- Sensitive exports, reset, restore, and deletion require authorization, confirmation, and audit events.
- Never disable lint, tests, security checks, or analyzer rules to make a build pass.

## Phase gates
Phase 0: repository baseline. Phase 1: core/database/HLC. Phase 2: identity/master data/shifts. Phase 3: inventory/loss/expenses. Phase 4: POS/payments/receipts. Phase 5: stocktake/reporting. Phase 6: backup/hardware/release. Phase 7: optional sync/catalog.

Do not advance a phase while its tests and gate are failing. Update docs/IMPLEMENTATION_STATUS.md after every accepted task.

## Stop conditions
Stop and escalate when a destructive migration, unclear financial formula, schema conflict, authorization ambiguity, secret/production credential, failed specification-dependent test, or guardrail bypass is involved.

## Required checks
Once the Flutter project exists, run: dart format --output=none --set-exit-if-changed ., flutter analyze, flutter test, and relevant integration tests. Never claim completion without recording the actual commands and results.
