# fault-tolerant-payment-service

A payment ledger service that moves money between wallets. It supports three operations: deposits into a wallet, transfers between wallets, and holds that commit or expire on a TTL. Every operation appends to a double-entry ledger that is never rewritten.

Most payment bugs only show up when something fails at the worst possible moment. This service reproduces those moments on demand: nine named scenarios, each answered by one mechanism.

## What breaks payments

| Failure mode | Mechanism |
| --- | --- |
| **Double-spend race**: concurrent withdrawals drive a balance negative | `SELECT ... FOR UPDATE` row locks plus a `CHECK (balance >= 0)` constraint as the last line. Proven by `double-spend-100-concurrent`. |
| **Retry duplicate**: the same request arrives twice and must apply once | Idempotency keys claimed inside the money transaction, with Postgres as authority and Redis as fast path. Proven by `retry-storm` and `redis-blackout-retry-storm`. |
| **Crash dual-write**: the process dies between database commit and event publish | Transactional outbox, at-least-once relay, consumer dedup on `event_id`. Proven by `kill-after-commit-before-publish` and `kill-after-publish-before-mark`. |

Four more scenarios cover deadlock on crossed transfers, commit-versus-expiry races, tamper evidence, and rebuilding every balance from the ledger alone. Each one is mapped to its mechanism in [docs/technical-design.md](docs/technical-design.md#failure-mode--mechanism-map).

## Measured

Numbers below come from a committed run, not from estimates. Anything the scenarios cannot observe is marked `UNMEASURED` in the report rather than invented.

| Scenario | Result | Condition |
| --- | --- | --- |
| `double-spend-100-concurrent` | 0 invariant violations | 100 concurrent withdrawals against a single wallet |
| TODO-scenario-name | 235 transfers/sec at 0 errors | TODO: harness, duration, host spec |
| TODO-scenario-name | 30 ms median recovery | TODO: measured from what event to what event |

Full run: [docs/torture-report.md](docs/torture-report.md).

## Stack

| Component | Tech | Responsibility |
| --- | --- | --- |
| API | Java 17, Vert.x + Mutiny | HTTP surface, request validation, idempotency key intake |
| Money path | Blocking JDBC on worker threads | Row-locked balance mutation and ledger append in one transaction |
| Store | PostgreSQL 17 | Balances, double-entry ledger, idempotency keys, outbox |
| Fast path | Redis 7 | Idempotency lookup ahead of Postgres; never the authority |

Scope is deliberately narrow: one instance, no auth, no multi-currency, and events go through a pluggable publisher rather than a real broker. Delivery is at-least-once; exactly-once delivery is not claimed. The full list is at the end of [docs/technical-design.md](docs/technical-design.md#out-of-scope-deliberate).

## Run

```bash
docker compose up -d          # Postgres :5432, Redis :6379
mvn package
java -jar target/fault-tolerant-payment-service-0.1.0-SNAPSHOT.jar   # :8080
```

Smoke: `POST /wallets` → `POST /deposits` (with an `Idempotency-Key` header) → `GET /wallets/:id` → `GET /ledger/verify`. Full API surface in [docs/technical-design.md](docs/technical-design.md#http-surface).

## Torture suite

```bash
docker compose up -d
mvn -Ptorture verify          # 9 named scenarios, real process kills
```

Writes `target/torture-report.md` with pass/fail plus measured numbers per scenario.

## Docs

- [docs/business-flows.md](docs/business-flows.md) covers what the service does in domain terms, with no implementation detail.
- [docs/technical-design.md](docs/technical-design.md) covers the schema, transaction boundary, locking, outbox, and hash chain.