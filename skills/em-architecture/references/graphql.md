# GraphQL — Gap Detection

## Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Introspection public in production | `introspection: true` with no auth requirement | 🟠 |
| No query depth limit | No `depthLimit()` validation rule — nested query DoS | 🔴 |
| No query complexity limit | No complexity estimator — expensive query DoS | 🔴 |
| No rate limiting | No per-user or per-operation rate limit | 🔴 |
| Authorization only at gateway | Field resolvers have no auth checks — bypass possible | 🔴 |
| Error leaks internal detail | `extensions.exception.stacktrace` returned to client | 🟠 |

## Performance Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| N+1 without DataLoader | Resolver calls `db.find(parent.userId)` per parent — no DataLoader | 🔴 |
| No DataLoader caching | DataLoader instantiated per-request but not per-operation (no deduplication) | 🟠 |
| Resolver fetches full entity for one field | Resolver loads entire User to return just `user.name` | 🟡 |
| No `@cacheControl` on stable fields | Frequently-queried, rarely-changing fields with no cache hints | 🟡 |

## Schema Quality Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Undocumented types/fields | Type or field missing `"""description"""` | 🟡 |
| Mutation returns entity directly | `mutation createUser: User` — no payload wrapper, no error union | 🟡 |
| Nullable everything | Fields that are always present marked nullable — client forced to null-check everywhere | 🟡 |
| No versioning plan | Schema changes breaking clients with no deprecation strategy | 🟠 |
| `@deprecated` fields never removed | Deprecated fields accumulating with no removal timeline | 🟡 |

## Generation Checklist
- [ ] Depth limit: `depthLimit(7)` in validation rules
- [ ] Complexity limit with field-level cost estimators
- [ ] DataLoader for every resolver that loads a related entity
- [ ] Mutations return payload type with `errors: [UserError!]!`
- [ ] Field-level auth in every resolver for sensitive data
- [ ] Introspection disabled or auth-gated in production
- [ ] All types and fields documented with descriptions
