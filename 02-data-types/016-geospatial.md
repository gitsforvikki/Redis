# Lesson 016 — Redis Geospatial

> **Section 02 — Data Types and Commands** | Beginner → Advanced  
> **Goal:** Learn how Redis stores locations, finds nearby points, calculates distances, and powers location-aware applications safely.

**Prerequisite:** [Lesson 015 — Redis HyperLogLog](015-hyperloglog.md)

## 1. What is Redis Geospatial?

Redis supports commands for storing and querying **geographical coordinates**: longitude and latitude.

You can ask:
- Which delivery partners are within 3 km of a customer?
- Which stores are near a location?
- What is the approximate distance between two saved points?
- Which nearby locations should be returned, sorted by distance?

\`\`\`mermaid
flowchart TD
    C["Customer location"] --> R["Redis GEOSEARCH"]
    R --> A["Store A — nearby"]
    R --> B["Store B — nearby"]
    R --> D["Store C — outside radius (excluded)"]
\`\`\`

**Important:** Redis geospatial data is stored internally using a **Sorted Set**. It is not an independent top-level data type. Coordinates are encoded into geospatial scores.

## 2. Coordinate basics: longitude vs latitude

A geographic point is commonly represented as:

\`\`\`text
longitude, latitude
\`\`\`

- **Longitude:** east/west position, from -180 to +180 degrees.
- **Latitude:** north/south position. Redis geospatial indexing supports approximately -85.05112878 to +85.05112878 degrees.

Redis GEO commands require **longitude first, latitude second**.

\`\`\`text
GEOADD key longitude latitude member
\`\`\`

Many mapping libraries or APIs use other conventions. Mixing up latitude and longitude can produce wrong results or invalid-coordinate errors.

### Example coordinates

The following are **illustrative approximate coordinates** for learning, not verified business locations or precise navigation points:

| Member | Longitude | Latitude |
| --- | ---: | ---: |
| bengaluru:center | 77.5946 | 12.9716 |
| bengaluru:point2 | 77.6100 | 12.9800 |
| bengaluru:point3 | 77.6500 | 13.0100 |

We will use these sample coordinates consistently in exercises.

## 3. GEOADD — store locations

\`\`\`redis
DEL demo:places
GEOADD demo:places 77.5946 12.9716 center
GEOADD demo:places 77.6100 12.9800 point2
GEOADD demo:places 77.6500 13.0100 point3
\`\`\`

\`GEOADD\` adds or updates member coordinates.

A member name is unique within the geospatial key. Adding the same member with new coordinates **moves/updates** that member rather than creating a duplicate.

\`\`\`redis
GEOADD demo:places 77.6000 12.9750 center
\`\`\`

Now \`center\` has the updated location.

### GEOADD options

Modern Redis versions support options such as:
- \`NX\`: add only new members.
- \`XX\`: update only existing members.
- \`CH\`: report changed members as well as additions.

Always verify supported options against the server version.

## 4. GEOPOS — retrieve saved coordinates

\`\`\`redis
GEOPOS demo:places center point2 missing
\`\`\`

Returns coordinates for saved members, and nil for missing members.

**Important:** Coordinates are encoded and decoded through the internal geospatial representation. Returned values may differ slightly from the original decimal coordinates due to quantization.

This makes Redis geospatial suitable for many proximity searches, but not a replacement for exact surveying or high-precision geodesy.

## 5. GEODIST — distance between two members

\`\`\`redis
GEODIST demo:places center point2 km
\`\`\`

Supported units include:
- \`m\`: meters
- \`km\`: kilometers
- \`mi\`: miles
- \`ft\`: feet

\`\`\`redis
GEODIST demo:places center point3 m
\`\`\`

If a member is missing, Redis returns nil.

### What does the distance mean?

Redis computes a geographic distance approximation using a spherical Earth model. It is **not driving distance**, **not walking distance**, and **not ETA**.

\`\`\`mermaid
flowchart LR
    A["Point A"] -->|"Geographic distance"| B["Point B"]
    A -. "Actual roads may detour" .-> B
\`\`\`

For navigation, travel time, traffic, or road distance, use an appropriate routing service.

## 6. GEOSEARCH — find nearby locations

\`GEOSEARCH\` is the main modern command for querying nearby points.

### Search around longitude/latitude

\`\`\`redis
GEOSEARCH demo:places FROMLONLAT 77.5946 12.9716 BYRADIUS 5 km
\`\`\`

This searches within a **5 km radius** of the specified coordinate.

### Search around a stored member

\`\`\`redis
GEOSEARCH demo:places FROMMEMBER center BYRADIUS 5 km
\`\`\`

Useful when the center point is already stored.

### Include distances and coordinates

\`\`\`redis
GEOSEARCH demo:places FROMLONLAT 77.5946 12.9716 BYRADIUS 10 km ASC WITHDIST WITHCOORD
\`\`\`

- \`ASC\`: nearest first.
- \`DESC\`: farthest first.
- \`WITHDIST\`: include distance in the requested unit.
- \`WITHCOORD\`: include coordinates.
- \`WITHHASH\`: include the internal geospatial hash value.

### Limit results

\`\`\`redis
GEOSEARCH demo:places FROMLONLAT 77.5946 12.9716 BYRADIUS 10 km ASC COUNT 2
\`\`\`

Returns at most two matching members.

**Performance note:** \`COUNT\` limits the returned results, but query work still depends on the search area and data distribution. \`ANY\` can let Redis stop earlier once enough matches are found, but then you must **not** assume the nearest N results are guaranteed.

## 7. Search by rectangular area: BYBOX

You can search within a width/height box instead of a radius:

\`\`\`redis
GEOSEARCH demo:places FROMLONLAT 77.5946 12.9716 BYBOX 10 8 km ASC WITHDIST
\`\`\`

This means a geographic box approximately 10 km wide and 8 km high, centered on the supplied coordinates.

### Radius vs box

| Search shape | Good for |
| --- | --- |
| \`BYRADIUS\` | “Within 5 km of me” |
| \`BYBOX\` | “Inside this approximate rectangular region” |

A geographic box is not a city administrative boundary. If you need exact polygons, jurisdiction checks, or complex GIS analysis, use a suitable spatial database or GIS service.

## 8. GEOSEARCHSTORE — save search results

Redis can store the result of a geospatial search into a destination Sorted Set:

\`\`\`redis
GEOSEARCHSTORE demo:nearby demo:places FROMLONLAT 77.5946 12.9716 BYRADIUS 10 km
ZRANGE demo:nearby 0 -1
\`\`\`

The destination is a Sorted Set. By default, stored scores retain geospatial encoding so the result can still be used as geospatial data.

### STOREDIST option

\`\`\`redis
GEOSEARCHSTORE demo:distance-scores demo:places FROMLONLAT 77.5946 12.9716 BYRADIUS 10 km STOREDIST
ZRANGE demo:distance-scores 0 -1 WITHSCORES
\`\`\`

With \`STOREDIST\`, scores represent **distance from the search center**, rather than geospatial coordinate encoding.

**Important:** A destination created with \`STOREDIST\` should be treated as a distance-ranked Sorted Set, **not** as a reusable geospatial index for GEOSEARCH.

## 9. GEOHASH — compact location representation

\`\`\`redis
GEOHASH demo:places center point2
\`\`\`

Redis returns geohash-style strings representing encoded positions.

A geohash is useful for spatial indexing and grouping, but it is **not** a secret, an encryption format, or an exact representation of arbitrary decimal coordinates.

## 10. Older commands: GEORADIUS and GEORADIUSBYMEMBER

You may see older tutorials using:

\`\`\`redis
GEORADIUS demo:places 77.5946 12.9716 5 km
GEORADIUSBYMEMBER demo:places center 5 km
\`\`\`

These older commands have been **deprecated since Redis 6.2** in favor of \`GEOSEARCH\` and \`GEOSEARCHSTORE\`.

**Recommended:** Learn and use modern \`GEOSEARCH\` syntax for new applications, while recognizing legacy commands when maintaining existing systems.

## 11. A practical delivery-partner model

Suppose a delivery app tracks currently available couriers:

\`\`\`text
geo:drivers:available:city
  driver:101 -> lon, lat
  driver:102 -> lon, lat
  driver:103 -> lon, lat
\`\`\`

Add or update a driver:

\`\`\`redis
GEOADD geo:drivers:available:bengaluru 77.5946 12.9716 driver:101
GEOADD geo:drivers:available:bengaluru 77.6100 12.9800 driver:102
\`\`\`

Find drivers within 3 km:

\`\`\`redis
GEOSEARCH geo:drivers:available:bengaluru FROMLONLAT 77.6000 12.9750 BYRADIUS 3 km ASC WITHDIST COUNT 10
\`\`\`

\`\`\`mermaid
sequenceDiagram
    participant D as Driver app
    participant API as Backend API
    participant R as Redis
    participant U as Customer app
    D->>API: Update GPS coordinates
    API->>R: GEOADD driver location
    U->>API: Find nearby available drivers
    API->>R: GEOSEARCH within radius
    R-->>API: Candidate driver IDs + distances
    API-->>U: Filtered eligible drivers
\`\`\`

### What this does not solve

A location index alone does not tell you:
- Is the driver actually online now?
- Has the driver accepted another order?
- Is the driver authorized to serve this region?
- Is the location stale?
- Can the driver reach the customer using roads?
- What is the ETA?

You need presence/heartbeat data, availability state, assignment coordination, and potentially a routing service.

## 12. Stale location and TTL: an important trap

You might think:

\`\`\`redis
EXPIRE geo:drivers:available:bengaluru 60
\`\`\`

This expires the **entire location index**, not one driver's entry.

**Redis geospatial members do not have independent TTLs.**

A better design might combine:
- A GEO key containing candidate driver locations.
- Individual heartbeat keys with TTL.
- Periodic cleanup of stale members.
- Authorization and availability checks before assignment.

For example:

\`\`\`redis
SET driver:101:heartbeat alive EX 60
GEOADD geo:drivers:available:bengaluru 77.5946 12.9716 driver:101
\`\`\`

When querying, check each candidate's heartbeat and state. You must also remove stale drivers from the GEO key eventually; otherwise the index grows and includes outdated locations.

### Why two commands are not enough for perfect correctness

The heartbeat update and \`GEOADD\` are separate commands. Failures can leave inconsistent state. Production systems need explicit reconciliation and failure handling.

## 13. Node.js: nearby location lookup

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

redis.on("error", (error) => console.error("Redis:", error.message));

const key = "demo:geo:lesson16";

try {
  await redis.connect();

  await redis.del(key);

  await redis.geoAdd(key, [
    { longitude: 77.5946, latitude: 12.9716, member: "center" },
    { longitude: 77.6100, latitude: 12.9800, member: "point2" },
    { longitude: 77.6500, latitude: 13.0100, member: "point3" },
  ]);

  const nearby = await redis.geoSearchWith(
    key,
    { longitude: 77.5946, latitude: 12.9716 },
    { radius: 5, unit: "km" },
    ["WITHDIST", "WITHCOORD"],
    { SORT: "ASC", COUNT: 10 },
  );

  console.log("Nearby locations:", nearby);

  const distance = await redis.geoDist(key, "center", "point2", "km");
  console.log("Geographic distance (km):", distance);
} catch (error) {
  console.error("Geospatial lookup failed:", error.message);
  process.exitCode = 1;
} finally {
  if (redis.isOpen) await redis.quit();
}
\`\`\`

### Why this example is realistic

- Uses the official Node.js Redis client.
- Stores three sample coordinates.
- Searches by radius.
- Requests distances and coordinates.
- Demonstrates distance lookup separately.

**Production improvements:**
- Validate longitude/latitude types and ranges.
- Avoid accepting spoofed GPS data without considering trust and abuse.
- Authenticate requests and limit access to precise locations.
- Minimize retention of sensitive location history.
- Handle stale data and driver assignment races.
- Do not interpret geographic distance as road distance or ETA.

## 14. Geospatial vs Sorted Set: what is the relationship?

Redis geospatial data is represented in a Sorted Set, where each member has a score encoding its location.

That means:

\`\`\`redis
TYPE demo:places
ZCARD demo:places
ZREM demo:places point3
\`\`\`

These Sorted Set commands can operate on the geospatial key.

**But avoid directly changing GEO scores with arbitrary \`ZADD\` values.** That can corrupt the intended location encoding and make geospatial queries meaningless.

### Why geospatial encoding is useful

Redis can organize points by spatial proximity using encoded coordinate information, making radius and box searches practical without storing every point in a separate custom structure.

## 15. Redis GEO vs PostGIS / spatial databases

| Requirement | Redis GEO | PostGIS / spatial DB |
| --- | --- | --- |
| Fast nearby point lookup | Good fit | Good fit |
| Frequently updated transient locations | Good fit | Depends on workload |
| Complex polygons and spatial joins | Limited | Strong |
| Authoritative geospatial records | Requires durability design | Often a natural fit |
| Road routing and live traffic | Not provided | Usually separate routing service |
| Full GIS analysis | Limited | Stronger |
| In-memory caching of nearby candidates | Strong use case | Possible, different trade-offs |

**Rule of thumb:** Redis GEO is excellent for a fast proximity index. Use spatial databases or routing engines for complex geometry, durable records, and route calculations.

## 16. Complexity and performance

| Command | Typical complexity | Note |
| --- | --- | --- |
| \`GEOADD\` | O(log N) per added member | Sorted Set update |
| \`GEOPOS\` | O(M) for M members requested | Coordinate lookup |
| \`GEODIST\` | O(1) | Distance between two saved members |
| \`GEOHASH\` | O(M) for M members | Hash lookup |
| \`GEOSEARCH\` | Depends on search region and candidates | Not simply O(1) |
| \`GEOSEARCHSTORE\` | Search cost plus result storage | Creates/replaces destination |

Large search radii and dense urban regions can produce many candidate matches. Use practical bounds, \`COUNT\`, and monitoring. If you use \`ANY\`, understand the nearest-result trade-off.

## 17. Common mistakes

1. Passing latitude before longitude.
2. Treating Redis GEO as a separate top-level data type.
3. Assuming GEODIST is driving distance.
4. Assuming GEOSEARCH gives ETA or traffic-aware routes.
5. Using deprecated GEORADIUS commands in new code.
6. Assuming geospatial members have independent TTLs.
7. Trusting stale driver coordinates.
8. Assuming GEOSEARCH returns only currently available drivers.
9. Updating GEO scores with arbitrary \`ZADD\` values.
10. Treating \`STOREDIST\` results as a reusable coordinate index.
11. Assuming \`COUNT ANY\` guarantees the nearest N members.
12. Using Redis GEO for complex polygon or jurisdiction queries without additional geometry checks.
13. Exposing precise user location without authorization and privacy controls.
14. Forgetting Cluster restrictions for multi-key workflows.

## 18. Hands-on exercises

Use a disposable local Redis instance.

1. Add three sample locations using \`GEOADD\`.
2. Retrieve their coordinates using \`GEOPOS\`.
3. Measure distance between two members in kilometers.
4. Find members within 5 km of a coordinate.
5. Search around a stored member.
6. Return nearest first with \`ASC WITHDIST\`.
7. Limit results using \`COUNT\`.
8. Compare \`BYRADIUS\` and \`BYBOX\`.
9. Save nearby results with \`GEOSEARCHSTORE\`.
10. Try \`STOREDIST\` and inspect destination scores.
11. Update a member's location and repeat the search.
12. Explain why one \`EXPIRE\` cannot expire only a stale driver.
13. Build the Node.js nearby-places example.
14. Design heartbeat validation and cleanup for a courier app.

## 19. Interview questions

**Q1. What is Redis Geospatial?** A set of commands for indexing and querying longitude/latitude coordinates, implemented using a Sorted Set.

**Q2. GEOADD argument order?** Longitude first, latitude second.

**Q3. What does GEODIST return?** Approximate geographic distance between two stored members.

**Q4. GEOSEARCH vs GEORADIUS?** GEOSEARCH is the modern preferred command; GEORADIUS is deprecated.

**Q5. BYRADIUS vs BYBOX?** Circle-like radius query vs rectangular region query.

**Q6. Does Redis GEO calculate ETA?** No.

**Q7. Can each location member have a TTL?** Not directly; key-level TTL expires the whole GEO index.

**Q8. Why use WITHDIST?** To return distance alongside matching members.

**Q9. What is STOREDIST?** A GEOSEARCHSTORE option that stores distances as Sorted Set scores.

**Q10. What does COUNT ANY change?** It can permit early stopping without guaranteeing nearest N results.

**Q11. Redis GEO vs PostGIS?** Redis GEO is a fast point-proximity index; PostGIS supports much richer spatial operations.

**Q12. How do you handle stale drivers?** Combine location index with heartbeat/availability state and cleanup/reconciliation.

## 20. Revision checklist

- [ ] Geospatial uses Sorted Sets
- [ ] Longitude before latitude
- [ ] \`GEOADD\`, \`GEOPOS\`, \`GEODIST\`
- [ ] \`GEOSEARCH\` with radius and box
- [ ] \`ASC\`, \`DESC\`, \`COUNT\`, \`ANY\`
- [ ] \`WITHDIST\`, \`WITHCOORD\`, \`WITHHASH\`
- [ ] \`GEOSEARCHSTORE\` and \`STOREDIST\`
- [ ] \`GEOHASH\`
- [ ] Deprecated \`GEORADIUS\`
- [ ] Geographic vs road distance
- [ ] Stale-location and heartbeat design
- [ ] Node.js integration
- [ ] Redis GEO vs PostGIS

---

**Previous:** [Lesson 015 — Redis HyperLogLog](015-hyperloglog.md)  
**Next:** [Lesson 017 — Redis Streams Introduction](017-streams-intro.md) (planned)

**Official references:** [Redis Geospatial](https://redis.io/docs/latest/develop/data-types/geospatial/) · [GEOADD](https://redis.io/docs/latest/commands/geoadd/) · [GEOSEARCH](https://redis.io/docs/latest/commands/geosearch/) · [GEODIST](https://redis.io/docs/latest/commands/geodist/) · [GEOSEARCHSTORE](https://redis.io/docs/latest/commands/geosearchstore/)
