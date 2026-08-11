## Summary

- Phase:
- Issue:
- Scope:
- Out of scope:

## Verification

- [ ] `dart format --output=none --set-exit-if-changed .`
- [ ] `flutter analyze`
- [ ] `flutter test`
- [ ] Relevant integration tests
- [ ] Offline/failure path tested where applicable

## Guardrail review

- [ ] Rupiah values are integer-based
- [ ] HPP/cost snapshots are preserved
- [ ] Append-only history is not silently modified
- [ ] Financial and inventory operations are atomic/idempotent
- [ ] No SQL/DAO/business rules were added to UI
- [ ] Authorization is enforced in application/domain layers
- [ ] Sensitive operations create audit events
- [ ] No secrets or PII were committed

## Data and migration

- Migration impact: None / Describe below
- [ ] Migration test added if schema changed
- [ ] Backup/restore impact reviewed if applicable

## AI agent completion report

```text
DONE:
FILES_CHANGED:
TESTS_RUN:
INVARIANTS_VERIFIED:
KNOWN_LIMITATIONS:
NEXT_RECOMMENDED_TASK:
```
