# Lesson 026 — Redis Transactions

> **Section 03 — Keys, Memory and Internals** | Beginner → Advanced  
> **Goal:** Understand MULTI, EXEC, DISCARD, WATCH, optimistic locking, transaction limitations, and safe Node.js patterns.

**Previous:** [Lesson 025 — Pipelining](025-pipelining.md)  
**Next:** [Lesson 027 — Lua Scripting and Redis Functions](027-lua-functions.md)

## 1. Why do we need transactions?

Imagine an application transferring 10 points from user A to user B:

\`\`\`redis
DECRBY points:user:A 10
INCRBY points:user:B 10
\`\`\`

If these are sent separately, another client can run commands between them. A client or connection failure can also leave you uncertain about what executed.

A Redis transaction groups commands so Redis executes the queued operations **sequentially without other clients' commands interleaving during EXEC**.

\`\`\`mermaid
sequenceDiagram
    participant A as Client A
    participant R as Redis
    participant B as Client B
    A->>R: MULTI
    A->>R: DECRBY points:A 10
    A->>R: INCRBY points:B 10
    A->>R: EXEC
    Note over R: Execute queued commands together
    R-->>A: Results
    B->>R: GET points:A
    R-->>B: Updated value
\`\`\`

**Critical:** Redis transactions are not identical to SQL transactions. Redis does not provide automatic rollback of already executed commands when a runtime command fails.

## 2. MULTI, EXEC, and DISCARD

### MULTI — start queuing

\`\`\`redis
MULTI
SET demo:tx:name Asha
INCR demo:tx:count
EXEC
\`\`\`

After \`MULTI\`, Redis replies \`QUEUED\` to valid queued commands. They are executed when \`EXEC\` is received.

### EXEC — run queued commands

\`EXEC\` executes the queued operations as a non-interleaved sequence and returns their individual results.

### DISCARD — abandon the transaction

\`\`\`redis
MULTI
SET demo:tx:temporary hello
DISCARD
GET demo:tx:temporary
\`\`\`

The \`SET\` never executes because the transaction was discarded.

**Important:** \`DISCARD\` discards commands that are still queued; it does not undo a transaction that already ran.

## 3. What Redis transaction atomicity actually means

During \`EXEC\`, commands from other clients do not run between the queued transaction commands.

\`\`\`mermaid
flowchart TD
    A["Client A: MULTI"] --> B["Queue SET A"]
    B --> C["Queue INCR B"]
    C --> D["EXEC"]
    D --> E["Execute SET A"]
    E --> F["Execute INCR B"]
    F --> G["Return results"]
\`\`\`

This is an important **isolation** property.

However:
- A transaction can contain commands that fail at execution time.
- Other queued commands may still succeed.
- Redis does not automatically undo earlier successful commands.
- Commands queued in a transaction cannot provide their results to your application before \`EXEC\`.

Therefore, **MULTI/EXEC is not enough for arbitrary conditional read-modify-write logic**.

## 4. Transaction error types

Redis distinguishes errors that occur while **queuing** from errors that occur **during execution**.

### A. Queue-time errors

Example: a command has invalid syntax or the wrong number of arguments.

\`\`\`redis
MULTI
SET demo:a 1
SET demo:b
EXEC
\`\`\`

Redis detects the malformed command during queuing. Modern Redis aborts the transaction at \`EXEC\` if a queue-time error occurred.

### B. Runtime errors

Example:

\`\`\`redis
SET demo:number hello
MULTI
INCR demo:number
SET demo:after success
EXEC
\`\`\`

The \`INCR\` fails at execution time because the value is not an integer, but the later \`SET\` can still execute.

\`\`\`mermaid
flowchart TD
    A["MULTI"] --> B["Queue INCR on text"]
    B --> C["Queue SET another key"]
    C --> D["EXEC"]
    D --> E["INCR returns error"]
    E --> F["SET still executes"]
\`\`\`

**Key interview fact:** Redis transactions do **not** provide SQL-style rollback on runtime errors.

## 5. Why GET then SET can lose updates

Suppose two API servers both update a counter:

\`\`\`text
Initial balance = 100

Client A: GET balance -> 100
Client B: GET balance -> 100
Client A: SET balance 110
Client B: SET balance 120
Final balance = 120
\`\`\`

Client A's update was lost.

\`\`\`mermaid
sequenceDiagram
    participant A as Client A
    participant R as Redis
    participant B as Client B
    A->>R: GET balance
    R-->>A: 100
    B->>R: GET balance
    R-->>B: 100
    A->>R: SET balance 110
    B->>R: SET balance 120
    Note over R: Lost update
\`\`\`

### First solution: a single atomic command

If you only need an increment:

\`\`\`redis
INCRBY balance 10
\`\`\`

Prefer a suitable single Redis command over a complicated transaction.

## 6. WATCH — optimistic concurrency control

\`WATCH\` lets Redis detect whether watched keys changed before your transaction executes.

The pattern is:

1. \`WATCH\` the relevant keys.
2. Read their current values.
3. Calculate the desired update in your application.
4. Queue commands with \`MULTI\`.
5. Run \`EXEC\`.
6. If a watched key changed, the transaction is aborted; retry after rereading.

\`\`\`mermaid
flowchart TD
    A["WATCH key"] --> B["GET current value"]
    B --> C["Calculate update"]
    C --> D["MULTI + queue writes"]
    D --> E["EXEC"]
    E --> F{"Watched key changed?"}
    F -->|"Yes"| G["Abort and retry from fresh read"]
    G --> A
    F -->|"No"| H["Commit queued commands"]
\`\`\`

### Redis CLI example

Use a disposable instance and two terminals.

**Terminal A:**

\`\`\`redis
SET demo:stock 5
WATCH demo:stock
GET demo:stock
\`\`\`

Before Terminal A commits, **Terminal B** runs:

\`\`\`redis
DECR demo:stock
\`\`\`

Now Terminal A:

\`\`\`redis
MULTI
SET demo:stock 4
EXEC
\`\`\`

Because the watched key changed, \`EXEC\` aborts instead of overwriting the newer value.

The exact CLI display for an aborted \`EXEC\` depends on the client/version; it represents a null/aborted transaction result.

## 7. WATCH details that matter

- \`WATCH\` applies to the **connection**.
- It watches keys for changes, not individual Hash fields.
- An expiration or eviction of a watched key can invalidate the transaction.
- \`EXEC\` clears the watch state.
- \`DISCARD\` clears the watch state.
- \`UNWATCH\` explicitly clears watches.
- Your retry loop should be **bounded** and use backoff under contention.

\`\`\`redis
WATCH demo:stock
UNWATCH
\`\`\`

**Production pitfall:** Do not mix unrelated concurrent \`WATCH\` transactions on the same connection. Use an isolated/dedicated client connection for a WATCH-based workflow.

## 8. Node.js example: MULTI/EXEC

Install:

\`\`\`bash
npm install redis
\`\`\`

Use ESM with \`"type": "module"\`.

\`\`\`js
import { createClient } from "redis";

const redis = createClient({
  url: process.env.REDIS_URL ?? "redis://127.0.0.1:6379",
});

redis.on("error", (err) => console.error("Redis error:", err.message));

try {
  await redis.connect();

  await redis.set("demo:points:a", "100");
  await redis.set("demo:points:b", "50");

  const results = await redis
    .multi()
    .decrBy("demo:points:a", 10)
    .incrBy("demo:points:b", 10)
    .exec();

  console.log("Transaction results:", results);
  console.log("A:", await redis.get("demo:points:a"));
  console.log("B:", await redis.get("demo:points:b"));
} catch (err) {
  console.error("Transaction failed:", err.message);
  process.exitCode = 1;
} finally {
  if (redis.isOpen) await redis.quit();
}
\`\`\`

This groups two writes. It does **not** validate that A had enough points before decrementing. You must enforce business rules separately.

## 9. Node.js example: WATCH and retry

Imagine decrementing stock only if stock is positive.

\`\`\`js
import { createClient, WatchError } from "redis";

const redis = createClient({
  url: process.env.REDIS_URL ?? "redis://127.0.0.1:6379",
});

redis.on("error", (err) => console.error("Redis error:", err.message));

async function reserveOne(client, stockKey, maxAttempts = 5) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    await client.watch(stockKey);

    try {
      const raw = await client.get(stockKey);
      if (raw === null) {
        await client.unwatch();
        throw new Error("Stock key does not exist");
      }

      const stock = Number(raw);
      if (!Number.isSafeInteger(stock) || stock < 0) {
        await client.unwatch();
        throw new Error("Invalid stock value");
      }

      if (stock === 0) {
        await client.unwatch();
        return false;
      }

      await client.multi().decr(stockKey).exec();
      return true;
    } catch (err) {
      if (err instanceof WatchError) {
        // Another client changed the key. Retry with a fresh read.
        continue;
      }

      // Clear watch state if this connection will be reused.
      await client.unwatch().catch(() => {});
      throw err;
    }
  }

  throw new Error("Too much contention; please retry later");
}

try {
  await redis.connect();
  await redis.set("demo:stock", "3");

  // Isolate WATCH state from unrelated operations.
  const isolated = redis.duplicate();
  isolated.on("error", console.error);
  await isolated.connect();

  try {
    console.log("Reserved:", await reserveOne(isolated, "demo:stock"));
  } finally {
    await isolated.quit();
  }
} catch (err) {
  console.error(err.message);
  process.exitCode = 1;
} finally {
  if (redis.isOpen) await redis.quit();
}
\`\`\`

**Version note:** This example uses the modern node-redis API. Confirm your installed major version's \`WatchError\` behavior and transaction methods.

### What about high contention?

If hundreds of clients compete for one stock key, optimistic retries may become expensive. A Lua script or Redis Function can execute the check-and-update entirely server-side.

### What about payments?

Even a correct Redis stock reservation does not make a PostgreSQL order transaction or payment processor operation atomic with Redis. Use authoritative inventory and reconciliation for business-critical flows.

## 10. WATCH vs Lua scripting

### WATCH

- Read in the application.
- Detect concurrent changes.
- Retry on conflict.
- Useful when logic is easy to express with normal client commands.

### Lua script / Redis Function

- Read and update inside Redis as one atomic server-side execution.
- Avoid the network gap between read and write.
- Must be carefully written to avoid long-running blocking operations.

\`\`\`mermaid
flowchart LR
    A["Client"] --> B{"Conditional logic?"}
    B -->|"Simple single operation"| C["INCR / SET NX / HINCRBY"]
    B -->|"Read then write with retry"| D["WATCH + MULTI/EXEC"]
    B -->|"Atomic server-side logic"| E["Lua / Redis Function"]
\`\`\`

We cover scripts and functions in Lesson 027.

## 11. Transactions vs pipelines

A pipeline sends commands efficiently to reduce network round trips.

A transaction queues commands for non-interleaved execution.

| Feature | Pipeline | Transaction |
| --- | --- | --- |
| Reduce network waiting | Yes | Often |
| Prevent interleaving during grouped execution | No | Yes |
| Roll back runtime errors | No | No |
| Conditional update based on earlier read | No | WATCH or scripting |
| Suitable for bulk independent writes | Yes | Sometimes, but not needed |
| Suitable for related updates requiring grouping | No | Yes |

**Node-redis API distinction:**

\`\`\`js
// Transaction
await redis.multi().set("a", "1").set("b", "2").exec();

// Non-transactional pipeline
await redis.multi().set("a", "1").set("b", "2").execAsPipeline();
\`\`\`

Do not confuse \`execAsPipeline()\` with \`exec()\`.

## 12. Transactions and Redis Cluster

Redis Cluster partitions keys into hash slots.

A transaction involving multiple keys must obey Cluster routing constraints; typically all transaction keys need to belong to the same slot.

Hash tags can help:

\`\`\`text
wallet:{user42}:balance
wallet:{user42}:reserved
\`\`\`

Both keys share the \`user42\` hash tag.

\`\`\`mermaid
flowchart LR
    A["wallet:{user42}:balance"] --> H["Hash slot for user42"]
    B["wallet:{user42}:reserved"] --> H
\`\`\`

**Warning:** Putting all users under one common hash tag would create a hot slot and defeat distribution. Colocate only keys that truly require multi-key operations.

## 13. Transaction durability is a separate concern

\`EXEC\` returning successfully does not mean every write is permanently safe against every hardware or node failure.

Persistence settings matter:
- RDB snapshots may lag behind current state.
- AOF fsync policy affects the durable write window.
- Replication can be asynchronous.

Redis transactions and persistence answer different questions:

- **Transaction:** How are commands executed together?
- **Persistence:** What can be recovered after a failure?
- **Replication:** What copies exist on other nodes?
- **Backup:** What older protected state can be restored?

## 14. What if the connection drops after EXEC?

Imagine:

1. Client sends \`EXEC\`.
2. Redis executes the transaction.
3. Connection drops before the reply arrives.

The client may not know whether the transaction ran.

Blindly retrying \`INCRBY\` or \`DECRBY\` can apply the operation twice.

For important workflows:
- Use idempotency keys.
- Store a unique operation ID.
- Prefer an authoritative database transaction for critical financial state.
- Reconcile outcomes when a response is uncertain.

**A Redis transaction does not solve distributed exactly-once execution.**

## 15. Common production patterns

### Atomic creation only if absent

\`\`\`redis
SET signup:email:hash pending NX EX 300
\`\`\`

This single command may be enough; no transaction required.

### Atomic counter

\`\`\`redis
INCR api:requests
\`\`\`

No transaction required.

### Related counter updates

\`\`\`redis
MULTI
INCR dashboard:applied
DECR dashboard:draft
EXEC
\`\`\`

Grouping prevents interleaving during execution, but you still need to consider invalid negative values and persistence.

### Conditional stock reservation

Use \`WATCH\` with retry or a server-side script, and keep authoritative stock consistency in the primary database.

## 16. Common mistakes

1. Assuming Redis transactions roll back on runtime errors.
2. Expecting results of queued commands before \`EXEC\`.
3. Confusing \`MULTI/EXEC\` with a pipeline.
4. Assuming \`MULTI/EXEC\` alone prevents lost updates from earlier application reads.
5. Forgetting \`WATCH\` and retry logic.
6. Sharing a WATCH connection across unrelated concurrent requests.
7. Retrying a non-idempotent transaction after an uncertain timeout.
8. Assuming transactions span Redis and PostgreSQL.
9. Ignoring Redis Cluster hash-slot restrictions.
10. Using long-running scripts to solve simple problems.
11. Treating \`EXEC\` success as proof of durable storage.
12. Forgetting business invariant checks such as stock cannot become negative.

## 17. Hands-on exercises

1. Run \`MULTI\`, queue two \`SET\` commands, and \`EXEC\`.
2. Queue a command and then \`DISCARD\`.
3. Demonstrate a queue-time syntax error.
4. Demonstrate a runtime \`WRONGTYPE\` error.
5. Confirm that a later valid command can execute after a runtime error.
6. Simulate a lost update with two clients.
7. Replace a GET/SET counter update with \`INCRBY\`.
8. Use two terminals to test \`WATCH\` abort behavior.
9. Write a bounded WATCH retry loop in Node.js.
10. Explain why WATCH needs an isolated connection.
11. Compare \`exec()\` and \`execAsPipeline()\`.
12. Design Cluster-compatible transaction keys with hash tags.
13. Explain what happens when the network drops after \`EXEC\`.
14. Choose between a single command, transaction, WATCH, and Lua for five use cases.

## 18. Interview questions

**Q1. What is MULTI?** It starts queuing commands for a Redis transaction.

**Q2. What is EXEC?** It executes the queued transaction commands without interleaving from other clients.

**Q3. What is DISCARD?** It discards queued commands before execution.

**Q4. Does Redis roll back runtime errors?** No.

**Q5. What is WATCH?** Optimistic concurrency control that aborts EXEC if watched keys changed.

**Q6. Why do WATCH transactions need retries?** Concurrent changes can invalidate the transaction.

**Q7. Can you read a queued GET result before EXEC?** No.

**Q8. Pipeline vs transaction?** Network batching vs non-interleaved execution.

**Q9. Is INCR atomic?** Yes, as a single Redis command.

**Q10. Are Redis and PostgreSQL transactions shared?** No.

**Q11. Can EXEC succeed but writes later be lost in a crash?** Yes, depending on persistence and failure conditions.

**Q12. Why are idempotency keys useful?** They help prevent duplicate business effects after uncertain retries.

## 19. Revision checklist

- [ ] MULTI / EXEC / DISCARD
- [ ] Queuing vs execution
- [ ] Transaction isolation
- [ ] Queue-time vs runtime errors
- [ ] No automatic rollback
- [ ] Lost update problem
- [ ] WATCH and optimistic locking
- [ ] UNWATCH and connection scope
- [ ] Bounded retry loops
- [ ] node-redis transactions
- [ ] Pipeline vs transaction
- [ ] Redis Cluster slots
- [ ] Durability vs atomicity
- [ ] Idempotency and uncertain outcomes
- [ ] WATCH vs Lua/Functions

---

**Official references:** [Redis transactions](https://redis.io/docs/latest/develop/using-commands/transactions/) · [MULTI](https://redis.io/docs/latest/commands/multi/) · [EXEC](https://redis.io/docs/latest/commands/exec/) · [WATCH](https://redis.io/docs/latest/commands/watch/) · [node-redis](https://github.com/redis/node-redis)
