# Lesson 015 — Redis HyperLogLog

> **Section 02 — Data Types and Commands** | Beginner → Advanced  
> **Goal:** Understand approximate unique counting, HyperLogLog commands, error and memory trade-offs, and production analytics patterns.

**Prerequisite:** [Lesson 014 — Redis Bitmaps and Bitfields](014-bitmaps-bitfields.md)

## 1. The problem: count unique visitors

Suppose your website receives millions of page views. You want to answer:

> How many **different users** visited today?

Counting every request is easy, but it is not the same as counting **unique** visitors:

\`\`\`text
Visits: A, B, A, C, B, A
Total visits = 6
Unique visitors = 3
\`\`\`

A Redis **Set** can store every distinct ID and provide an **exact** count using \`SCARD\`. However, memory grows as the number of unique IDs grows.

**HyperLogLog (HLL)** is a probabilistic data structure that estimates the number of distinct elements while using very little memory.

\`\`\`mermaid
flowchart LR
  E["Events: A, B, A, C, B, A"] --> H["Redis HyperLogLog"]
  H --> N["Estimated unique count ≈ 3"]
\`\`\`

**Critical:** HyperLogLog is **approximate**. Do not use it when exact counts are legally, financially, or operationally required.

## 2. What is Redis HyperLogLog?

HyperLogLog estimates **cardinality**: the number of distinct values in a collection.

It is useful when:
- You only need a **count**, not the actual members.
- The number of distinct items may be very large.
- A small estimation error is acceptable.
- Memory efficiency is more important than perfect accuracy.

Redis HyperLogLog is implemented using a special encoding within a **String key**, rather than a general-purpose Set of members.

### A useful comparison

| Requirement | Redis Set | Redis HyperLogLog |
| --- | --- | --- |
| Count unique elements | Exact | Approximate |
| List members | Yes, \`SMEMBERS\` | No |
| Check whether a particular ID exists | Yes, \`SISMEMBER\` | No |
| Delete a particular member | Yes, \`SREM\` | No |
| Memory growth with cardinality | Grows with members | Small bounded sketch |
| Good for millions of unique visitors | When exact membership matters | When approximate count is enough |

Redis HyperLogLog's standard dense representation is approximately **12 KiB**, though small sparse representations may use less, and actual key memory includes overhead. The typical relative standard error is about **0.81%**.

**Important:** 0.81% is a statistical error characteristic, **not** a guarantee that every result lies within 0.81% of the true value.

## 3. How does it work? A simple mental model

A Set remembers **which** values you added.

HyperLogLog instead remembers a compact **statistical summary** of hashed values.

Conceptually:
1. Hash each input value.
2. Observe patterns in the binary hash, such as long runs of zero bits.
3. Maintain compact registers summarizing those patterns.
4. Estimate the number of distinct inputs from the register values.

\`\`\`mermaid
flowchart TD
  A["user:42"] --> H["Hash input"]
  B["user:99"] --> H
  C["user:42 again"] --> H
  H --> R["Compact HLL registers"]
  R --> E["Estimate distinct count"]
\`\`\`

The actual algorithm includes corrections and estimation details. You do **not** need to implement the mathematics to use Redis correctly, but you must understand the accuracy trade-off.

### Why duplicates generally do not change the estimate

Adding the same value produces the same hash, so the sketch's state normally does not gain new distinct-information from duplicates.

## 4. PFADD — add observations

\`\`\`redis
DEL analytics:unique:demo
PFADD analytics:unique:demo user:1 user:2 user:3
PFADD analytics:unique:demo user:1 user:2
\`\`\`

The sketch has seen three distinct IDs.

### Important return-value trap

\`PFADD\` returns:
- \`1\` if the sketch's internal registers/representation changed.
- \`0\` if the sketch did not change.

It **does not** return “number of newly unique users.” Adding a previously unseen value can return \`0\` if it does not change the sketch.

## 5. PFCOUNT — estimate cardinality

\`\`\`redis
PFCOUNT analytics:unique:demo
\`\`\`

For this tiny example, you will typically see \`3\`.

But it is an **estimate**, not an exact-count promise.

### Example: daily unique visitors

\`\`\`redis
PFADD visitors:2026-10-21 user:42
PFADD visitors:2026-10-21 user:99
PFADD visitors:2026-10-21 user:42
PFCOUNT visitors:2026-10-21
\`\`\`

**Use a stable identifier** if you intend to count unique users. If the identifier changes per request, your metric becomes something else.

### Multiple keys

\`\`\`redis
PFCOUNT visitors:2026-10-20 visitors:2026-10-21
\`\`\`

With multiple keys, Redis estimates the cardinality of their **union** — unique visitors across both days.

**Not** the sum of each day's counts! Someone visiting on both days should count only once in the combined estimate.

## 6. PFMERGE — combine HyperLogLogs

\`\`\`redis
PFADD visitors:mon user:1 user:2
PFADD visitors:tue user:2 user:3
PFMERGE visitors:two-days visitors:mon visitors:tue
PFCOUNT visitors:two-days
\`\`\`

Expected estimate near \`3\`.

\`\`\`mermaid
flowchart LR
  M["Monday HLL"] --> U["PFMERGE: two-day HLL"]
  T["Tuesday HLL"] --> U
  U --> C["PFCOUNT union"]
\`\`\`

### Why merge instead of adding counts?

Monday = 2 unique visitors. Tuesday = 2 unique visitors.

Adding gives 4, but \`user:2\` visited both days, so actual combined unique visitors = 3.

HyperLogLog merging helps estimate the **distinct union** without storing the individual IDs.

### Redis Cluster caveat

Multi-key \`PFCOUNT\` and \`PFMERGE\` have cross-slot restrictions in Redis Cluster. Plan key placement or aggregate in another system if your deployment spans slots.

## 7. Key design for real analytics

A simple pattern:

\`\`\`text
hll:visitors:daily:2026-10-21
hll:visitors:daily:2026-10-22
hll:visitors:monthly:2026-10
\`\`\`

**Daily unique visitors:**

\`\`\`redis
PFADD hll:visitors:daily:2026-10-21 user:42
PFCOUNT hll:visitors:daily:2026-10-21
\`\`\`

**Monthly unique visitors:** Merge daily sketches or add users directly to a monthly sketch. Do not simply sum daily counts.

### TTL and retention

\`\`\`redis
EXPIRE hll:visitors:daily:2026-10-21 2592000
\`\`\`

This sets a 30-day expiration.

**Important:** \`PFADD\` updates the sketch but does not normally remove the existing key TTL. Plan retention deliberately, especially when monthly summaries depend on daily keys.

## 8. Practical Node.js example

Install:

\`\`\`bash
npm install redis
\`\`\`

Use ESM (\`"type": "module"\`).

\`\`\`js
import { createClient } from "redis";

const redis = createClient({
  url: process.env.REDIS_URL ?? "redis://127.0.0.1:6379",
});

redis.on("error", (err) => console.error("Redis error:", err.message));

const dailyKey = (day) => \`hll:visitors:daily:\${day}\`;

async function recordVisit(userId, day) {
  // In production, derive userId from authenticated server-side identity.
  return redis.pfAdd(dailyKey(day), String(userId));
}

async function getDailyUniqueVisitors(day) {
  return redis.pfCount(dailyKey(day));
}

try {
  await redis.connect();

  await recordVisit(42, "2026-10-21");
  await recordVisit(99, "2026-10-21");
  await recordVisit(42, "2026-10-21");

  console.log(
    "Estimated unique visitors:",
    await getDailyUniqueVisitors("2026-10-21"),
  );
} catch (error) {
  console.error("Analytics error:", error.message);
  process.exitCode = 1;
} finally {
  if (redis.isOpen) await redis.quit();
}
\`\`\`

### Production considerations

- Decide whether the metric is **unique users**, **unique devices**, or **unique sessions**.
- Avoid counting bots as users if your metric excludes them.
- Do not use raw emails or other sensitive personal data as identifiers without privacy review.
- Use consistent date/time-zone boundaries.
- Be prepared for Redis eviction, data loss, or delayed events.
- Explain that displayed counts are approximate.
- If reporting must be exact, calculate it from a suitable authoritative dataset.

## 9. What HyperLogLog cannot do

### Cannot retrieve members

\`\`\`redis
SMEMBERS hll:visitors:daily:2026-10-21
\`\`\`

This is not valid as a way to list HLL visitors. HyperLogLog is not a Redis Set.

### Cannot check membership

There is no \`PFISMEMBER\` command. You cannot reliably answer “Did user 42 visit?” from the HLL sketch.

### Cannot remove one observation

There is no direct operation to remove an individual visitor while preserving an exact corrected sketch. If your data must support deletion or corrections, plan to rebuild from source events or use a different data structure.

### Cannot guarantee exact small or large counts

Small counts can look exact in examples, but the abstraction is still approximate.

## 10. Performance and memory

| Command | Typical complexity | Note |
| --- | --- | --- |
| \`PFADD\` | O(1) per element | Small bounded sketch update |
| \`PFCOUNT\` one key | O(1) with a small bounded number of registers | May involve cached count or recomputation |
| \`PFCOUNT\` multiple keys | O(N) in number of keys plus sketch merge work | Cross-slot constraints in Cluster |
| \`PFMERGE\` | O(N) in number of source sketches | Merge compact sketches |

Redis documentation expresses some complexities with implementation-dependent constants. Even if sketch memory is small, **a very large number of daily or per-segment keys** still consumes metadata and operational resources.

## 11. HLL vs exact analytics: a decision example

Suppose your application dashboard shows:

- **“Visitors today: approximately 1.2 million”** → HLL may be appropriate.
- **“Users eligible for a ₹500 payout: exactly 15,210”** → Use an exact authoritative dataset.
- **“Show all visitors who came today”** → HLL cannot provide the list.
- **“How many unique visitors came over the last seven days?”** → Merge or count union of seven HLL keys.

\`\`\`mermaid
flowchart TD
  A["Need distinct count?"] -->|No| X["Choose another structure"]
  A -->|Yes| B{"Must count be exact?"}
  B -->|Yes| S["Set or authoritative database"]
  B -->|No| C{"Need member lookup/list?"}
  C -->|Yes| S
  C -->|No| H["HyperLogLog"]
\`\`\`

## 12. Common mistakes

1. Treating HyperLogLog as an exact distinct counter.
2. Assuming \`PFADD\` returns number of new unique values.
3. Adding daily \`PFCOUNT\` results to calculate monthly uniques.
4. Expecting to list or delete individual members.
5. Counting request IDs when you intended unique user IDs.
6. Ignoring time-zone boundaries.
7. Forgetting privacy implications of identifiers.
8. Using approximate counts for financial or compliance decisions.
9. Assuming the relative error bound is guaranteed for every estimate.
10. Ignoring Cluster multi-key restrictions.
11. Assuming the sketch uses zero memory because it is compact.
12. Using HLL as the only record when data recovery matters.

## 13. Hands-on exercises

1. Add duplicate visitor IDs and inspect \`PFCOUNT\`.
2. Compare \`PFADD\` return values with the distinct count.
3. Create two daily sketches with one overlapping visitor.
4. Use multi-key \`PFCOUNT\` for their union.
5. Merge them with \`PFMERGE\`.
6. Compare \`SCARD\` on a Set with \`PFCOUNT\` on HLL for 1,000 IDs.
7. Explain why \`PFCOUNT(day1) + PFCOUNT(day2)\` can overcount.
8. Add a TTL and inspect it with \`TTL\`.
9. Build a Node.js endpoint for approximate daily visitors.
10. Design retention and recovery for a 90-day analytics dashboard.

## 14. Interview questions

**Q1. What does HyperLogLog estimate?** The number of distinct elements (cardinality).

**Q2. Is it exact?** No, it is probabilistic.

**Q3. What is the typical standard error?** Approximately 0.81%.

**Q4. What is its approximate dense memory size?** About 12 KiB, plus overhead.

**Q5. PFADD return value?** Whether the sketch representation changed, not the number of newly distinct members.

**Q6. PFCOUNT with multiple keys?** Estimates cardinality of the union.

**Q7. PFMERGE?** Combines HLL sketches into a destination sketch.

**Q8. Can you list visitors from HLL?** No.

**Q9. Can you remove one visitor?** Not directly.

**Q10. HLL vs Set?** HLL provides approximate counts with small memory; Set stores exact identifiable members.

## 15. Revision checklist

- [ ] Distinct count vs total events
- [ ] Approximate cardinality and error
- [ ] \`PFADD\`, \`PFCOUNT\`, \`PFMERGE\`
- [ ] HLL vs Set
- [ ] Daily and monthly unique visitors
- [ ] Union vs sum
- [ ] Small memory, statistical limitations
- [ ] No member listing or deletion
- [ ] Node.js implementation
- [ ] Privacy, TTL, Cluster, recovery

---

**Previous:** [Lesson 014 — Redis Bitmaps and Bitfields](014-bitmaps-bitfields.md)  
**Next:** [Lesson 016 — Redis Geospatial](016-geospatial.md)

**Official references:** [Redis HyperLogLog](https://redis.io/docs/latest/develop/data-types/hyperloglogs/) · [PFADD](https://redis.io/docs/latest/commands/pfadd/) · [PFCOUNT](https://redis.io/docs/latest/commands/pfcount/) · [PFMERGE](https://redis.io/docs/latest/commands/pfmerge/)
