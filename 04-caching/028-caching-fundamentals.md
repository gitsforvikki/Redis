# Lesson 028 — Redis Caching Fundamentals

> **Section 04 — Caching in Real Applications** | Beginner → Advanced  
> **Goal:** Understand what a cache is, why it improves applications, cache hits/misses, freshness, TTL, consistency, failure handling, and when Redis caching is a good choice.

**Previous:** [Lesson 027 — Lua Scripting and Redis Functions](../03-internals/027-lua-functions.md)  
**Next:** [Lesson 029 — Cache-Aside Pattern](029-cache-aside.md)

## 1. What is caching?

A **cache** is a place where an application keeps a copy of data that is expensive or slow to obtain repeatedly.

Example: Your CareerLoop dashboard needs application counts. PostgreSQL is the source of truth. Instead of calculating counts on every request, you can store a recent result in Redis.

\`\`\`mermaid
flowchart LR
    U["Browser"] --> API["Next.js API"]
    API --> R[("Redis cache")]
    R -. "Cache miss" .-> DB[("PostgreSQL")]
    DB -. "Load result" .-> R
\`\`\`

**Key principle:** A cache is usually a **derived copy**, not the authoritative data store. If the cached value disappears, the application should know how to fetch or reconstruct it.

## 2. Why caching matters

Suppose a product API runs a complex query taking 80 ms. Redis access might take a few milliseconds under favorable conditions.

Without cache:

\`\`\`text
Request -> API -> PostgreSQL -> API -> Response
\`\`\`

With cache hit:

\`\`\`text
Request -> API -> Redis -> API -> Response
\`\`\`

Benefits can include:
- Lower response latency.
- Reduced database queries.
- Higher throughput.
- Lower backend cost.
- Reduced load during traffic spikes.

But caching introduces new problems:
- Stale data.
- Invalidation.
- Memory limits and eviction.
- Cache stampedes.
- Additional infrastructure.
- Failure handling.

**Caching is a trade-off, not a free speed upgrade.**

## 3. Cache hit and cache miss

### Cache hit

The requested value is present and usable.

\`\`\`mermaid
sequenceDiagram
    participant U as User
    participant API as API
    participant R as Redis
    U->>API: GET /companies/101
    API->>R: GET cache:company:101
    R-->>API: Cached JSON
    API-->>U: Response
\`\`\`

### Cache miss

The value is absent, expired, evicted, or unusable. The application fetches it from the source of truth.

\`\`\`mermaid
sequenceDiagram
    participant U as User
    participant API as API
    participant R as Redis
    participant DB as PostgreSQL
    U->>API: GET /companies/101
    API->>R: GET cache:company:101
    R-->>API: null
    API->>DB: SELECT company
    DB-->>API: Company row
    API->>R: SET cache with TTL
    API-->>U: Company response
\`\`\`

**Important:** Redis returning \`null\` for a missing cache key does not prove that the underlying company does not exist.

## 4. Cache hit rate

A simple hit-rate formula:

\`\`\`text
Hit rate = cache hits / (cache hits + cache misses)
\`\`\`

Example:

\`\`\`text
Hits = 900
Misses = 100

Hit rate = 900 / 1000 = 90%
\`\`\`

A higher hit rate can be helpful, but it does not always mean a better application.

Example:
- Caching a very cheap lookup may raise hit rate without meaningful savings.
- Caching an expensive query used 10 times per minute may deliver major benefits even with fewer hits.
- Stale results can be incorrect despite a high hit rate.

### What else should you measure?

- p50, p95, and p99 response latency.
- Database query volume and CPU.
- Cache memory use.
- Redis command latency.
- Evictions.
- Cache freshness.
- Error rate.
- Cost of recomputing values.

## 5. What should you cache?

Good candidates:
- Public product descriptions.
- Frequently requested company details.
- Expensive dashboard aggregations.
- Read-heavy configuration.
- Trending content.
- Derived counts and rankings, when staleness is acceptable.

Poor candidates unless carefully designed:
- Payment authorization decisions.
- Security-sensitive permissions without a robust invalidation strategy.
- Rapidly changing inventory where stale reads cause overselling.
- Highly personalized data without tenant/user isolation.
- Very cheap data with low reuse.
- Data that cannot be safely serialized or reconstructed.

### Practical decision questions

1. Is the data read frequently?
2. Is the source lookup expensive?
3. Can a cached copy be slightly stale?
4. How will updates invalidate it?
5. How much memory will it use?
6. What happens if Redis is unavailable?
7. Is the cache key correctly scoped to the user and tenant?

## 6. Cache freshness and stale data

Imagine a product price changes from ₹1,000 to ₹1,200 in PostgreSQL, but Redis still contains ₹1,000.

\`\`\`mermaid
sequenceDiagram
    participant Admin as Admin
    participant DB as PostgreSQL
    participant R as Redis
    participant U as User
    Admin->>DB: Update price to 1200
    Note over R: Cache still says 1000
    U->>R: Read product price
    R-->>U: 1000 (stale)
\`\`\`

A cache may be **fast but incorrect**.

You need a freshness policy:
- TTL expiration.
- Explicit invalidation after updates.
- Versioned keys.
- Event-driven invalidation.
- Stale-while-revalidate where acceptable.

The correct choice depends on business requirements.

## 7. TTL: time to live

A TTL sets a maximum intended lifetime for a cache entry unless it is refreshed or removed earlier.

\`\`\`redis
SET cache:product:101 "{\"name\":\"Keyboard\"}" EX 300
TTL cache:product:101
\`\`\`

This cache entry expires after 300 seconds.

### Important TTL facts

- Keys may disappear **before** TTL due to eviction or deletion.
- A normal \`SET\` replaces the value and typically clears an existing TTL unless a new TTL or \`KEEPTTL\` is used.
- Reading with \`GET\` does not automatically refresh TTL.
- TTL does not guarantee the data remains fresh until expiration.

### Choosing TTL

| Data | Illustrative TTL | Reason |
| --- | --- | --- |
| Public catalog | 5–15 minutes | Moderate update frequency |
| Trending dashboard | 30–120 seconds | Frequent changes |
| Feature config | 1–5 minutes | Depends on rollout needs |
| Negative cache | 10–60 seconds | Avoid repeated missing lookups |
| User-specific summary | Short or event-invalidated | Reduce stale personalized data |

These are **examples**, not universal defaults.

## 8. Cache keys are part of correctness

Bad key:

\`\`\`text
dashboard:summary
\`\`\`

If every user shares this key, one user may receive another user's private dashboard.

Better:

\`\`\`text
careerloop:prod:user:42:dashboard:v1
\`\`\`

A useful cache key often includes:
- Application/environment.
- Tenant or user identity.
- Resource identity.
- Query parameters.
- Schema/version information.

Example:

\`\`\`text
careerloop:prod:user:42:applications:status=interview:page=1:v2
\`\`\`

**Security:** Cache key isolation does not replace authorization. Validate access before returning protected data.

## 9. Cache patterns overview

This section introduces the main patterns. Later lessons cover them in depth.

### Cache-aside (lazy loading)

The application checks Redis first, then loads from the database on a miss.

\`\`\`mermaid
flowchart TD
    A["Read request"] --> B["Check Redis"]
    B --> C{"Hit?"}
    C -->|"Yes"| D["Return cache"]
    C -->|"No"| E["Read database"]
    E --> F["Populate cache"]
    F --> G["Return result"]
\`\`\`

### Read-through

A caching layer handles loading data from the source when the cache misses.

### Write-through

Writes update the backing store and cache through a coordinated caching layer.

### Write-behind (write-back)

Writes reach the cache first, then are asynchronously propagated to the backing store.

### Write-around

Writes bypass the cache and go directly to the backing store, reducing pollution from data unlikely to be reread.

**Important:** Redis does not automatically implement every pattern just because you call \`GET\` and \`SET\`. Most patterns require application or library logic.

## 10. Cache invalidation: the hardest part

When PostgreSQL data changes, how does Redis learn that its cached copy is stale?

### Delete after successful update

\`\`\`text
1. UPDATE company in PostgreSQL
2. DEL cache:company:101 in Redis
3. Next read refills the cache
\`\`\`

This is a common starting pattern.

But there is a race:

\`\`\`text
Reader: cache miss
Reader: fetches old database value
Writer: updates database
Writer: deletes cache
Reader: writes old value into cache
\`\`\`

The cache can become stale again after invalidation.

**TTL limits the damage window but does not fully solve this race.** Stronger correctness may require versioning, coordinated writes, or different architecture.

## 11. Cache stampede

Suppose one popular cache key expires. A thousand requests arrive at once.

\`\`\`mermaid
flowchart TD
    A["Hot key expires"] --> B["1000 cache misses"]
    B --> C["1000 database queries"]
    C --> D["Database overloaded"]
    D --> E["Slow API responses"]
\`\`\`

Common mitigations:
- Request coalescing / single-flight.
- Short-lived distributed lock with safe fallback.
- Stale-while-revalidate.
- Prewarming popular keys.
- TTL jitter for groups of keys.
- Rate limiting and database protection.

**TTL jitter does not by itself prevent a stampede on one very popular key.**

## 12. Cache penetration and negative caching

A client repeatedly requests a company ID that does not exist.

Without negative caching:

\`\`\`text
Redis miss -> PostgreSQL miss
Redis miss -> PostgreSQL miss
Redis miss -> PostgreSQL miss
\`\`\`

You can temporarily cache the fact that the resource was not found.

Example:

\`\`\`redis
SET cache:company:missing:999 "__NOT_FOUND__" EX 30
\`\`\`

**Important:** Use an explicit marker distinct from legitimate data. Negative caching must have a short TTL or invalidation strategy because a resource may later be created.

For private resources, do not cache “not found” in a way that leaks authorization decisions across users.

## 13. Cache avalanche

A **cache avalanche** occurs when many cache entries become unavailable together, causing a large surge of requests to the backend.

Causes include:
- Redis outage.
- Synchronized TTL expiration.
- Mass eviction.
- Cold cache after deployment or restart.

Mitigations:
- TTL jitter.
- Gradual cache warming.
- Request coalescing.
- Backend rate limits and bulkheads.
- Replication/high availability where appropriate.
- Graceful degradation.

Stampede often describes many requests for the **same** hot key; avalanche describes **many keys** becoming unavailable around the same time.

## 14. Redis memory and eviction

Redis has limited memory. When configured with \`maxmemory\` and an eviction policy, it may remove cached keys.

Common cache policies:
- \`allkeys-lru\`: approximate least recently used.
- \`allkeys-lfu\`: approximate least frequently used.

\`\`\`redis
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
INFO stats
INFO memory
\`\`\`

**Cache correctness principle:** The application must handle a missing key even if it was just written.

For dedicated caches, allkeys policies are often useful starting points, but benchmark with your workload.

## 15. Node.js + Redis + PostgreSQL mental model

\`\`\`mermaid
flowchart TD
    U["Browser"] --> API["Next.js / Express API"]
    API --> R[("Redis: fast derived data")]
    API --> DB[("PostgreSQL: authoritative records")]
    DB --> API
    API --> U
\`\`\`

Do not make Redis the only place storing a business-critical application record merely because it is fast.

## 16. Practical Node.js cache helper

Install:

\`\`\`bash
npm install redis
\`\`\`

ESM example:

\`\`\`js
import { createClient } from "redis";

const redis = createClient({
  url: process.env.REDIS_URL ?? "redis://127.0.0.1:6379",
  socket: {
    connectTimeout: 1000,
  },
});

redis.on("error", (err) => console.error("Redis:", err.message));

async function readCompanyFromDatabase(id) {
  // Replace with a real PostgreSQL query.
  return { id, name: "Example Company" };
}

async function getCompany(id) {
  const key = \`careerloop:dev:company:\${id}:v1\`;

  if (redis.isReady) {
    try {
      const raw = await redis.get(key);
      if (raw !== null) {
        return JSON.parse(raw);
      }
    } catch (err) {
      console.warn("Cache read failed:", err.message);
    }
  }

  const company = await readCompanyFromDatabase(id);

  if (redis.isReady) {
    try {
      await redis.set(key, JSON.stringify(company), { EX: 300 });
    } catch (err) {
      console.warn("Cache write failed:", err.message);
    }
  }

  return company;
}

try {
  try {
    await redis.connect();
  } catch {
    console.warn("Redis unavailable; using database fallback");
  }

  console.log(await getCompany(101));
} finally {
  if (redis.isOpen) await redis.quit();
}
\`\`\`

### What this example teaches

- Cache hit returns a stored JSON copy.
- Cache miss reads from the source.
- Cache write uses a TTL.
- Redis errors do not automatically make the API fail if the database is healthy.

### What this example does NOT solve

- Cache stampedes.
- Database overload during Redis outages.
- Cross-user authorization.
- Schema changes.
- Cache invalidation after updates.
- Query timeouts and circuit-breaker behavior.
- Strong consistency requirements.

These are important production concerns, not optional details.

## 17. Serialization and data integrity

Redis Strings store bytes. Applications often serialize objects as JSON.

\`\`\`js
const payload = JSON.stringify({ id: 101, createdAt: new Date() });
const parsed = JSON.parse(payload);
\`\`\`

After parsing, \`createdAt\` is a **string**, not a JavaScript \`Date\` instance.

Other pitfalls:
- \`undefined\` properties can be omitted.
- \`BigInt\` cannot be serialized by ordinary \`JSON.stringify\` without conversion.
- Circular references cause errors.
- Large objects increase network and memory costs.
- Schema changes can make old cache entries incompatible.

Use schema validation and versioned keys for important cached data.

## 18. Multi-level caching

Modern applications may have several cache layers:

\`\`\`mermaid
flowchart TD
    A["Browser cache"] --> B["CDN / edge cache"]
    B --> C["Next.js / application cache"]
    C --> D["Redis distributed cache"]
    D --> E["PostgreSQL"]
\`\`\`

Each layer has different:
- Scope.
- Freshness controls.
- Invalidation behavior.
- Sharing model.
- Failure modes.

**Next.js note:** Framework caching and Redis caching are separate mechanisms. Adding Redis does not automatically configure Next.js route/data caching or its revalidation behavior.

## 19. When caching can make performance worse

Caching may hurt when:
- Almost every request has a unique key.
- The original query is already very cheap.
- Serialization is expensive.
- Cache hit rate is low.
- Redis is far away and adds network latency.
- Invalidation complexity causes repeated recomputation.
- The cache introduces extra failure points.

Always compare real before/after metrics.

## 20. Practical caching checklist

Before caching an endpoint, document:

1. **Source of truth:** Where is the authoritative value?
2. **Cache key:** Does it include user/tenant/query scope?
3. **Value format:** JSON, Hash, or another type?
4. **TTL:** What staleness is acceptable?
5. **Invalidation:** What happens after a write?
6. **Miss behavior:** How is the value loaded?
7. **Outage behavior:** What if Redis is unavailable?
8. **Stampede control:** What if many clients miss simultaneously?
9. **Memory:** How large and popular are the keys?
10. **Metrics:** How will you prove the cache helps?

## 21. Common mistakes

1. Treating Redis as the source of truth for every cached value.
2. Assuming a high hit rate guarantees good performance.
3. Caching sensitive data without user/tenant isolation.
4. Returning stale prices or permissions without a correctness policy.
5. Forgetting TTL or invalidation.
6. Assuming keys survive until TTL ends.
7. Treating a cache miss as “database record not found.”
8. Ignoring stampedes and avalanches.
9. Assuming TTL jitter solves every hot-key problem.
10. Failing open for critical security or rate-limit checks.
11. Ignoring Redis outages and database fallback overload.
12. Treating Next.js and Redis caches as the same layer.
13. Ignoring JSON serialization limitations.
14. Adding a cache without measuring benefits.

## 22. Hands-on exercises

1. Explain a cache in one sentence.
2. Draw cache-hit and cache-miss flows.
3. Calculate hit rate for 950 hits and 50 misses.
4. List three good and three poor cache candidates.
5. Design a user-scoped CareerLoop dashboard key.
6. Choose a TTL for a frequently updated company summary.
7. Explain a stale-cache scenario after PostgreSQL update.
8. Describe the cache invalidation race.
9. Explain cache stampede vs avalanche.
10. Design negative caching for a missing product.
11. Implement a Redis cache helper with a PostgreSQL fallback.
12. Explain why a cached Date becomes a string after JSON parsing.
13. Identify metrics to measure before and after caching.
14. Describe what your API does when Redis is down.

## 23. Interview questions

**Q1. What is caching?** Keeping reusable copies of data to reduce repeated expensive work.

**Q2. What is a cache hit?** The requested usable value is present in the cache.

**Q3. What is a cache miss?** The cache cannot supply the requested value.

**Q4. What is hit rate?** Hits divided by hits plus misses.

**Q5. What is TTL?** A configured expiration lifetime for a key.

**Q6. What is stale data?** A cached copy that no longer matches the source of truth.

**Q7. What is cache invalidation?** Removing or updating cached data when it is no longer valid.

**Q8. What is cache-aside?** Application reads cache first and loads from the database on misses.

**Q9. What is a cache stampede?** Many requests recomputing the same missing popular value.

**Q10. What is cache avalanche?** Many keys becoming unavailable together, overwhelming the backend.

**Q11. What is negative caching?** Temporarily caching the absence of a result.

**Q12. Is Redis caching automatically consistent with PostgreSQL?** No; the application needs a consistency and invalidation strategy.

## 24. Revision checklist

- [ ] Cache purpose and trade-offs
- [ ] Cache hits and misses
- [ ] Hit rate and latency metrics
- [ ] Choosing cache candidates
- [ ] TTL and freshness
- [ ] Cache key design and security
- [ ] Cache-aside, read-through, write-through
- [ ] Write-behind and write-around
- [ ] Invalidation races
- [ ] Stampede and avalanche
- [ ] Negative caching
- [ ] Memory eviction
- [ ] Node.js cache helper
- [ ] Serialization concerns
- [ ] Multi-level caching
- [ ] Redis outage strategy

---

**Official references:** [Redis caching](https://redis.io/solutions/caching/) · [Redis cache eviction](https://redis.io/docs/latest/develop/reference/eviction/) · [Redis EXPIRE](https://redis.io/docs/latest/commands/expire/) · [Next.js caching](https://nextjs.org/docs/app/guides/caching)
