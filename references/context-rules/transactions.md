# Context Rules: Transactions Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How | Why |
|---|---|---|---|
| `tx.Begin()` or `db.BeginTx` | Full function + all callers | Use File Access instructions from your prompt | Verify `defer tx.Rollback()` exists, Commit on success, rollback on ALL error paths |
| `UPDATE` or `INSERT` statement | Other writes to the same table(s) in the repo | Search repo for table name, then use File Access instructions | Check for concurrent modification, missing WHERE clause, consistency |
| Multiple state changes in one handler (DB + cache + queue) | Full handler/service method | Use File Access instructions from your prompt | Verify atomicity — if DB succeeds but cache fails, is state consistent? |
| External API call inside a DB transaction | Full transaction scope | Use File Access instructions from your prompt | External calls can timeout holding DB lock — move outside tx or add compensation |
| `DELETE` statement | Related INSERT/UPDATE and foreign key constraints | Search for table name, check migration files via File Access instructions | Cascade effects, orphaned records |
| Redis/cache SET after DB write | Full function | Use File Access instructions from your prompt | If DB write succeeds but cache SET fails, stale data is served |
| Message publish (Kafka/RabbitMQ/NATS) after DB write | Full function | Use File Access instructions from your prompt | Outbox pattern needed — message can be lost if publish fails after commit |
| Retry logic around state-changing operation | Full function | Use File Access instructions from your prompt | Verify idempotency — retry of non-idempotent op causes duplicate state |
| `context.WithTimeout` near `tx.Begin`/`db.BeginTx` | Full function — check rollback on context cancel | Use File Access instructions from your prompt | Context cancellation doesn't auto-rollback — transaction stays open holding connection |
