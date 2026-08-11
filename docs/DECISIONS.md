# Decision Log

## How to use

Record approved decisions that affect product behavior, schema, money, security, architecture, or implementation order. Never overwrite an old decision; add a new dated entry that supersedes it.

## D-001 — Source naming
- Status: Accepted
- Date: 2026-08-11
- Decision: The repository filename `prd.md` is the canonical PRD filename.
- Reason: The repository contains `prd.md`; agents must not expect `prd-2.md`.

## D-002 — Branch workflow
- Status: Accepted
- Date: 2026-08-11
- Decision: Implementation changes use feature branches and pull requests; `main` is not a direct coding branch.

## D-003 — MVP-first execution
- Status: Accepted
- Date: 2026-08-11
- Decision: Stabilize local SQLite transactions, inventory, POS, expenses, stocktake, and reports before optional cloud sync or external catalog features.

## Template
- ID:
- Status: Proposed / Accepted / Superseded
- Date:
- Decision:
- Evidence:
- Consequences:
- Reviewer:
