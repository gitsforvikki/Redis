# Lesson 014 — Redis Bitmaps and Bitfields

> **Section 02 — Data Types and Commands** | Beginner → Advanced  
> **Goal:** Understand how Redis stores compact binary flags and packed integers, how bit operations work, and when they are worth the complexity.

**Prerequisite:** [Lesson 013 — Redis Sorted Sets](013-sorted-sets.md)

## 1. What is a Bitmap in Redis?

A **Bitmap** is a way to treat the bits inside a Redis **String** as individual on/off flags.

**Important:** Redis Bitmap is **not a separate top-level Redis data type**. It uses the String data type and bit-oriented commands.

Each bit is either:
- \`0\`: off / false / not recorded
- \`1\`: on / true / recorded

\`\`\`mermaid
flowchart LR
    A["Redis String"] --> B["Bit 0"]
    A --> C["Bit 1"]
    A --> D["Bit 2"]
    A --> E["Bit 3"]
    B --> F["1"]
    C --> G["0"]
    D --> H["1"]
    E --> I["1"]
\`\`\`

### Why use Bitmaps?

They can represent large collections of **dense, numbered Boolean flags** compactly.

Example: For each day of a month, record whether a user logged in.

\`\`\`text
Day:       1 2 3 4 5 6 7
Logged in: 1 0 1 1 0 0 1
\`\`\`

Seven flags require seven bits in the logical representation. Actual Redis memory usage also includes key/value metadata and allocation overhead.

## 2. Bits, bytes, offsets: understand the foundation

- **1 bit** = one 0/1 value.
- **8 bits** = one byte.
- Redis bit offsets start at **0**.
- Bitmap commands operate on the bits of a String.

Redis uses **most-significant-bit-first** ordering within each byte for the bit offsets exposed by \`SETBIT\` and \`GETBIT\`.

For a byte:

\`\`\`text
Offset:  0 1 2 3 4 5 6 7
Bits:    1 0 1 0 0 0 0 0
Binary:  10100000
Hex:     A0
\`\`\`

This matters when you inspect raw bytes or combine Redis bit commands with application binary encodings.

## 3. SETBIT — set a flag

\`\`\`redis
DEL demo:attendance
SETBIT demo:attendance 0 1
SETBIT demo:attendance 2 1
SETBIT demo:attendance 3 1
\`\`\`

The logical bits begin:

\`\`\`text
Offset: 0 1 2 3 4 5 6 7
Value:  1 0 1 1 0 0 0 0
\`\`\`

\`SETBIT key offset value\` accepts only \`0\` or \`1\` as the bit value.

**Return value:** \`SETBIT\` returns the **previous bit value**, not the new bit value.

Try:

\`\`\`redis
SETBIT demo:attendance 0 1
\`\`\`

Because offset 0 is already 1, the command returns 1.

### Why this is useful

If you record daily attendance, you can tell whether a bit was already set by inspecting the previous value. But a single bit cannot store multiple attendance timestamps or distinguish multiple visits.

## 4. GETBIT — read a flag

\`\`\`redis
GETBIT demo:attendance 0
GETBIT demo:attendance 1
GETBIT demo:attendance 2
\`\`\`

Expected results: 1, 0, 1.

Reading a bit beyond the current String length returns 0.

**Subtle issue:** Zero can mean either an explicitly unset bit or an offset beyond the stored data. If your business logic needs to distinguish “never initialized” from “false,” you need additional metadata or a different representation.

## 5. BITCOUNT — count the 1 bits

\`\`\`redis
BITCOUNT demo:attendance
\`\`\`

For our demo, expected count: \`3\`.

### Real-world interpretation

If one bit represents attendance on one day, \`BITCOUNT\` counts the days marked present.

It does **not** count total login events. A user logging in 20 times on the same day still corresponds to one set bit.

### Complexity

\`BITCOUNT\` must examine data across the selected range. Counting over a large Bitmap is not always constant-time.

## 6. BITPOS — find a bit

\`\`\`redis
BITPOS demo:attendance 1
BITPOS demo:attendance 0
\`\`\`

- Find the first set bit.
- Find the first unset bit.

\`BITPOS\` supports ranges; exact range-unit options vary by Redis version. Use the official command documentation when working with \`BYTE\` or \`BIT\` units.

**Important:** Finding zero bits in padded or out-of-range areas can have subtle behavior. Always test your intended boundaries.

## 7. BITOP — combine Bitmaps

Redis supports bitwise operations across String values:

- \`AND\`
- \`OR\`
- \`XOR\`
- \`NOT\`

### Example: attendance overlap

\`\`\`redis
DEL attendance:alice attendance:bob attendance:both
SETBIT attendance:alice 0 1
SETBIT attendance:alice 1 1
SETBIT attendance:alice 3 1

SETBIT attendance:bob 1 1
SETBIT attendance:bob 2 1
SETBIT attendance:bob 3 1

BITOP AND attendance:both attendance:alice attendance:bob
BITCOUNT attendance:both
\`\`\`

Expected count: \`2\`, because both attended on offsets 1 and 3.

\`\`\`mermaid
flowchart TD
    A["Alice: 1 1 0 1"] --> C["AND: 0 1 0 1"]
    B["Bob:   0 1 1 1"] --> C
    C --> D["2 shared days"]
\`\`\`

### OR — attended by at least one

\`\`\`redis
BITOP OR attendance:either attendance:alice attendance:bob
BITCOUNT attendance:either
\`\`\`

Expected count: \`4\`.

### XOR — attended by exactly one

\`\`\`redis
BITOP XOR attendance:one-only attendance:alice attendance:bob
BITCOUNT attendance:one-only
\`\`\`

Expected count: \`2\`.

### NOT — invert bits

\`\`\`redis
BITOP NOT attendance:inverted attendance:alice
\`\`\`

**Important:** \`NOT\` flips all bits in the source String's allocated length, not only the days you consider meaningful. If you only track four days, you must mask or restrict your interpretation accordingly.

## 8. Practical Bitmap modeling: user attendance

A key design:

\`\`\`text
attendance:user:42:2026-10
\`\`\`

Map **day 1 → offset 0**, **day 2 → offset 1**, and so on.

\`\`\`redis
SETBIT attendance:user:42:2026-10 0 1
SETBIT attendance:user:42:2026-10 4 1
SETBIT attendance:user:42:2026-10 10 1
BITCOUNT attendance:user:42:2026-10
\`\`\`

Expected: \`3\` recorded attendance days.

### Why use separate monthly keys?

- The offset stays small (0–30).
- Keys can be retained or expired by month.
- A key corresponds to a clear reporting period.

**But note:** If attendance must be authoritative, Redis alone may not meet your audit, recovery, and integrity requirements. Use a primary database and treat the Bitmap as an index/cache when appropriate.

## 9. Memory math: when Bitmaps help

Suppose you have **1,000,000 users** and one Boolean flag per user.

Theoretical raw bit storage:

\`\`\`text
1,000,000 bits / 8 = 125,000 bytes
≈ 122 KiB
\`\`\`

Compare this with storing one separate Redis key per user, which would incur key and value metadata overhead for every user.

**But there is a trap:** Bitmap offsets must be **dense or deliberately mapped**.

If user IDs are huge and sparse:

\`\`\`redis
SETBIT demo:sparse 100000000 1
\`\`\`

Redis may need to allocate a String spanning that far offset, even though you set only one bit. This can waste memory and cause latency.

### Better approaches for sparse IDs

- Map user IDs to compact integer indexes, if you can manage the mapping.
- Use a Redis Set when the number of members is sparse.
- Use a database or other specialized representation.

**Do not use unbounded external IDs directly as Bitmap offsets.**

## 10. What is a Bitfield?

A **Bitfield** lets you treat ranges of bits inside a Redis String as **small signed or unsigned integers**.

Instead of storing only one flag per bit, you can pack numbers such as:

\`\`\`text
User A: 3-bit counter
User B: 3-bit counter
User C: 3-bit counter
\`\`\`

\`\`\`mermaid
flowchart LR
    A["Redis String bytes"] --> B["Bits 0–2: counter A"]
    A --> C["Bits 3–5: counter B"]
    A --> D["Bits 6–8: counter C"]
\`\`\`

This is useful when:
- Values have known small ranges.
- You have many dense numeric entries.
- Memory efficiency is important.
- Your team can maintain the encoding safely.

It is **not** a replacement for Hashes or ordinary counters when simplicity matters more than packed storage.

## 11. BITFIELD syntax

A \`BITFIELD\` command can perform one or more operations:

\`\`\`text
BITFIELD key
  GET <encoding> <offset>
  SET <encoding> <offset> <value>
  INCRBY <encoding> <offset> <increment>
\`\`\`

Common encodings:
- \`u8\`: unsigned 8-bit integer (0–255).
- \`i8\`: signed 8-bit integer (-128 to 127).
- \`u4\`: unsigned 4-bit integer (0–15).
- \`i16\`: signed 16-bit integer.

The encoding specifies both signedness and width. Redis Bitfield commands support specific widths and offsets; check command documentation for detailed limits.

## 12. BITFIELD GET and SET

\`\`\`redis
DEL demo:bitfield
BITFIELD demo:bitfield SET u8 0 200
BITFIELD demo:bitfield GET u8 0
\`\`\`

Expected \`GET\`: \`200\`.

The \`SET\` operation returns the **previous value** at that bit range, typically \`0\` for a new zero-filled String.

### Pack two 4-bit counters into one byte

\`\`\`redis
DEL demo:packed
BITFIELD demo:packed SET u4 0 9 SET u4 4 6
BITFIELD demo:packed GET u4 0 GET u4 4
\`\`\`

Expected values: \`9\` and \`6\`.

Logical layout:

\`\`\`text
9 = 1001
6 = 0110

Packed byte: 10010110
             ^^^^ ^^^^
             u4@0 u4@4
\`\`\`

**Important:** Overlapping bit ranges affect each other. Design and document offsets carefully.

## 13. BITFIELD INCRBY and overflow modes

\`\`\`redis
DEL demo:counter4
BITFIELD demo:counter4 SET u4 0 14
BITFIELD demo:counter4 INCRBY u4 0 1
\`\`\`

Result: \`15\`.

What happens if we add 1 again? Four unsigned bits cannot represent 16.

Redis supports overflow modes:

### WRAP (default)

\`\`\`redis
BITFIELD demo:counter4 OVERFLOW WRAP INCRBY u4 0 1
\`\`\`

15 + 1 wraps to 0.

### SAT (saturate)

\`\`\`redis
BITFIELD demo:counter4 SET u4 0 15
BITFIELD demo:counter4 OVERFLOW SAT INCRBY u4 0 1
\`\`\`

Stays at 15.

### FAIL

\`\`\`redis
BITFIELD demo:counter4 OVERFLOW FAIL INCRBY u4 0 1
\`\`\`

The overflowing operation is not applied, and its result is nil.

### Choosing an overflow policy

| Policy | On overflow | Example use |
| --- | --- | --- |
| WRAP | Roll over | Cyclic counter |
| SAT | Clamp to min/max | Bounded metric |
| FAIL | Reject that operation | Detect invalid increment |

**Do not assume WRAP is safe for a business counter.** Silent rollover can corrupt application meaning.

## 14. BITFIELD_RO — read-only packed values

Redis provides \`BITFIELD_RO\` for read-only bitfield operations:

\`\`\`redis
BITFIELD_RO demo:packed GET u4 0 GET u4 4
\`\`\`

Useful when you want to express a read-only operation clearly. It is available in modern Redis versions; check compatibility.

## 15. Node.js: Bitmap attendance example

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

redis.on("error", (error) => console.error("Redis error:", error.message));

const attendanceKey = (userId, yearMonth) =>
  \`attendance:user:\${userId}:\${yearMonth}\`;

async function markAttendance(userId, yearMonth, dayOfMonth) {
  if (!Number.isInteger(dayOfMonth) || dayOfMonth < 1 || dayOfMonth > 31) {
    throw new Error("Invalid day");
  }

  const key = attendanceKey(userId, yearMonth);
  return redis.setBit(key, dayOfMonth - 1, 1);
}

async function hasAttendance(userId, yearMonth, dayOfMonth) {
  if (!Number.isInteger(dayOfMonth) || dayOfMonth < 1 || dayOfMonth > 31) {
    throw new Error("Invalid day");
  }

  return (await redis.getBit(
    attendanceKey(userId, yearMonth),
    dayOfMonth - 1,
  )) === 1;
}

async function attendedDays(userId, yearMonth) {
  return redis.bitCount(attendanceKey(userId, yearMonth));
}

try {
  await redis.connect();

  await markAttendance(42, "2026-10", 1);
  await markAttendance(42, "2026-10", 3);
  await markAttendance(42, "2026-10", 5);

  console.log("Day 3?", await hasAttendance(42, "2026-10", 3));
  console.log("Days present:", await attendedDays(42, "2026-10"));
} catch (error) {
  console.error("Attendance failed:", error.message);
  process.exitCode = 1;
} finally {
  if (redis.isOpen) await redis.quit();
}
\`\`\`

Expected:
- Day 3: \`true\`
- Days present: \`3\`

### Production improvements

- Validate that \`yearMonth\` is a valid calendar month and the day exists in that month.
- Authenticate the user; do not accept arbitrary user IDs as proof of identity.
- Decide the authoritative data source and retention policy.
- Consider time zones: “today” differs across regions.
- Handle Redis downtime and recovery.
- Consider key TTL and whether historical attendance must remain forever.

## 16. Atomicity and concurrency

A single \`SETBIT\` or \`BITFIELD INCRBY\` command executes atomically on the Redis server.

But this does **not** make a multi-command workflow atomic.

For example:

\`\`\`text
GETBIT attendance 5
if zero:
  SETBIT attendance 5 1
  INCR monthly-total
\`\`\`

Two clients can both read zero and both increment the total, even though only one new attendance bit is needed.

### Safer patterns

- Use \`SETBIT\`'s **previous bit value** when a single operation is sufficient.
- If you must atomically update the bit and another counter, consider a Redis Lua script/function.
- If authoritative data is in PostgreSQL, plan cross-system consistency explicitly.

We will study transactions and scripting in Lessons 026–027.

## 17. Bitmap vs Set vs HyperLogLog

| Requirement | Bitmap | Set | HyperLogLog |
| --- | --- | --- | --- |
| Exact membership | Yes, with dense numeric offsets | Yes, by arbitrary member ID | No |
| Exact distinct count | \`BITCOUNT\` for flags | \`SCARD\` | No, approximate |
| Stores arbitrary user IDs directly | Poor for sparse huge IDs | Yes | Accepts inputs but does not expose members |
| Memory for dense Boolean universe | Often excellent | Higher per member | Small fixed-ish sketch |
| Individual member TTL | No | No | No |
| Recover exact members | Yes, if mapping known | Yes | No |

### Example decision

- Track daily login flags for a user over a month → Bitmap.
- Track exact users who liked a post → Set.
- Estimate unique website visitors with low memory → HyperLogLog.
- Track 4-bit counters for dense slots → Bitfield.

## 18. Performance and complexity

| Command | Typical cost | Practical note |
| --- | --- | --- |
| \`GETBIT\` | O(1) | Reads one bit |
| \`SETBIT\` | O(1) for existing allocated area; growth can allocate | Huge sparse offsets can be expensive |
| \`BITCOUNT\` | O(N) over examined bytes | Large ranges cost CPU |
| \`BITPOS\` | O(N) over examined bytes | Searches until a match |
| \`BITOP\` | O(N) over input length | Creates/overwrites destination |
| \`BITFIELD\` | Depends on operations and allocation | Packed access can be efficient |

**Important:** Sparse high offsets can cause sudden String growth and latency. Test realistic ID distributions and memory usage.

## 19. Common mistakes

1. **Calling Bitmap a separate Redis data type.** It is a bit-oriented use of Strings.
2. **Using large sparse user IDs as bit offsets.** Memory can grow unexpectedly.
3. **Assuming SETBIT returns the new bit.** It returns the old bit.
4. **Treating a bit as an event count.** It records a Boolean state, not repeated occurrences.
5. **Assuming GETBIT 0 means explicitly false.** It can also mean the offset was never stored.
6. **Using BITCOUNT as O(1).** It scans data.
7. **Forgetting NOT flips every allocated bit.** Limit interpretation to meaningful positions.
8. **Overlapping Bitfield offsets accidentally.** Fields can corrupt each other.
9. **Ignoring Bitfield overflow behavior.** Default WRAP can silently roll over.
10. **Assuming bit commands solve multi-command races.** Only individual commands are atomic.
11. **Ignoring time zones and calendar validity in attendance tracking.**
12. **Using Redis-only flags for mandatory audit records without durability planning.**

## 20. Hands-on exercises

Use a disposable local Redis instance.

1. Set bits 0, 2, and 4 and verify with \`GETBIT\`.
2. Call \`SETBIT\` twice on the same offset and explain the return values.
3. Count set bits using \`BITCOUNT\`.
4. Create two attendance Bitmaps and compute their AND and OR.
5. Compute XOR and explain what it means.
6. Use \`BITPOS\` to find the first set bit.
7. Pack two unsigned 4-bit numbers into one byte using \`BITFIELD\`.
8. Increment a 4-bit counter past 15 using WRAP.
9. Repeat with SAT and FAIL; compare results.
10. Build monthly attendance in Node.js.
11. Explain why sparse user IDs are dangerous as offsets.
12. Compare Bitmap vs Set for 100,000 randomly distributed user IDs.

## 21. Interview questions

**Q1. Is Bitmap a separate Redis data type?** No. It uses the String type.

**Q2. What does SETBIT return?** The previous bit value.

**Q3. What does BITCOUNT calculate?** Number of set bits in the selected data.

**Q4. How can you find shared attendance days?** Apply BITOP AND to two compatible Bitmaps, then BITCOUNT.

**Q5. Why are sparse high offsets problematic?** Redis may allocate a large String to reach the offset.

**Q6. What is a Bitfield?** A way to read/write packed signed or unsigned integers within a String.

**Q7. What does u4 mean?** An unsigned four-bit integer, range 0–15.

**Q8. What happens on Bitfield overflow?** Depends on WRAP, SAT, or FAIL mode.

**Q9. Is BITCOUNT constant-time?** No; it examines bytes across the selected range.

**Q10. Does a Bitmap replace an event log?** No. A bit records state, not every occurrence or audit detail.

**Q11. Are two SETBIT operations together atomic?** Each is atomic individually, but the pair is not automatically one transaction.

**Q12. Bitmap vs Set?** Bitmap fits dense numeric flags; Set fits unique arbitrary members.

## 22. Revision checklist

- [ ] Bitmap is backed by Redis String
- [ ] Bits, bytes and offsets
- [ ] \`SETBIT\`, \`GETBIT\`, \`BITCOUNT\`, \`BITPOS\`
- [ ] \`BITOP AND/OR/XOR/NOT\`
- [ ] Dense vs sparse offsets
- [ ] Monthly attendance design
- [ ] \`BITFIELD GET/SET/INCRBY\`
- [ ] Signed and unsigned encodings
- [ ] Overflow WRAP/SAT/FAIL
- [ ] \`BITFIELD_RO\`
- [ ] Atomicity and race conditions
- [ ] Bitmap vs Set vs HyperLogLog
- [ ] Node.js integration and production trade-offs

---

**Previous:** [Lesson 013 — Redis Sorted Sets](013-sorted-sets.md)  
**Next:** [Lesson 015 — Redis HyperLogLog](015-hyperloglog.md) (planned)

**Official references:** [Redis Bitmaps](https://redis.io/docs/latest/develop/data-types/bitmaps/) · [Redis Bitfields](https://redis.io/docs/latest/develop/data-types/bitfields/) · [SETBIT](https://redis.io/docs/latest/commands/setbit/) · [BITFIELD](https://redis.io/docs/latest/commands/bitfield/) · [BITOP](https://redis.io/docs/latest/commands/bitop/)
