# CatatNiaga

CatatNiaga is an offline-first Android POS for Indonesian SMEs. The repository currently contains the product requirements, architecture baseline, and AI-agent execution playbook. Application implementation must follow the phase gates and guardrails in `AGENTS.md`.

## Read first

1. [AGENTS.md](AGENTS.md)
2. [PRD](prd.md)
3. [Architecture Definition Document](architecture-definition-document.md)
4. [AI Agent Implementation Playbook](docs/catatniaga-ai-agent-implementation-playbook-en.md)
5. [Implementation Status](docs/IMPLEMENTATION_STATUS.md)
6. [Decisions](docs/DECISIONS.md)
7. [Known Conflicts](docs/KNOWN_CONFLICTS.md)

## Current state

The repository is in the repository-bootstrap stage. The Flutter application, database implementation, tests, and CI for Flutter are not yet implemented. Do not generate a large application in one change. Start with Phase 0 and create an issue for each bounded task.

## Planned architecture

```text
Presentation -> Application -> Domain <- Data -> Infrastructure
```

Planned feature areas include identity/access, catalog, inventory, POS, payments, transactions, customer finance, expenses/shifts, reporting, receipts/hardware, backup/sync, and import/export.

## Development rules

- Use a feature branch and pull request; do not push directly to `main`.
- Every issue must identify phase, scope, invariants, tests, and migration impact.
- Every PR must pass the PR checklist and review gates.
- Financial, schema, authorization, and security changes require human review.
