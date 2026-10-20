# Lesson 018 — Choosing the Right Redis Data Type

> **Section 02 — Data Types and Commands** | Beginner → Intermediate  
> **Goal:** Choose Redis data structures based on real application requirements, not just command familiarity.

**Previous:** [Lesson 017 — Redis Streams Introduction](017-streams-intro.md)  
**Next:** [Lesson 019 — Redis Key Design](../03-internals/019-key-design.md)

---

## 1. Why data type selection matters

Knowing Redis commands is only the beginning. In production, the more important question is:

> **What information am I storing, and what operations must my application perform?**

Imagine a job-application tracker with these requirements:

- Cache a company profile for 10 minutes.
- Store a recruiter's name and email.
- Keep the 20 latest notifications.
- Remember which job IDs a user saved.
- Rank companies by application count.
- Estimate how many unique visitors opened a job posting.
- Record background events for workers.

Using the same Redis type for all of these is possible in some cases, but usually leads to awkward code, extra memory, race conditions, or slow queries.

**Choose by access pattern first, then check correctness, memory, and operational needs.**

## 2. Redis data types at a glance

| Data type | Stores | Main strength | Typical example |
| --- | --- | --- | --- |
| String | Bytes, text, numbers, serialized JSON | Simple value and atomic counters | API cache, rate-limit counter |
| Hash | Fields and values under one key | Partial object field access | User session metadata |
| List | Ordered sequence | Push/pop from ends | Recent activity, simple queue |
| Set | Unique unordered members | Membership and set operations | Saved job IDs, tags |
| Sorted Set | Unique members with numeric scores | Rankings and score/time ranges | Leaderboard, scheduled index |
| Bitmap | Individual bits within a String | Dense Boolean flags | Daily attendance |
| Bitfield | Packed small integers within a String | Compact numeric slots | Bounded per-slot counters |
| HyperLogLog | Approximate distinct-count sketch | Very low-memory cardinality | Unique visitors |
| Geospatial | Coordinate-indexed members in Sorted Set | Nearby-point searches | Nearby delivery partners |
| Stream | ID-ordered event entries | Replayable event log and consumer groups | Order events, worker processing |

**Important:** Bitmaps and Bitfields operate on **Strings**. Redis geospatial indexes use **Sorted Sets**. HyperLogLog is a special String-backed representation. They are command families / specialized representations, not all separate top-level types.

## 3. Start with the questions, not the commands

Before choosing a type, answer:

1. Is the value **one object/value** or a **collection**?
2. Must collection members be **unique**?
3. Does **ordering** matter? Which ordering: insertion, score, timestamp, or entry ID?
4. Do I need to **look up individual members**, or only count them?
5. Must results be **exact**, or is approximation acceptable?
6. Is data **temporary** or **authoritative**?
7. Will multiple clients update it concurrently?
8. How large can it become?
9. Do I need a **history** or only the latest state?
10. What happens if Redis restarts, evicts the key, or becomes unavailable?

