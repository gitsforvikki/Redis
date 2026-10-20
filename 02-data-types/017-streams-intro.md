# Lesson 017 — Redis Streams Introduction

> **Section 02: Data Types and Commands**  
> **Level:** Beginner to Intermediate  
> **Goal:** Learn event streams from first principles, use the core commands, and understand how Streams differ from Lists, Pub/Sub, and durable job systems.

**Previous:** [Lesson 016 — Redis Geospatial](016-geospatial.md)  
**Next:** [Lesson 018 — Choosing the Right Redis Data Type](018-data-type-selection.md)

---

## 1. Why Redis Streams exist

Imagine a shopping application. When an order is placed, multiple background operations might be needed:

1. Reserve inventory.
2. Send an order-confirmation email.
3. Update analytics.
4. Notify the delivery service.

Doing everything inside the HTTP request can make the API slow and fragile.

Instead, the API can **publish an event**, and other services can process it later.

A simple Redis List can act as a queue, but an item removed with \`LPOP\` is no longer available if the worker crashes before finishing. Redis Pub/Sub sends events to currently connected subscribers but does not retain a replayable history.

Redis **Streams** provide a different model: an **append-only, ID-ordered log of entries** that can be read later.

\`\`\`mermaid
flowchart LR
    API["Order API"] -->|"XADD"| S[("Redis Stream: orders:events")]
    S --> I["Inventory worker"]
    S --> E["Email worker"]
    S --> A["Analytics worker"]
\`\`\`

**Important:** A Stream does not automatically guarantee that events are never lost or processed exactly once. Persistence, replication, trimming, acknowledgments, retries, and idempotency all matter.

## 2. What is a Stream?

A Stream is a Redis data type containing entries. Every entry has:

- A **unique entry ID** within the Stream.
- One or more **field-value pairs**.

Example:

\`\`\`text
Key: orders:events

1760000000000-0
  type: ORDER_CREATED
  orderId: 101
  userId: 42

1760000000100-0
  type: ORDER_PAID
  orderId: 101
  userId: 42
\`\`\`

Stream IDs normally look like:

\`\`\`text
<milliseconds>-<sequence>
\`\`\`

When Redis generates IDs, the first component is generally based on its clock, and the sequence distinguishes entries. IDs are ordered, but do not treat them as an authoritative business-event timestamp.

### A Stream is not a Hash

A Redis Hash stores fields for **one logical object**. A Stream stores **multiple ordered entries**, each with its own fields.

### A Stream is not a Set

Sets deduplicate members. Streams do **not** deduplicate identical event payloads. Adding the same event twice creates two entries.

## 3. XADD — append an event

Open \`redis-cli\` against a disposable local Redis instance:

\`\`\`redis
DEL demo:orders
XADD demo:orders * type ORDER_CREATED orderId 101 userId 42
XADD demo:orders * type ORDER_PAID orderId 101 userId 42
XADD demo:orders * type ORDER_CREATED orderId 102 userId 99
\`\`\`

The \`*\` asks Redis to generate the entry ID.

Each \`XADD\` returns its new ID. Your IDs will differ from the examples.

**Production point:** An HTTP request might time out after Redis accepted an \`XADD\`. Retrying with \`*\` may create a duplicate business event. Consumers should use a stable business event ID and idempotency rules.

## 4. XLEN — count retained entries

\`\`\`redis
XLEN demo:orders
\`\`\`

Expected: \`3\`.

This is the number of **currently retained** entries. If entries are deleted or trimmed, \`XLEN\` decreases. It is not a lifetime total.

## 5. XRANGE — read event history

Read the entire retained history:

\`\`\`redis
XRANGE demo:orders - +
\`\`\`

Here:
- \`-\` means the smallest possible ID.
- \`+\` means the largest possible ID.

Limit the result:

\`\`\`redis
XRANGE demo:orders - + COUNT 2
\`\`\`

Read in reverse order:

\`\`\`redis
XREVRANGE demo:orders + - COUNT 2
\`\`\`

**Why limit results?** Returning millions of entries in one response can consume significant network bandwidth and application memory.

### Read after a particular ID

Suppose an entry's actual ID is \`1760000000000-0\`:

\`\`\`redis
XRANGE demo:orders (1760000000000-0 + COUNT 10
\`\`\`

The opening \`(\` means the lower bound is **exclusive**. Replace the sample ID with an ID returned by your own Redis instance.

## 6. XREAD — consume entries

\`XREAD\` returns entries with IDs **greater than** the ID you provide.

\`\`\`redis
XREAD COUNT 10 STREAMS demo:orders 0
\`\`\`

This starts after ID \`0\`, so it reads the demo history.

After processing the results, store the **last successfully processed ID** and use it next time:

\`\`\`redis
XREAD COUNT 10 STREAMS demo:orders 1760000000100-0
\`\`\`

Replace that ID with your own last processed entry.

\`\`\`mermaid
flowchart TD
    A["Start at saved cursor"] --> B["XREAD after cursor"]
    B --> C["Process entries"]
    C --> D["Save last processed ID"]
    D --> B
\`\`\`

**Critical distinction:** Plain \`XREAD\` does not maintain server-side acknowledgment or pending-message state for your application. You are responsible for the cursor and failure handling.

### What if processing crashes?

If you save the cursor **before** completing the work, you might skip an event permanently.

If you save the cursor **after** completing the work, a crash can cause the same event to be processed again.

This is why real systems need **idempotent processing** and often use **consumer groups**.

## 7. Blocking reads — wait for new events

Open **Terminal A**:

\`\`\`redis
XREAD BLOCK 0 STREAMS demo:live $
\`\`\`

Open **Terminal B**:

\`\`\`redis
XADD demo:live * message hello
\`\`\`

Terminal A should receive the new entry.

- \`BLOCK 0\`: wait indefinitely for data.
- \`$\`: start at the current end and wait for **future entries**.

### The dangerous $ mistake

Do **not** repeatedly use \`$\` for every iteration of a long-running consumer. Messages arriving between iterations may be skipped.

Correct approach for a new-only consumer:

1. Start with \`$\` if you intentionally want to ignore existing history.
2. When a message arrives, save its actual ID.
3. Use that ID for subsequent reads.

If you need historical entries, start at \`0\` or a saved checkpoint.

**Connection note:** Blocking commands occupy their client connection while waiting. Use a dedicated Redis client connection for a long-running blocking consumer, rather than sharing it with latency-sensitive API operations.

## 8. Read from multiple Streams

\`\`\`redis
XREAD COUNT 10 STREAMS demo:orders demo:live 0 0
\`\`\`

The first group of arguments lists Stream keys; the second group supplies one starting ID for each key in the same order.

This does not create a single global ordering across different Streams.

## 9. XDEL — delete an entry

\`\`\`redis
XDEL demo:orders 1760000000000-0
\`\`\`

Use a real ID from \`XRANGE\` instead of the sample ID.

\`XDEL\` removes the entry logically. Internal Stream storage may not immediately reclaim all associated memory.

**Warning:** Deleting an event can prevent a slower consumer from processing it. Use deletion only when it matches your retention and correctness requirements.

## 10. XTRIM — limit Stream size

A Stream grows as events are appended. Without a retention policy, memory usage can grow indefinitely.

Approximate length-based trimming:

\`\`\`redis
XTRIM demo:orders MAXLEN ~ 1000
\`\`\`

Exact length-based trimming:

\`\`\`redis
XTRIM demo:orders MAXLEN = 1000
\`\`\`

Append and trim in one command:

\`\`\`redis
XADD demo:orders MAXLEN ~ 1000 * type ORDER_CREATED orderId 103
\`\`\`

Approximate trimming is often more efficient because Redis can remove internal chunks.

### Retention is a business decision

\`\`\`mermaid
flowchart LR
    P["Producer appends"] --> S["Stream grows"]
    S --> T["Retention / XTRIM"]
    T --> R["Old events removed"]
    R --> C["Slow consumer may miss events"]
\`\`\`

If a consumer falls behind and its needed events are trimmed, those events may no longer be recoverable from the Stream.

Consider:
- Expected consumer downtime.
- Maximum processing delay.
- Replay requirements.
- Memory limits.
- Whether an authoritative event record exists elsewhere.

## 11. Consumer groups — the basic idea

A **consumer group** lets several workers cooperate to process Stream entries.

\`\`\`mermaid
flowchart TD
    S["Stream: jobs:email"] --> G["Group: email-workers"]
    G --> W1["Worker 1"]
    G --> W2["Worker 2"]
    G --> W3["Worker 3"]
\`\`\`

Redis tracks deliveries to the group, including messages delivered but not acknowledged.

### Important terms

| Term | Meaning |
| --- | --- |
| Group | Named set of cooperating consumers |
| Consumer | Worker identity within the group |
| Pending Entries List (PEL) | Tracks entries delivered but not acknowledged |
| Acknowledgment | Confirmation that processing has completed |
| Claiming | Recovery mechanism for pending entries assigned to stalled consumers |

### Create a group

\`\`\`redis
XGROUP CREATE demo:orders order-workers 0 MKSTREAM
\`\`\`

- \`0\`: begin from retained history.
- \`$\`: begin from the current end, for future entries.
- \`MKSTREAM\`: create the Stream if missing.

### Read as a group member

\`\`\`redis
XREADGROUP GROUP order-workers worker-1 COUNT 2 STREAMS demo:orders >
\`\`\`

The \`>\` asks for entries not previously delivered to the group.

### Acknowledge after successful processing

\`\`\`redis
XACK demo:orders order-workers 1760000000000-0
\`\`\`

Use the real returned ID.

**Do not acknowledge before the business operation succeeds.** Otherwise, a crash can make unfinished work appear complete.

### What happens after a crash?

An unacknowledged entry remains pending in group state, but Redis does **not** automatically retry it for another worker. Your application must inspect and recover abandoned entries, often with commands such as \`XPENDING\` and \`XAUTOCLAIM\`.

We will study this deeply in:
- [Lesson 048 — Streams and Consumer Groups](../06-messaging/048-streams-consumers.md)
- [Lesson 049 — Acknowledgments, Retries and Recovery](../06-messaging/049-ack-retry.md)

**Exactly-once warning:** Redis Streams with consumer groups are typically used to build **at-least-once** processing. Duplicate delivery and duplicate business side effects remain possible; design consumers to be idempotent.

## 12. Streams vs Lists vs Pub/Sub

| Requirement | List | Pub/Sub | Stream |
| --- | --- | --- | --- |
| Retained messages | Yes, until removed | No | Yes, until removed/trimmed |
| Replay history | Depends on whether items remain | No | Yes |
| Multiple independent readers | Application-managed | Active subscribers | Yes |
| Consumer-group tracking | No | No | Yes |
| Acknowledgment bookkeeping | Not built in | Not built in | Yes, with groups |
| Automatic retries and backoff | Not built in | Not built in | Requires application logic |
| Typical fit | Simple queue/stack | Ephemeral live signals | Event log / worker processing |

### Real-world decisions

- “User is typing” notification → Pub/Sub can be appropriate because missing a transient event is often acceptable.
- Simple disposable FIFO tasks → List may suffice.
- Order-created events requiring history and worker coordination → Streams with carefully designed consumers.
- Delayed emails, retries, backoff, dashboards → A queue library such as BullMQ may be easier than implementing every feature yourself.

## 13. Node.js example — publish and read events

Install:

\`\`\`bash
npm install redis
\`\`\`

Use ESM (\`"type": "module"\` in \`package.json\`).

\`\`\`js
import { createClient } from "redis";

const redis = createClient({
  url: process.env.REDIS_URL ?? "redis://127.0.0.1:6379",
});

redis.on("error", (error) => {
  console.error("Redis error:", error.message);
});

const streamKey = "demo:orders:lesson17";

async function publishOrderCreated(orderId, userId) {
  return redis.xAdd(streamKey, "*", {
    type: "ORDER_CREATED",
    orderId: String(orderId),
    userId: String(userId),
    createdAt: new Date().toISOString(),
  });
}

async function getRecentEvents(limit = 10) {
  return redis.xRevRange(streamKey, "+", "-", { COUNT: limit });
}

try {
  await redis.connect();

  const firstId = await publishOrderCreated(101, 42);
  const secondId = await publishOrderCreated(102, 99);

  console.log("Created IDs:", firstId, secondId);
  console.log("Recent events:", await getRecentEvents());
} catch (error) {
  console.error("Stream operation failed:", error.message);
  process.exitCode = 1;
} finally {
  if (redis.isOpen) await redis.quit();
}
\`\`\`

**Expected behavior:** Each run appends two new entries and displays recent entries in descending ID order.

### Production improvements

- Validate and authenticate event producers.
- Use stable business event IDs for deduplication.
- Avoid sensitive data in unprotected event payloads.
- Monitor memory, stream length, and consumer lag.
- Use TLS/authentication when connecting to managed Redis.
- Define retention and disaster recovery.
- Handle failures between database commits and event publication.

## 14. Database + Stream consistency: transactional outbox

Suppose an order API performs:

\`\`\`text
1. INSERT order in PostgreSQL
2. XADD order event in Redis
\`\`\`

If the process crashes between these steps, the order exists but the event is missing.

A **transactional outbox** pattern helps:

\`\`\`mermaid
flowchart TD
    A["Order API"] --> B["PostgreSQL transaction"]
    B --> C["Insert order and outbox event"]
    C --> D["Commit transaction"]
    D --> E["Publisher reads outbox"]
    E --> F["XADD Redis Stream"]
    F --> G["Idempotent consumers"]
\`\`\`

The database transaction atomically records the order and a row describing the event to publish. A publisher later retries sending that event.

**Important:** Retries can publish the same event more than once, so consumers still need idempotency. The outbox improves reliability but does not provide magical exactly-once delivery across PostgreSQL and Redis.

## 15. Performance and production trade-offs

| Operation | General cost | Risk |
| --- | --- | --- |
| \`XADD\` | Efficient amortized append | Unbounded growth without retention |
| \`XLEN\` | Constant-time length lookup | Does not show historical total |
| \`XRANGE\` | Depends on number returned | Huge responses |
| \`XREAD\` | Depends on returned entries | Cursor mistakes and missed work |
| \`XTRIM\` | Depends on trimming | Slow consumers lose history |
| \`XACK\` | Per-entry bookkeeping | Premature ack loses processing opportunity |

Also monitor pending entries, consumer idle time, retries, replication lag, and persistence settings.

## 16. Common mistakes

1. Assuming Streams and Pub/Sub behave the same way.
2. Assuming \`XREAD\` automatically saves a cursor.
3. Using \`$\` on every read and silently skipping events.
4. Expecting \`XADD\` to deduplicate repeated payloads.
5. Acknowledging before processing finishes.
6. Believing a pending message will automatically be retried.
7. Assuming consumer groups guarantee exactly-once side effects.
8. Trimming history before slow consumers can process it.
9. Treating Stream IDs as authoritative event timestamps.
10. Sharing a blocking connection with unrelated API traffic.
11. Assuming PostgreSQL and Redis writes are one transaction.
12. Treating a Redis Stream as an eternal audit log without a durability plan.

## 17. Hands-on exercises

1. Add three entries with \`XADD\`.
2. Read all entries with \`XRANGE\`.
3. Read the latest two entries with \`XREVRANGE\`.
4. Check \`XLEN\`.
5. Use \`XREAD\` starting at \`0\`.
6. Read only entries after a known ID.
7. Use two terminals to test \`XREAD BLOCK\`.
8. Explain why repeatedly using \`$\` is dangerous.
9. Trim a demo Stream to approximately five entries.
10. Create a consumer group.
11. Read and acknowledge a message.
12. Explain how a worker crash affects the PEL.
13. Run the Node.js example.
14. Draw the transactional-outbox architecture.

## 18. Interview questions

**Q1. What is a Redis Stream?** An append-only, ID-ordered collection of entries with field-value data.

**Q2. What does \`XADD\` return?** The ID of the new entry.

**Q3. What does \`*\` mean in \`XADD\`?** Redis generates the entry ID.

**Q4. \`XRANGE\` vs \`XREVRANGE\`?** Ascending vs descending ID order.

**Q5. What does \`XREAD ... $\` mean?** Start at the current end and read future entries.

**Q6. Why not reuse \`$\` on every loop?** It can skip entries added between calls.

**Q7. What is a consumer group?** Cooperating consumers with delivery and pending-entry tracking.

**Q8. What is \`XACK\`?** An acknowledgment of an entry delivered to a consumer group.

**Q9. What happens when a worker crashes before \`XACK\`?** Its delivered entry can remain pending and requires recovery logic.

**Q10. Are Streams exactly once?** Not automatically; idempotent processing is required.

**Q11. Why is aggressive trimming dangerous?** Slow consumers may lose unprocessed entries.

**Q12. Why use a transactional outbox?** To reliably record an event to publish alongside a database transaction.

## 19. Revision checklist

- [ ] Stream structure and IDs
- [ ] \`XADD\`, \`XLEN\`
- [ ] \`XRANGE\`, \`XREVRANGE\`
- [ ] \`XREAD\` and cursors
- [ ] Blocking reads and \`$\`
- [ ] \`XDEL\`, \`XTRIM\`
- [ ] Consumer groups and PEL
- [ ] \`XREADGROUP\`, \`XACK\`
- [ ] Idempotency and crash recovery
- [ ] Streams vs Lists vs Pub/Sub
- [ ] Node.js integration
- [ ] Transactional outbox

---

**Official references:** [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/) · [XADD](https://redis.io/docs/latest/commands/xadd/) · [XREAD](https://redis.io/docs/latest/commands/xread/) · [XREADGROUP](https://redis.io/docs/latest/commands/xreadgroup/) · [XACK](https://redis.io/docs/latest/commands/xack/)
