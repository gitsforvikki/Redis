# Lesson 013 — Redis Sorted Sets

> **Section 02 — Data Types and Commands** | Beginner → Advanced  
> **Goal:** Understand score-based ordering, leaderboards, ranking, range queries, atomic updates, time-window modeling, and practical Node.js integration.

**Prerequisite:** [Lesson 012 — Redis Sets](012-sets.md)

## 1. What is a Sorted Set?

A Redis **Sorted Set** (also called a **ZSET**) stores **unique members**, each associated with a **numeric score**. Redis maintains members ordered by score.

\`\`\`text
Key: leaderboard:weekly

Member       Score
------------------
asha         120
vikram       175
neha         210
\`\`\`

\`\`\`mermaid
flowchart LR
    A["asha: 120"] --> B["vikram: 175"] --> C["neha: 210"]
\`\`\`

It combines ideas from:
- **Set:** members are unique.
- **Ranking structure:** every member has a score.
- **Ordered index:** query lowest/highest scores, rank, or score ranges.

**Important:** Members are unique, **scores do not have to be unique**. Equal-score members are ordered lexicographically by member bytes for ascending ordering.

### Real uses

- Game leaderboards
- Most active contributors
- Time-based indexes
- Priority-oriented work selection
- Rate-limit sliding-window event indexes
- Scheduling candidates by timestamp

Sorted Sets do not automatically execute scheduled tasks, acknowledge work, or enforce fairness. They are a building block.

## 2. ZADD — insert or update scores

\`\`\`redis
DEL demo:leaderboard
ZADD demo:leaderboard 120 asha 175 vikram 210 neha
\`\`\`

The result is the number of **new members** added, not the number of existing scores updated.

\`\`\`redis
ZADD demo:leaderboard 230 asha
ZSCORE demo:leaderboard asha
\`\`\`

Asha's score becomes 230; no duplicate member is created.

### Important ZADD options

| Option | Meaning |
| --- | --- |
| \`NX\` | Add only new members |
| \`XX\` | Update only existing members |
| \`GT\` | Update only if new score is greater |
| \`LT\` | Update only if new score is less |
| \`CH\` | Return count of changed members, not just newly added |
| \`INCR\` | Increment a member's score instead of setting it |

Check command compatibility and option combinations in the official docs. \`NX\`, \`GT\`, and \`LT\` cannot be combined arbitrarily.

## 3. ZSCORE, ZCARD, ZREM

\`\`\`redis
ZSCORE demo:leaderboard neha
ZCARD demo:leaderboard
ZREM demo:leaderboard vikram
ZCARD demo:leaderboard
\`\`\`

- \`ZSCORE\`: retrieve one member's score; nil if absent.
- \`ZCARD\`: count unique members.
- \`ZREM\`: remove members and return how many were removed.

\`ZMSCORE\` retrieves scores for multiple members in a single command (Redis 6.2+):

\`\`\`redis
ZMSCORE demo:leaderboard asha neha unknown
\`\`\`

## 4. ZRANGE — retrieve members in order

Prepare predictable data:

\`\`\`redis
DEL demo:rank
ZADD demo:rank 10 alice 20 bob 30 charlie 40 diana
\`\`\`

### Ascending order

\`\`\`redis
ZRANGE demo:rank 0 -1
ZRANGE demo:rank 0 -1 WITHSCORES
\`\`\`

Output order: alice, bob, charlie, diana.

### Descending order

\`\`\`redis
ZRANGE demo:rank 0 -1 REV WITHSCORES
\`\`\`

Output order: diana, charlie, bob, alice.

\`REV\` is a modern \`ZRANGE\` option. Older code may use \`ZREVRANGE\`, which remains familiar but is superseded by the unified \`ZRANGE\` syntax in newer Redis versions.

### Top 3 leaderboard

\`\`\`redis
ZRANGE demo:rank 0 2 REV WITHSCORES
\`\`\`

**Remember:** Like \`LRANGE\`, the end index is inclusive.

## 5. Rank vs score

A **score** is the numeric value stored for a member.

A **rank** is its position in the sorted ordering.

\`\`\`redis
ZRANK demo:rank alice
ZRANK demo:rank diana
ZREVRANK demo:rank diana
\`\`\`

- \`ZRANK\`: zero-based ascending position.
- \`ZREVRANK\`: zero-based descending position.
- Missing member: nil.

### Display human-friendly rank

If \`ZREVRANK\` returns \`0\`, display **rank 1** to the user:

\`\`\`js
const displayRank = zeroBasedRank === null ? null : zeroBasedRank + 1;
\`\`\`

**Tie behavior:** Equal scores still get distinct rank positions because members have a deterministic secondary lexicographic ordering. If your product requires equal scores to share the same displayed rank, implement that rule separately.

## 6. Score-range queries

Use \`ZRANGE ... BYSCORE\`:

\`\`\`redis
ZRANGE demo:rank 15 35 BYSCORE WITHSCORES
\`\`\`

Returns bob (20) and charlie (30).

### Exclusive boundaries

\`\`\`redis
ZRANGE demo:rank (20 40 BYSCORE WITHSCORES
\`\`\`

This means score **greater than 20** and **up to 40 inclusive**.

### Pagination within a score range

\`\`\`redis
ZRANGE demo:rank 0 +inf BYSCORE LIMIT 0 2 WITHSCORES
\`\`\`

Here \`LIMIT 0 2\` asks for two members starting at offset zero.

**Performance warning:** Large offsets can require extra work. For very large leaderboards, design pagination deliberately rather than assuming offset pagination is free.

## 7. ZCOUNT — count by score range

\`\`\`redis
ZCOUNT demo:rank 20 40
\`\`\`

Returns \`3\` for bob, charlie, diana.

Useful when you need a count rather than transferring all matching members.

## 8. ZINCRBY — atomic score increments

\`\`\`redis
ZINCRBY demo:rank 5 bob
ZSCORE demo:rank bob
\`\`\`

Bob's score becomes 25.

### Why not ZSCORE followed by ZADD?

\`\`\`mermaid
sequenceDiagram
    participant A as API A
    participant R as Redis
    participant B as API B
    A->>R: ZSCORE bob -> 20
    B->>R: ZSCORE bob -> 20
    A->>R: ZADD bob 25
    B->>R: ZADD bob 25
    Note over R: One increment lost
\`\`\`

Use \`ZINCRBY\` for an atomic single-command increment.

**Precision note:** Sorted Set scores are IEEE 754 double-precision floating-point values. Integers are exactly represented only within the safe range (roughly ±2^53). Do not use ZSET scores as a replacement for precise financial ledgers.

## 9. Remove by rank or score

\`\`\`redis
ZREMRANGEBYRANK demo:rank 0 0
ZREMRANGEBYSCORE demo:rank -inf 15
\`\`\`

- \`ZREMRANGEBYRANK\` removes a range by **ascending rank**.
- \`ZREMRANGEBYSCORE\` removes members whose scores fall within a range.

### Keep only the top 100

After adding or updating a member:

\`\`\`redis
ZREMRANGEBYRANK leaderboard:weekly 0 -101
\`\`\`

This removes everything except the 100 highest-ranked entries, if there are more than 100. The example is a single command, but pairing it with \`ZADD\` requires a transaction or script if atomic update-and-trim matters.

**Important:** Removing low-ranking users from the main leaderboard loses their scores and ranks. Only do this if a top-100-only dataset is actually your requirement.

## 10. ZPOPMIN, ZPOPMAX and blocking pops

\`\`\`redis
DEL demo:priority
ZADD demo:priority 5 low 1 urgent 3 medium
ZPOPMIN demo:priority
ZPOPMAX demo:priority
\`\`\`

- \`ZPOPMIN\`: remove lowest-score member.
- \`ZPOPMAX\`: remove highest-score member.
- \`BZPOPMIN\` / \`BZPOPMAX\`: wait for an available member.

**Important:** Popping removes an item immediately. A worker crash after pop can lose it. Reliable job execution requires processing-state tracking, retries, idempotency, and recovery.

## 11. ZSCAN and large Sorted Sets

\`\`\`redis
ZSCAN demo:rank 0 COUNT 10
\`\`\`

Like other SCAN-family commands:
- Use the returned cursor until it becomes zero.
- \`COUNT\` is a hint.
- Duplicates may appear.
- Concurrent modifications mean no consistent snapshot.
- Scanned results are **not sorted in score order**.

Use \`ZRANGE\` for ranked ordering; use \`ZSCAN\` for incremental inspection/processing.

## 12. Lexicographical ranges and equal scores

When members have equal scores, Redis can use lexicographical ordering.

\`\`\`redis
DEL demo:names
ZADD demo:names 0 anna 0 bob 0 carla 0 dan
ZRANGE demo:names [b [d BYLEX
\`\`\`

This returns names in the specified inclusive lexicographic range (subject to lexicographic comparison rules).

**Important:** Lex-range operations are designed for Sorted Sets whose members share the same score. Mixed scores can make results unintuitive; do not use this as a general full-text search substitute.

## 13. Real-world project: leaderboard

A gaming service awards points:

\`\`\`redis
ZINCRBY game:season:2026 50 user:42
ZINCRBY game:season:2026 30 user:99
ZINCRBY game:season:2026 20 user:42
ZRANGE game:season:2026 0 9 REV WITHSCORES
ZREVRANK game:season:2026 user:42
\`\`\`

\`\`\`mermaid
flowchart LR
    A["Game event"] --> B["Validate and record event"]
    B --> C["ZINCRBY leaderboard"]
    C --> D["ZRANGE REV top 10"]
    C --> E["ZREVRANK user"]
\`\`\`

**Correctness considerations:**
- Validate score awards server-side.
- Prevent duplicate awards from retried events.
- Keep an authoritative event/score record if recovery matters.
- Decide tie ranking rules.
- Define season rollover and retention.

## 14. Real-world project: time-based index

A Sorted Set score can represent a Unix timestamp.

\`\`\`redis
ZADD scheduled:reminders 1893456000 reminder:101
ZADD scheduled:reminders 1893456060 reminder:102
ZRANGE scheduled:reminders -inf 1893456000 BYSCORE
\`\`\`

This finds entries scheduled no later than the supplied timestamp.

**Important:** The timestamp is an example. A Sorted Set does not automatically send reminders. A scheduler/worker must poll or wait, claim work safely, execute it, and handle retries.

### Sliding-window rate limiting concept

A ZSET can store request IDs as members and request timestamps as scores:

\`\`\`text
rate:user:42
  req-a -> 1000.100
  req-b -> 1000.350
  req-c -> 1000.900
\`\`\`

The algorithm removes events older than the window, counts the remainder, and conditionally inserts the new request. Those steps must be made atomic (often with a Lua script/function) to avoid races.

We implement robust rate limiting in Lesson 042.

## 15. Node.js practical example

Install:

\`\`\`bash
npm install redis
\`\`\`

Use an ESM project (\`"type": "module"\`).

\`\`\`js
import { createClient } from "redis";

const redis = createClient({
  url: process.env.REDIS_URL ?? "redis://127.0.0.1:6379",
});

redis.on("error", (error) => console.error("Redis:", error.message));

const key = "demo:leaderboard:lesson13";

try {
  await redis.connect();

  await redis.del(key);
  await redis.zAdd(key, [
    { score: 100, value: "asha" },
    { score: 150, value: "neha" },
    { score: 120, value: "vikram" },
  ]);

  await redis.zIncrBy(key, 40, "asha");

  const top = await redis.zRangeWithScores(key, 0, 2, { REV: true });
  const rank = await redis.zRevRank(key, "asha");
  const score = await redis.zScore(key, "asha");

  console.log("Top players:", top);
  console.log("Asha's rank:", rank === null ? null : rank + 1);
  console.log("Asha's score:", score);
} catch (error) {
  console.error("Leaderboard failed:", error.message);
  process.exitCode = 1;
} finally {
  if (redis.isOpen) await redis.quit();
}
\`\`\`

**Expected:** Asha becomes 140, Neha remains 150, and Vikram remains 120. Neha ranks first, Asha second, Vikram third.

### API architecture considerations

- Authenticate users and validate events before awarding points.
- Never let the client choose arbitrary points.
- Cache invalidation and score updates need clear ownership.
- Plan Redis downtime and rebuilding from the primary database.
- Redis Cluster multi-key ZSET operations have hash-slot constraints.

## 16. Sorted Set vs Set vs List

| Requirement | Best starting point | Reason |
| --- | --- | --- |
| Unique members, no ranking | Set | Efficient membership |
| Ordered sequence with duplicates | List | Preserve insertion order |
| Ranking by numeric score | Sorted Set | Ordered by score |
| Time-indexed entries | Sorted Set | Query by timestamp score |
| Durable work queue with ack/retry | Streams/BullMQ | More suitable processing guarantees |
| Exact financial transaction ledger | Database | Durability and integrity requirements |

## 17. Performance and scaling

| Command | Typical time complexity |
| --- | --- |
| \`ZADD\` | O(log N) per member |
| \`ZINCRBY\` | O(log N) |
| \`ZREM\` | O(log N) per member |
| \`ZSCORE\` | O(1) |
| \`ZCARD\` | O(1) |
| \`ZRANK\` / \`ZREVRANK\` | O(log N) |
| \`ZRANGE\` | O(log N + M) for M returned members |
| \`ZCOUNT\` | O(log N) |
| \`ZPOPMIN\` | O(log N) for one member |

Time complexity is not the whole story. Very large result ranges cost network bandwidth and client memory. Scores, member lengths, and internal encodings also affect memory usage.

## 18. Common mistakes

1. Assuming scores must be unique — they do not.
2. Confusing \`ZRANK\` with \`ZREVRANK\`.
3. Forgetting ranks are zero-based.
4. Assuming ties automatically share the same displayed rank.
5. Using \`ZRANGE 0 -1\` on huge leaderboards.
6. Implementing score increments with separate \`ZSCORE\` and \`ZADD\`.
7. Treating \`ZPOPMIN\` as a complete reliable job queue.
8. Assuming Sorted Sets trigger tasks when timestamps arrive.
9. Using floating-point scores for exact financial accounting.
10. Assuming \`ZSCAN\` returns sorted results.
11. Ignoring version differences in \`ZRANGE\` options.
12. Ignoring cluster multi-key constraints.

## 19. Hands-on exercises

1. Create a leaderboard with five users and scores.
2. Update one user's score using \`ZADD\`.
3. Increment a user's score using \`ZINCRBY\`.
4. Display the top three members using \`ZRANGE ... REV WITHSCORES\`.
5. Find a user's descending rank with \`ZREVRANK\`.
6. Count members with scores between 50 and 100.
7. Remove the lowest-ranked member.
8. Model reminder IDs using timestamp scores.
9. Use \`ZSCAN\` and explain why the output is not sorted.
10. Build a Node.js leaderboard endpoint.
11. Explain how duplicate score events could be prevented.
12. Explain why a time-indexed ZSET does not replace a durable scheduler.

## 20. Interview questions

**Q1. What is a Sorted Set?** Unique members with numeric scores, ordered by score.

**Q2. Can two members share a score?** Yes.

**Q3. ZRANK vs ZREVRANK?** Ascending vs descending zero-based rank.

**Q4. Is ZINCRBY atomic?** Yes, as a single Redis command.

**Q5. How do you get the top 10?** \`ZRANGE key 0 9 REV WITHSCORES\` on a compatible Redis version.

**Q6. Does ZSCAN return sorted members?** No.

**Q7. How can Sorted Sets model scheduled tasks?** Store timestamps as scores and query due members, but add a worker and reliable processing.

**Q8. What happens when ZADD updates an existing member?** Its score changes; no duplicate member is created.

**Q9. Are ZSET scores precise for arbitrary large integers?** No; they are double-precision floating-point numbers.

**Q10. Sorted Set vs Set?** Both store unique members, but Sorted Sets attach scores and maintain ranking.

## 21. Revision checklist

- [ ] Unique members with numeric scores
- [ ] \`ZADD\`, \`ZSCORE\`, \`ZCARD\`, \`ZREM\`
- [ ] \`ZRANGE\` and \`REV\`
- [ ] \`ZRANK\` and \`ZREVRANK\`
- [ ] Score ranges and \`ZCOUNT\`
- [ ] \`ZINCRBY\` atomicity
- [ ] \`ZPOPMIN\` / \`ZPOPMAX\`
- [ ] \`ZSCAN\` limitations
- [ ] Lexicographic tie ordering
- [ ] Leaderboards, time indexes and scheduling caveats
- [ ] Node.js integration and scaling

---

**Previous:** [Lesson 012 — Redis Sets](012-sets.md)  
**Next:** [Lesson 014 — Bitmaps and Bitfields](014-bitmaps-bitfields.md)

**Official references:** [Redis Sorted Sets](https://redis.io/docs/latest/develop/data-types/sorted-sets/) · [ZADD](https://redis.io/docs/latest/commands/zadd/) · [ZRANGE](https://redis.io/docs/latest/commands/zrange/) · [ZINCRBY](https://redis.io/docs/latest/commands/zincrby/)
