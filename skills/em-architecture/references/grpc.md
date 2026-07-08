# gRPC — Gap Detection

## Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No TLS in production | `insecure` / `grpc.Dial` without TLS credentials | 🔴 |
| No auth interceptor | Server with no metadata-based auth validation | 🔴 |
| Auth in handler not interceptor | JWT validation inside individual RPC handlers (inconsistent, error-prone) | 🟠 |
| No input validation | Request message fields used without validation beyond protobuf defaults | 🟠 |
| gRPC port publicly exposed | Port 50051 open on public network interface | 🔴 |

## Performance Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No deadline on client call | `client.GetUser(req)` without a deadline/context timeout | 🟠 |
| Deadline not propagated | Service receives a deadline context but creates a new background context for downstream calls | 🟠 |
| New channel per request | `grpc.Dial(...)` inside a handler — expensive, causes connection exhaustion | 🔴 |
| L4 load balancer only | Service behind an L4 (TCP) load balancer — gRPC connections are sticky, no real balancing | 🟠 |
| Unary for large result set | Returning 10,000 records in a single unary response | 🟡 |

## Schema Quality Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Field number reused | Deleted field number used again in `.proto` — wire format corruption | 🔴 |
| No `reserved` on deleted fields | Deleted field numbers not reserved — can be accidentally reused | 🟠 |
| `string` for timestamps | `string created_at` instead of `google.protobuf.Timestamp` | 🟡 |
| First enum value not `UNSPECIFIED = 0` | Enum default (0) is a meaningful value | 🟡 |
| No `buf breaking` in CI | Breaking proto changes not caught before deployment | 🟠 |

## Generation Checklist
- [ ] TLS credentials on all client/server connections
- [ ] Auth validated in a server interceptor, not in handlers
- [ ] Deadline set on every client call; propagated to downstream calls
- [ ] Channel created once and reused (module-level or singleton)
- [ ] Deleted field numbers added to `reserved` statement
- [ ] `google.protobuf.Timestamp` for all date/time fields
- [ ] `buf breaking` check in CI pipeline
