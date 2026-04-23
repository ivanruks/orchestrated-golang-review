# Context Rules: Transactions Agent

When trigger seen in diff, load context described. **How**: File Access instructions from agent prompt; search repo when noted.

| Trigger | Load | Why |
|---|---|---|
| `tx.Begin()`/`db.BeginTx` | Full func + callers | `defer tx.Rollback()` exists? Commit on success? Rollback on ALL errors? |
| `UPDATE`/`INSERT` statement | Other writes to same table (search repo) | Concurrent modification, missing WHERE, consistency |
| Multiple state changes (DB + cache + queue) | Full handler/service method | Atomicity — if DB ok but cache fails, consistent? |
| External API call inside DB tx | Full tx scope | Can timeout holding DB lock — move outside or compensate |
| `DELETE` statement | Related INSERT/UPDATE + FK constraints (search repo) | Cascade effects, orphaned records |
| Redis/cache SET after DB write | Full function | DB ok but cache fails → stale data |
| Message publish (Kafka/RabbitMQ/NATS) after DB write | Full function | Outbox needed — message lost if publish fails after commit |
| Retry around state-changing op | Full function | Idempotent? Retry of non-idempotent → duplicate |
| `context.WithTimeout` near `tx.Begin`/`db.BeginTx` | Full func — rollback on ctx cancel | Ctx cancel ≠ auto-rollback — tx stays open, holds conn |
