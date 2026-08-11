# Known Conflicts and Open Decisions

## C-001 — PRD filename
- Status: Resolved
- Repository file: `prd.md`
- Older reference: `prd-2.md`
- Resolution: Use `prd.md` everywhere.

## C-002 — Schema scope
- Status: Open
- Issue: The PRD contains a concrete Drift baseline while the ADD describes a broader future domain set.
- Risk: Implementing every future table immediately can overbuild the MVP and create migration churn.
- Recommendation: Implement the minimum MVP schema first; add broader tables only through a reviewed schema-change issue.
- Decision needed: Confirm the exact Phase 1 MVP table list.

## C-003 — State and approval policy
- Status: Open
- Issue: Some baseline examples default expenses and stock-loss records to APPROVED, while the architecture describes DRAFT/PENDING_APPROVAL workflows.
- Risk: Unauthorized records could be committed as approved.
- Recommendation: Use policy-driven state transitions in the application layer; never trust a status sent by UI.
- Decision needed: Confirm thresholds and required approver for each role.

## C-004 — Optional integrations
- Status: Open
- Issue: Google Drive, external product API, multi-device sync, and long-term archival are specified as optional or later capabilities.
- Recommendation: Keep them outside the MVP transaction path.
- Decision needed: Approve a separate post-MVP phase for each integration.