\`\`\`mermaid
flowchart TD
    A["What do you need?"] --> B{"One value or object?"}
    B -->|"One value / counter"| S["String"]
    B -->|"Fields in an object"| H["Hash"]
    B -->|"Collection"| C{"Unique members?"}
    C -->|"No, order matters"| L["List"]
    C -->|"Yes"| D{"Rank or score range?"}
    D -->|"Yes"| Z["Sorted Set"]
    D -->|"No"| E{"Only approximate distinct count?"}
    E -->|"Yes"| P["HyperLogLog"]
    E -->|"No"| F{"Dense numeric flags?"}
    F -->|"Yes"| M["Bitmap"]
    F -->|"No"| SET["Set"]
    A --> G{"Need an event log and replay?"}
    G -->|"Yes"| X["Stream"]
    A --> I{"Need nearby coordinates?"}
    I -->|"Yes"| GEO["Geospatial"]
\`\`\`

This is a **starting guide**, not a complete algorithm. Some requirements combine several types.

## 4. String vs Hash — storing an object

Suppose you want to cache:

\`\`\`json
{
  "id": 42,
  "name": "Asha",
  "city": "Bengaluru"
}
\`\`\`

### Option A: String containing JSON

\`\`\`redis
SET cache:user:42 "{\"id\":42,\"name\":\"Asha\",\"city\":\"Bengaluru\"}" EX 600
GET cache:user:42
\`\`\`

**Advantages:**
- One read gets the entire serialized object.
- Natural when your API already returns JSON.
- Simple cache-aside logic.

**Disadvantages:**
- Updating one field usually means reading, parsing, changing, and writing the whole JSON.
- Concurrent read-modify-write cycles can overwrite each other's changes.
- The value is not queryable as ordinary JSON fields using core String commands.

### Option B: Hash

\`\`\`redis
HSET user:42 name Asha city Bengaluru
HGET user:42 city
HGETALL user:42
HSET user:42 city Pune
\`\`\`

**Advantages:**
- Update individual fields directly.
- Retrieve selected fields with \`HMGET\`.
- Avoid rewriting an entire serialized object for one field update.

**Disadvantages:**
- Nested objects require an encoding strategy.
- Hash fields do not have independent TTLs in older Redis versions; newer Redis versions support hash-field expiration, so check server compatibility before relying on it.
- Data modeling can become more complicated for nested documents.

### Decision

| Need | Prefer |
| --- | --- |
| Cache an entire API JSON response | String |
| Update several flat fields independently | Hash |
| Nested JSON queries | Evaluate RedisJSON / database |
| Authoritative user profile | Primary database, optionally cached in Redis |

**Lesson:** Redis is often a cache or fast-access layer, not the sole source of truth.

## 5. List vs Set — ordered history or unique membership?

### Recent notifications: List

\`\`\`redis
LPUSH notifications:user:42 "Application viewed"
LPUSH notifications:user:42 "Interview scheduled"
LRANGE notifications:user:42 0 19
LTRIM notifications:user:42 0 19
\`\`\`

A List preserves an ordered sequence and allows duplicates.

**Important:** \`LPUSH\` and \`LTRIM\` are separate commands. If you need the append-and-trim workflow to be atomic, use a transaction or Lua script.

### Saved jobs: Set

\`\`\`redis
SADD saved-jobs:user:42 job:101 job:102
SADD saved-jobs:user:42 job:101
SISMEMBER saved-jobs:user:42 job:101
SCARD saved-jobs:user:42
\`\`\`

A Set stores each member once. Repeatedly adding \`job:101\` does not create duplicates.

### Decision

- Need **order and duplicates** → List.
- Need **uniqueness and membership** → Set.
- Need **uniqueness plus rank/order by score** → Sorted Set.

## 6. Set vs Sorted Set — uniqueness alone or ranking?

A Set answers:

> Is user 42 following company 101?

\`\`\`redis
SADD company:101:followers user:42
SISMEMBER company:101:followers user:42
\`\`\`

A Sorted Set answers:

> Who has the highest score?

\`\`\`redis
ZADD interview:leaderboard 50 user:42 80 user:99 65 user:15
ZRANGE interview:leaderboard 0 2 REV WITHSCORES
ZREVRANK interview:leaderboard user:42
\`\`\`

**Key differences:**
- Both have unique members.
- Sets have no member scores.
- Sorted Sets order members by numeric score.
- A Sorted Set is usually more complex and expensive than a Set when ranking is unnecessary.

### Tie behavior

Two members can have the same score. Redis still orders them deterministically by member value. If your business requires tied members to share the same displayed rank, implement that rule separately.

## 7. Sorted Set vs Stream — timestamps are not an event log

A Sorted Set can index tasks by due time:

\`\`\`redis
ZADD scheduled:reminders 1893456000 reminder:101
ZRANGE scheduled:reminders -inf 1893456000 BYSCORE
\`\`\`

This finds due reminders, but it does not automatically:
- Deliver them to a worker.
- Track acknowledgments.
- Retry failed processing.
- Recover work claimed by a crashed worker.

A Stream stores events with IDs:

\`\`\`redis
XADD orders:events * type ORDER_CREATED orderId 101
XRANGE orders:events - +
\`\`\`

With consumer groups, Streams can track delivered-but-unacknowledged entries.

### Decision

| Requirement | Better starting point |
| --- | --- |
| Top scores / rank | Sorted Set |
| Query items due before a timestamp | Sorted Set |
| Replay events by ID | Stream |
| Cooperative workers with pending tracking | Stream + consumer group |
| Delayed jobs, retries, backoff, scheduling | Consider BullMQ |

A robust scheduler can use a Sorted Set as one component and Streams or a job queue for execution.

## 8. Bitmap vs Set — dense flags or sparse identifiers?

Suppose you have a known population of one million sequential slots and need one Boolean per slot.

A Bitmap's theoretical raw storage is:

\`\`\`text
1,000,000 bits / 8 = 125,000 bytes ≈ 122 KiB
\`\`\`

\`\`\`redis
SETBIT attendance:2026-10-21 42 1
GETBIT attendance:2026-10-21 42
BITCOUNT attendance:2026-10-21
\`\`\`

This can be compact when offsets are **dense and bounded**.

But if your user IDs are sparse and huge, this is dangerous:

\`\`\`redis
SETBIT demo:sparse 100000000 1
\`\`\`

Redis must represent the bits up to that offset, potentially allocating a large String.

Use a Set instead when storing a sparse collection of arbitrary IDs:

\`\`\`redis
SADD attendance:2026-10-21 user:100000000
SISMEMBER attendance:2026-10-21 user:100000000
\`\`\`

### Decision

- Dense integer-indexed flags → Bitmap.
- Sparse IDs / arbitrary member strings → Set.
- Small packed integer fields → Bitfield, only if encoding complexity is justified.

## 9. Set vs HyperLogLog — exact or approximate unique count?

Suppose a website receives ten million unique visitors.

### Exact distinct visitors

\`\`\`redis
SADD visitors:today user:1 user:2 user:3
SCARD visitors:today
\`\`\`

You can also check membership and list members, but memory grows with the collection.

### Approximate distinct visitors

\`\`\`redis
PFADD hll:visitors:today user:1 user:2 user:3
PFCOUNT hll:visitors:today
\`\`\`

HyperLogLog provides a compact estimate with a typical standard error of about **0.81%**, not a guaranteed error bound for every result. Its dense representation uses about **12 KiB** plus overhead.

**Crucial:** HyperLogLog cannot tell you exactly which users visited or whether one particular user is present.

### Decision

- Financially important exact count → Set or authoritative database.
- Analytics dashboard with acceptable approximation → HyperLogLog.
- Need to delete a specific member → Set.
- Need to estimate union of daily visitors → HyperLogLog with multi-key \`PFCOUNT\` or \`PFMERGE\`.

Never sum daily unique counts to calculate monthly uniques when users can visit on multiple days.

## 10. Geospatial vs Sorted Set — location or arbitrary numeric score?

Redis geospatial indexes use Sorted Sets internally, but they encode longitude and latitude into special scores.

\`\`\`redis
GEOADD geo:stores 77.5946 12.9716 store:1
GEOADD geo:stores 77.6100 12.9800 store:2
GEOSEARCH geo:stores FROMLONLAT 77.6000 12.9750 BYRADIUS 5 km ASC WITHDIST
\`\`\`

Use geospatial commands for **nearby-point lookup**.

Use regular Sorted Sets for **scores, priorities, rankings, or timestamps**.

**Do not manually overwrite geospatial scores with arbitrary \`ZADD\` values** or you may invalidate the location encoding.

**Geographic distance is not driving distance or ETA.** For routing, use an appropriate routing service.

## 11. Pub/Sub vs Stream — live signals or replayable events?

Redis Pub/Sub is a **messaging mechanism**, not a retained collection data type.

### Live typing indicator

\`\`\`redis
PUBLISH chat:room:101 "user:42 typing"
\`\`\`

If nobody is subscribed, the message is not saved for later delivery.

### Important order event

\`\`\`redis
XADD orders:events * type ORDER_CREATED orderId 101
\`\`\`

A Stream retains the event until it is deleted or trimmed. Consumers can read it later, subject to retention and durability.

### Decision

- Ephemeral live update → Pub/Sub.
- Need replay/history → Stream.
- Need worker acknowledgment and recovery → Stream with consumer group and explicit recovery logic.
- Need complete background-job features → Consider a dedicated queue library.

## 12. Choosing types for a real project: CareerLoop

Imagine a job application tracker built with Next.js, PostgreSQL, and Redis.

| Feature | Redis type | Why | Source of truth |
| --- | --- | --- | --- |
| Cached dashboard summary | String | Cache serialized JSON with TTL | PostgreSQL |
| Recruiter contact fields | Hash | Fast access to individual fields | PostgreSQL |
| Recent UI notifications | List | Latest-first bounded history | PostgreSQL if durable |
| Saved job IDs | Set | Unique membership | PostgreSQL |
| Most-applied companies | Sorted Set | Ranking by application count | PostgreSQL |
| Daily active-user flags | Bitmap | Compact dense flags if IDs mapped | Analytics records |
| Approximate unique visitors | HyperLogLog | Small-memory distinct estimate | Depends on reporting needs |
| Nearby offices | Geospatial | Search by coordinates | Database |
| Application-status events | Stream | Ordered event history for workers | PostgreSQL + outbox |

\`\`\`mermaid
flowchart TD
    U["Next.js / API"] --> DB[("PostgreSQL: authoritative data")]
    U --> R[("Redis")]
    R --> C["Strings: dashboard cache"]
    R --> S["Sets: saved jobs"]
    R --> Z["Sorted Sets: rankings"]
    R --> E["Streams: async events"]
    DB --> O["Outbox publisher"]
    O --> E
\`\`\`

**Why not store everything in Redis?** Redis may be used as a primary database in some designs, but a job tracker often benefits from relational constraints, transactions, joins, and durable records in PostgreSQL. Redis then improves latency and supports specialized operations.

## 13. Another real-world project: CodeBuddy

Consider a developer networking and chat platform.

| Requirement | Starting choice | Explanation |
| --- | --- | --- |
| Cached profile | String | Serialized JSON |
| Online presence | String with TTL | Heartbeat-based presence; handle expiry carefully |
| Users who liked a post | Set | Unique user IDs |
| Mutual connections | Set operations | Intersection between connection sets |
| Weekly activity leaderboard | Sorted Set | Scores and ranking |
| Typing indicator | Pub/Sub | Transient real-time signal |
| Durable chat messages | Database | Persistent message history |
| Message-processing events | Stream | Worker consumption and replay |
| Estimated unique visitors | HyperLogLog | Approximate analytics |

**Important:** A chat message should not exist only in Pub/Sub if users expect to see it after reconnecting. Store durable messages in a database and use real-time messaging for delivery notifications.

## 14. Data type choice is also about operations

The same business entity can be modeled in several ways.

For example, a shopping cart:

### String with JSON

\`\`\`text
cart:user:42 -> '{"items":[{"productId":"p1","quantity":2}]}'
\`\`\`

Easy to read as one object, but concurrent updates can overwrite each other unless coordinated.

### Hash of quantities

\`\`\`redis
HSET cart:user:42 product:p1 2 product:p2 1
HINCRBY cart:user:42 product:p1 1
\`\`\`

Easy to increment one item's quantity atomically.

### Database

If you need complex pricing rules, inventory reservations, transactional consistency, and durable checkout records, use a database as the authoritative source.

**Lesson:** The correct model depends on the operations, not merely the name “cart.”

## 15. Atomicity: one command vs many commands

Redis executes individual commands atomically on a server, but multi-command application workflows are not automatically atomic.

### Bad read-modify-write counter

\`\`\`text
GET counter
add 1 in Node.js
SET counter newValue
\`\`\`

Two clients may read the same old value and overwrite each other's increment.

### Better single-command increment

\`\`\`redis
INCR page:views
\`\`\`

### Another example: add then trim

\`\`\`redis
LPUSH notifications:user:42 "New interview"
LTRIM notifications:user:42 0 19
\`\`\`

These are two separate commands. Use \`MULTI/EXEC\` or a Lua script if the pair must be atomic.

### Cross-system consistency

Even if Redis operations are atomic, **PostgreSQL + Redis** is not one shared transaction. Design cache invalidation, outbox publication, retries, and recovery explicitly.

## 16. TTL: expiration belongs in the model

Ask: **How long should this data exist?**

\`\`\`redis
SET cache:company:101 "{\"name\":\"Example Co\"}" EX 600
SET session:token:abc active EX 3600
EXPIRE leaderboard:weekly 604800
TTL cache:company:101
\`\`\`

Important details:

- TTL is normally applied to the **key**.
- A key expiring removes its entire value, not one Set member or one Stream entry.
- Redis versions differ in support for hash-field expiration; verify compatibility before designing around it.
- Streams generally need explicit retention/trimming, not just key TTL.
- Updating a String with ordinary \`SET\` can clear its existing TTL unless options such as \`KEEPTTL\` are used.

**Never assume all Redis members have independent expiration.**

## 17. Memory and performance trade-offs

| Pattern | Advantage | Potential cost |
| --- | --- | --- |
| One JSON String | Simple and compact for whole-object reads | Full rewrite for small changes |
| Hash | Partial field updates | More modeling for nested objects |
| List | Fast end operations | Searching/removing arbitrary items can be expensive |
| Set | Fast membership | No ordering; memory per member |
| Sorted Set | Ranking and range queries | More memory/CPU than plain Set |
| Bitmap | Dense Boolean flags | Huge sparse offsets waste memory |
| HyperLogLog | Tiny approximate distinct count | No exact membership |
| Geospatial | Fast nearby-point candidates | Not a full GIS/routing engine |
| Stream | Replay and consumer-group tracking | Retention, lag, pending-state operations |

### Avoid huge responses

Examples that can become expensive:

\`\`\`redis
SMEMBERS huge:set
LRANGE huge:list 0 -1
ZRANGE huge:leaderboard 0 -1 WITHSCORES
XRANGE huge:stream - +
\`\`\`

For large collections, use pagination, bounded ranges, or SCAN-family commands where appropriate.

**SCAN is not a consistent snapshot**, and its output is not automatically sorted.

## 18. A production decision checklist

Before introducing a Redis key, document:

1. **Purpose:** What question does it answer?
2. **Key naming:** For example, \`cache:company:101\`.
3. **Type:** String, Hash, Set, etc.
4. **Access pattern:** Commands and frequency.
5. **Size bound:** Can it grow forever?
6. **TTL/retention:** When does data expire?
7. **Source of truth:** Can it be rebuilt?
8. **Consistency:** What happens on concurrent updates?
9. **Failure handling:** What if Redis is unavailable?
10. **Security:** Does it contain tokens, personal data, or sensitive locations?
11. **Monitoring:** How will you detect latency, memory pressure, or lag?
12. **Cluster compatibility:** Will multi-key operations require the same hash slot?

\`\`\`mermaid
flowchart TD
    A["Identify access pattern"] --> B["Choose candidate Redis type"]
    B --> C["Check exactness and atomicity"]
    C --> D["Check memory, TTL and growth"]
    D --> E["Check durability and recovery"]
    E --> F["Implement and benchmark"]
    F --> G["Monitor and revisit"]
\`\`\`

## 19. Common mistakes

1. Choosing String for every problem without considering field-level updates.
2. Using List when uniqueness is required.
3. Using Set when sorted rankings are required.
4. Using Sorted Set only because it sounds more powerful than Set.
5. Treating HyperLogLog as an exact unique counter.
6. Using Bitmap offsets directly from huge sparse IDs.
7. Assuming geospatial distance equals driving distance.
8. Using Pub/Sub for events that must survive disconnections.
9. Assuming a Stream is automatically a reliable job system.
10. Forgetting key TTL and unbounded growth.
11. Using multiple Redis commands without considering races.
12. Treating Redis and PostgreSQL writes as one transaction.
13. Ignoring eviction and persistence when data matters.
14. Fetching entire large collections instead of bounding reads.

## 20. Practice: choose the type yourself

For each requirement, choose a type and explain **why**.

| Requirement | Suggested answer |
| --- | --- |
| Cache a product JSON response for 5 minutes | String |
| Store and update flat profile fields | Hash |
| Keep last 20 notifications | List |
| Remember unique tags for a post | Set |
| Rank users by points | Sorted Set |
| Track presence on 31 days | Bitmap |
| Pack 4-bit counters for dense slots | Bitfield |
| Estimate daily unique visitors | HyperLogLog |
| Find couriers within 3 km | Geospatial |
| Store order events for workers | Stream |
| Broadcast transient typing state | Pub/Sub (messaging) |
| Store authoritative payments | Database, not a Redis-only shortcut |

### Challenge 1 — Rate limiter

You need a limit of 100 requests per minute.

Possible choices:
- **String + INCR + TTL:** simple fixed-window counter.
- **Sorted Set:** store timestamped request events for a sliding window.
- **Lua script/function:** make multi-command checks and updates atomic.

No single Redis type automatically provides every rate-limiting guarantee.

### Challenge 2 — Notification system

You need:
- Latest notifications displayed in UI.
- Durable history.
- Real-time delivery.
- Email sending with retries.

A reasonable architecture combines:
- PostgreSQL for durable notifications.
- List or String for recent cached UI data.
- Pub/Sub or Socket.IO adapter for live updates.
- Stream/BullMQ for background email processing.

**A good system can use several data types together.**

### Challenge 3 — E-commerce inventory

You need exact stock levels and must prevent overselling.

Redis atomic counters can be useful for a fast reservation layer, but the complete design must consider:
- Authoritative inventory.
- Reservation expiry.
- Checkout transaction consistency.
- Idempotency.
- Reconciliation after failures.

Do not assume \`DECR\` alone solves every inventory correctness problem.

## 21. Interview questions

**Q1. String vs Hash?** String is convenient for one serialized value; Hash supports individual field updates.

**Q2. List vs Set?** List preserves sequence and duplicates; Set stores unique members.

**Q3. Set vs Sorted Set?** Sorted Set adds numeric scores and ranking.

**Q4. Why use HyperLogLog?** Approximate unique counting with very low memory.

**Q5. Why can Bitmaps waste memory?** Large sparse offsets force storage up to the highest offset.

**Q6. Does Redis GEO calculate ETA?** No, it provides geographic proximity/distance, not road routing.

**Q7. Pub/Sub vs Stream?** Pub/Sub is ephemeral delivery; Streams retain entries for replay.

**Q8. Can Streams guarantee exactly-once business processing?** Not automatically; design idempotency and recovery.

**Q9. Is Redis always the source of truth?** No. Often a database owns authoritative data while Redis accelerates reads and specialized operations.

**Q10. Why does TTL matter to type selection?** Data lifetime and member-level expiration requirements influence the structure and key design.

**Q11. Are two Redis commands automatically atomic together?** No. Use transactions or scripting when needed.

**Q12. What is the first question before choosing a type?** What operations and correctness guarantees does the application need?

## 22. Revision checklist

- [ ] String vs Hash
- [ ] List vs Set
- [ ] Set vs Sorted Set
- [ ] Sorted Set vs Stream
- [ ] Bitmap vs Set
- [ ] Set vs HyperLogLog
- [ ] Geospatial vs Sorted Set
- [ ] Pub/Sub vs Stream
- [ ] Exact vs approximate results
- [ ] TTL and retention
- [ ] Atomicity and concurrency
- [ ] Durability and source of truth
- [ ] Memory growth and bounded queries
- [ ] Multi-type application architecture

---

## Section 02 completed

You have now studied the major Redis data structures and the decisions behind using them:

- [Lesson 009 — Strings](009-strings.md)
- [Lesson 010 — Hashes](010-hashes.md)
- [Lesson 011 — Lists](011-lists.md)
- [Lesson 012 — Sets](012-sets.md)
- [Lesson 013 — Sorted Sets](013-sorted-sets.md)
- [Lesson 014 — Bitmaps and Bitfields](014-bitmaps-bitfields.md)
- [Lesson 015 — HyperLogLog](015-hyperloglog.md)
- [Lesson 016 — Geospatial](016-geospatial.md)
- [Lesson 017 — Streams Introduction](017-streams-intro.md)
- [Lesson 018 — Data Type Selection](018-data-type-selection.md)

**Next section:** Redis Keys, Memory and Internals. We will study key naming, TTL/eviction, memory encodings, persistence, pipelining, transactions, and Lua scripting.

**Official references:** [Redis Data Types](https://redis.io/docs/latest/develop/data-types/) · [Redis Commands](https://redis.io/docs/latest/commands/) · [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/) · [Redis Sorted Sets](https://redis.io/docs/latest/develop/data-types/sorted-sets/)
