# Lesson 021 — Redis Memory Encodings

> **Section 03 — Keys, Memory and Internals** | Intermediate  
> **Goal:** Understand how Redis represents values in memory, why encoding changes, and how to inspect and optimize memory without relying on implementation details.

**Previous:** [Lesson 020 — TTL and Eviction](020-ttl-eviction.md)  
**Next:** [Lesson 022 — RDB Persistence](022-rdb.md)

## 1. Data type and internal encoding are not the same

When you execute:

\`\`\`redis
SET counter 42
HSET user:42 name Asha city Bengaluru
SADD tags:42 react nextjs redis
\`\`\`

Redis exposes **logical types** such as String, Hash, and Set. Internally, Redis can store these values using different **encodings** depending on their size, shape, and server version.

\`\`\`mermaid
flowchart TD
    A["Redis key"] --> B["Logical data type"]
    B --> C["Internal encoding"]
    C --> D["Memory representation"]
    B --> S["String"]
    B --> H["Hash"]
    B --> T["Set"]
    B --> Z["Sorted Set"]
\`\`\`

Why does this matter? Two values with the same logical type may consume very different amounts of memory. Redis chooses representations that trade memory efficiency against operation speed.

**Important:** Encodings are implementation details, not application-level contracts. They can change across Redis releases. Always check your actual Redis version.

## 2. Useful commands for exploring memory

On a disposable Redis instance:

\`\`\`redis
SET demo:string 42
OBJECT ENCODING demo:string
TYPE demo:string
MEMORY USAGE demo:string
\`\`\`

- \`TYPE\` reports the logical Redis type.
- \`OBJECT ENCODING\` reports the internal encoding.
- \`MEMORY USAGE\` estimates bytes used by the key and value, including relevant overhead.
- \`INFO memory\` reports server-wide memory metrics.

\`\`\`redis
INFO memory
MEMORY STATS
MEMORY DOCTOR
\`\`\`

**Caution:** \`MEMORY USAGE\` is a diagnostic measurement, not a precise billing metric or a guarantee of future allocation. Collection sampling can affect estimates; \`MEMORY USAGE key SAMPLES 0\` inspects all nested elements but may cost more.

## 3. Redis Strings: integer, embedded, and raw representations

Redis Strings are binary-safe values. They can represent text, JSON, numeric counters, or arbitrary bytes.

For Strings, you may encounter encodings such as:

| Encoding | General meaning |
| --- | --- |
| \`int\` | Integer stored in an optimized representation |
| \`embstr\` | Small String with compact combined allocation |
| \`raw\` | General String allocation for larger or modified values |

Try:

\`\`\`redis
SET demo:int 100
OBJECT ENCODING demo:int

SET demo:short hello
OBJECT ENCODING demo:short

SET demo:long "A relatively long text value to inspect in this Redis version"
OBJECT ENCODING demo:long
\`\`\`

The exact results and length thresholds are **version-dependent**. Do not hardcode an \`embstr\` cutoff in application logic.

### Why integer encoding can be useful

If Redis recognizes a suitable integer value, it can avoid storing the value as a general-purpose byte String.

But not every numeric-looking value is necessarily encoded as \`int\`; formatting and operations matter.

### Encoding can change

A String may start in a compact encoding and later be represented differently after modifications.

\`\`\`redis
SET demo:mutable hello
OBJECT ENCODING demo:mutable
APPEND demo:mutable world
OBJECT ENCODING demo:mutable
\`\`\`

Do not assume that every command preserves the original encoding.

## 4. Redis Hashes: compact listpack vs hashtable

Hashes are useful for storing fields of an object.

\`\`\`redis
HSET demo:user name Asha city Bengaluru role developer
OBJECT ENCODING demo:user
\`\`\`

Modern Redis versions commonly use a compact **listpack** representation for small Hashes, and a **hashtable** representation when the Hash grows beyond configured thresholds.

\`\`\`mermaid
flowchart LR
    A["Small Hash"] --> B["Compact listpack"]
    B -->|"Too many fields or large values"| C["Hashtable"]
\`\`\`

Why switch?

- Compact encodings reduce per-field memory overhead for small collections.
- Hashtables support efficient direct field lookup as collections grow.

### Inspect relevant configuration

\`\`\`redis
CONFIG GET hash-max-listpack-entries
CONFIG GET hash-max-listpack-value
\`\`\`

These control thresholds on Redis versions that support these options. Older versions used different names and encodings.

**Do not assume every Hash uses a hashtable.**

### Thresholds are not business limits

If a Hash crosses a compact-encoding threshold, Redis can convert it to a larger representation. The data remains logically a Hash, but memory usage and operation costs may change.

Do not build correctness logic around a specific encoding.

## 5. Redis Lists: quicklist and listpack

Redis Lists support operations at both ends:

\`\`\`redis
LPUSH demo:notifications "Applied" "Interview" "Offer"
OBJECT ENCODING demo:notifications
\`\`\`

Modern Redis Lists commonly use a **quicklist** representation: a linked structure whose nodes contain packed groups of entries (typically listpacks).

\`\`\`mermaid
flowchart LR
    A["Quicklist node"] --> B["Packed values"]
    B --> C["Quicklist node"]
    C --> D["Packed values"]
\`\`\`

This combines:
- Efficient push/pop near the ends.
- Reduced overhead compared with a separate linked-list node for every item.
- Configurable packing/compression trade-offs.

Configuration may include:

\`\`\`redis
CONFIG GET list-max-listpack-size
CONFIG GET list-compress-depth
\`\`\`

Avoid changing these without benchmarks and understanding the impact on your workload.

## 6. Redis Sets: integer sets and hash-based sets

A Set contains unique members.

\`\`\`redis
SADD demo:numbers 1 2 3
OBJECT ENCODING demo:numbers
\`\`\`

A small Set of integer-compatible members may use an **intset** representation on versions/configurations that support it.

\`\`\`redis
SADD demo:numbers hello
OBJECT ENCODING demo:numbers
\`\`\`

Adding a non-integer member can trigger conversion to a more general representation.

Modern Redis releases may also support compact listpack encodings for small Sets. The exact available encodings and thresholds depend on version.

### Why not always use intset?

It only fits certain values and sizes. General-purpose membership and large Sets require other representations.

**Key lesson:** A logical Set can be represented in multiple ways while \`SADD\`, \`SISMEMBER\`, and \`SCARD\` keep the same external behavior.

## 7. Sorted Sets: compact and scalable representations

Sorted Sets combine unique members with numeric scores.

\`\`\`redis
ZADD demo:ranking 100 user:42 85 user:99
OBJECT ENCODING demo:ranking
\`\`\`

For small Sorted Sets, Redis may use compact listpack encoding. Larger ones commonly use a **skiplist plus dictionary** arrangement.

\`\`\`mermaid
flowchart TD
    A["Sorted Set"] --> B{"Small and compact?"}
    B -->|"Yes"| C["Listpack"]
    B -->|"No"| D["Skiplist + dictionary"]
    D --> E["Efficient score ranges"]
    D --> F["Efficient member lookup"]
\`\`\`

The skiplist supports ordered traversal and range operations. The dictionary helps locate members.

Inspect thresholds:

\`\`\`redis
CONFIG GET zset-max-listpack-entries
CONFIG GET zset-max-listpack-value
\`\`\`

**Do not confuse the skiplist internal structure with a Redis Stream or List.** They solve different problems.

## 8. Bitmaps, HyperLogLog, GEO, and Streams

### Bitmaps

Bitmaps operate on Strings. A bitmap's memory footprint depends strongly on its **highest offset**.

\`\`\`redis
SETBIT demo:flags 7 1
MEMORY USAGE demo:flags
\`\`\`

A very large sparse offset can allocate substantial memory even when only one bit is set.

### HyperLogLog

HyperLogLog is a specialized approximate distinct-count structure backed by a String representation. Small sketches can be sparse; dense sketches use roughly 12 KiB plus overhead.

\`\`\`redis
PFADD demo:hll user:1 user:2
MEMORY USAGE demo:hll
\`\`\`

### Geospatial

Redis GEO commands store location members using a Sorted Set with encoded geographic scores.

\`\`\`redis
GEOADD demo:geo 77.5946 12.9716 bengaluru
TYPE demo:geo
OBJECT ENCODING demo:geo
\`\`\`

### Streams

Streams use specialized structures to store ordered entries efficiently. Modern implementations use a radix tree with packed entry blocks, plus additional consumer-group state when applicable.

\`\`\`redis
XADD demo:events * type ORDER_CREATED orderId 101
TYPE demo:events
OBJECT ENCODING demo:events
\`\`\`

**Stream memory includes more than event payloads:** entry metadata, IDs, consumer groups, and pending entries also cost memory.

## 9. Why encoding changes can cause memory jumps

Imagine a Hash with many tiny fields. Its compact representation can be very efficient.

After adding a large field value, Redis may convert it to a hashtable:

\`\`\`mermaid
flowchart TD
    A["Small Hash in listpack"] --> B["Add large field"]
    B --> C["Encoding conversion"]
    C --> D["More memory overhead"]
    D --> E["Potentially faster field access"]
\`\`\`

The key did not change its logical type, yet its memory use can increase sharply.

### Practical experiment

\`\`\`redis
DEL demo:hash
HSET demo:hash a 1 b 2
OBJECT ENCODING demo:hash
MEMORY USAGE demo:hash

HSET demo:hash large "A value that may or may not cross this version's configured threshold"
OBJECT ENCODING demo:hash
MEMORY USAGE demo:hash
\`\`\`

To guarantee a conversion experiment, first inspect the server's thresholds and then generate values larger than the configured limit in a local test.

## 10. Measuring memory correctly

### Per-key memory

\`\`\`redis
MEMORY USAGE demo:hash
MEMORY USAGE demo:hash SAMPLES 0
\`\`\`

### Server-wide memory

\`\`\`redis
INFO memory
\`\`\`

Useful fields often include:
- \`used_memory\`: memory allocated by Redis's allocator.
- \`used_memory_peak\`: peak observed usage.
- \`used_memory_rss\`: resident memory reported by the OS.
- \`mem_fragmentation_ratio\`: an indicator that requires context.
- \`maxmemory\`: configured eviction threshold.

**Warning:** A high fragmentation ratio does not automatically mean every byte is wasted. Small datasets, allocator behavior, shared memory, and workload patterns complicate interpretation.

### Memory fragmentation

Allocators reserve memory in blocks. After many allocations and frees, Redis may have memory reserved that is not immediately returned to the OS.

\`\`\`mermaid
flowchart LR
    A["Allocate objects"] --> B["Delete some objects"]
    B --> C["Free gaps in allocated memory"]
    C --> D["RSS may stay elevated"]
\`\`\`

Redis may support active defragmentation depending on build and configuration. Never enable it blindly without measuring CPU and memory impact.

## 11. Key overhead vs value overhead

A million separate tiny keys often use much more memory than just the raw bytes of their values because each key has metadata, dictionary entries, allocations, and optional TTL bookkeeping.

Compare:

\`\`\`text
Option A:
user:1:name -> Asha
user:1:city -> Bengaluru

Option B:
user:1 -> Hash { name: Asha, city: Bengaluru }
\`\`\`

For many small related fields, Hashes may save memory.

But a single enormous Hash can become a hot key, complicate TTL boundaries, and make some operations expensive.

**Do not optimize only for memory.** Also evaluate access patterns, latency, correctness, and maintenance.

## 12. CPU vs memory trade-offs

Compact encodings can save memory but sometimes require scanning or unpacking data to find or update elements.

General structures can use more memory but provide better performance for larger collections.

\`\`\`mermaid
flowchart TD
    A["Choose representation"] --> B["Compact encoding"]
    A --> C["General encoding"]
    B --> D["Lower overhead for small values"]
    B --> E["Potentially more scanning"]
    C --> F["Higher metadata overhead"]
    C --> G["Better scaling for large collections"]
\`\`\`

Redis makes many of these choices automatically. Application developers should generally focus on **data type and access pattern**, then measure real memory and latency.

## 13. Production optimization workflow

1. **Measure:** Identify high-memory keys and types.
2. **Understand:** Is the cost in values, key names, TTL metadata, or collection overhead?
3. **Reproduce:** Create representative data in a staging environment.
4. **Compare:** Test alternatives such as String JSON vs Hash fields.
5. **Benchmark:** Measure CPU, latency, and memory under realistic operations.
6. **Migrate safely:** Version keys or implement a gradual migration.
7. **Monitor:** Watch memory, eviction, hit rate, and application errors.

### What not to do

- Do not change encoding thresholds across production without testing.
- Do not rely on one small sample key to predict millions of keys.
- Do not assume \`MEMORY USAGE\` is identical to serialized payload size.
- Do not use \`KEYS *\` for large production scans.
- Do not trade away required atomicity or TTL behavior merely to save bytes.

## 14. Node.js memory-inspection script

Install:

\`\`\`bash
npm install redis
\`\`\`

Use ESM.

\`\`\`js
import { createClient } from "redis";

const redis = createClient({
  url: process.env.REDIS_URL ?? "redis://127.0.0.1:6379",
});

redis.on("error", (err) => console.error("Redis error:", err.message));

try {
  await redis.connect();

  await redis.set("demo:encoding:int", "42");
  await redis.hSet("demo:encoding:hash", {
    name: "Asha",
    city: "Bengaluru",
  });
  await redis.sAdd("demo:encoding:set", ["react", "redis"]);

  const keys = [
    "demo:encoding:int",
    "demo:encoding:hash",
    "demo:encoding:set",
  ];

  for (const key of keys) {
    const [type, encoding, bytes] = await Promise.all([
      redis.type(key),
      redis.sendCommand(["OBJECT", "ENCODING", key]),
      redis.memoryUsage(key),
    ]);

    console.log({ key, type, encoding, bytes });
  }
} catch (error) {
  console.error("Inspection failed:", error.message);
  process.exitCode = 1;
} finally {
  if (redis.isOpen) await redis.quit();
}
\`\`\`

**Note:** The encoding output can differ across Redis versions. That is expected.

## 15. Common mistakes

1. Treating logical data type and internal encoding as the same concept.
2. Assuming all Strings occupy their text length in memory.
3. Assuming every Hash is a hashtable.
4. Assuming Lists store one linked-list node per element.
5. Assuming all Sets use intset or all Sorted Sets use skiplist.
6. Depending on encoding names or thresholds in application correctness logic.
7. Ignoring the memory impact of a huge Bitmap offset.
8. Measuring only value bytes and ignoring key metadata.
9. Assuming a small collection stays in a compact encoding forever.
10. Treating high RSS as proof of a leak without investigating fragmentation.
11. Optimizing memory while making important operations slower.
12. Ignoring version differences in Redis internals.

## 16. Hands-on exercises

1. Create a String and inspect \`TYPE\`, \`OBJECT ENCODING\`, and \`MEMORY USAGE\`.
2. Compare an integer String with a short text String.
3. Append to a String and inspect whether encoding changes.
4. Create a small Hash and inspect its encoding.
5. Read Hash listpack thresholds from \`CONFIG GET\`.
6. Cross a Hash threshold in a disposable local instance.
7. Create a List and inspect its encoding.
8. Create an integer-only Set, then add a text member.
9. Inspect a small Sorted Set.
10. Compare memory usage of 100 separate String keys vs one Hash with 100 fields.
11. Create a small Bitmap and estimate its raw bit storage.
12. Inspect \`INFO memory\` before and after deleting demo keys.
13. Explain why \`used_memory_rss\` can exceed \`used_memory\`.
14. Run the Node.js inspection script.

## 17. Interview questions

**Q1. What is an internal encoding?** Redis's representation of a logical value in memory.

**Q2. Does TYPE show encoding?** No; use \`OBJECT ENCODING\`.

**Q3. Why does Redis use compact encodings?** To reduce memory overhead for small collections.

**Q4. Why can a Hash switch from listpack to hashtable?** Its size or field/value lengths may cross configured thresholds.

**Q5. What is quicklist?** A list representation combining linked nodes with packed elements.

**Q6. What is intset?** A compact representation for suitable integer-only Sets.

**Q7. Why use skiplist in Sorted Sets?** Efficient ordered access and range queries.

**Q8. What is memory fragmentation?** Allocated/resident memory can remain unevenly used after allocation and deletion.

**Q9. Is MEMORY USAGE equal to payload length?** No; it includes additional overhead.

**Q10. Should applications depend on encoding thresholds?** No; encodings are implementation details and version-dependent.

## 18. Revision checklist

- [ ] Logical type vs internal encoding
- [ ] \`TYPE\`, \`OBJECT ENCODING\`, \`MEMORY USAGE\`
- [ ] String integer / embstr / raw
- [ ] Hash listpack / hashtable
- [ ] List quicklist
- [ ] Set compact / intset / hashtable representations
- [ ] Sorted Set listpack / skiplist
- [ ] Bitmap, HyperLogLog, GEO, Stream storage
- [ ] Encoding conversions
- [ ] Key metadata overhead
- [ ] Memory fragmentation
- [ ] CPU vs memory trade-offs
- [ ] Version compatibility
- [ ] Measurement and safe optimization

---

**Official references:** [Redis memory optimization](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/) · [OBJECT ENCODING](https://redis.io/docs/latest/commands/object-encoding/) · [MEMORY USAGE](https://redis.io/docs/latest/commands/memory-usage/) · [INFO](https://redis.io/docs/latest/commands/info/)
