# Redis — Beginner to Advanced & Production

A practical Redis course in **simple English** with diagrams, commands, runnable Node.js examples, production trade-offs, exercises and interview questions.

> **Status:** Curriculum only. Lesson links below point to planned Markdown files and become active as we publish lessons one by one. No lesson content has been added yet.

## Section 01 — Redis Fundamentals
- [Lesson 001 — What Is Redis?](01-fundamentals/001-what-is-redis.md)
- [Lesson 002 — Why Redis? Use Cases and Trade-offs](01-fundamentals/002-why-redis.md)
- [Lesson 003 — Redis vs Traditional Databases](01-fundamentals/003-redis-vs-databases.md)
- [Lesson 004 — Redis Architecture and Event Loop](01-fundamentals/004-architecture-event-loop.md)
- [Lesson 005 — Redis Installation and Setup](01-fundamentals/005-installation.md)
- [Lesson 006 — Redis CLI and First Commands](01-fundamentals/006-cli.md)
- [Lesson 007 — Keys, Namespaces and Expiration](01-fundamentals/007-keys-expiration.md)
- [Lesson 008 — Redis Configuration and INFO](01-fundamentals/008-configuration-info.md)

## Section 02 — Data Types and Commands
- [Lesson 009 — Strings](02-data-types/009-strings.md)
- [Lesson 010 — Hashes](02-data-types/010-hashes.md)
- [Lesson 011 — Lists](02-data-types/011-lists.md)
- [Lesson 012 — Sets](02-data-types/012-sets.md)
- [Lesson 013 — Sorted Sets](02-data-types/013-sorted-sets.md)
- [Lesson 014 — Bitmaps and Bitfields](02-data-types/014-bitmaps-bitfields.md)
- [Lesson 015 — HyperLogLog](02-data-types/015-hyperloglog.md)
- [Lesson 016 — Geospatial Indexes](02-data-types/016-geospatial.md)
- [Lesson 017 — Streams Introduction](02-data-types/017-streams-intro.md)
- [Lesson 018 — Choosing the Right Data Type](02-data-types/018-data-type-selection.md)

## Section 03 — Keys, Memory and Internals
- [Lesson 019 — Key Design and Naming](03-internals/019-key-design.md)
- [Lesson 020 — TTL, Expiration and Eviction](03-internals/020-ttl-eviction.md)
- [Lesson 021 — Memory Layout and Encodings](03-internals/021-memory-encodings.md)
- [Lesson 022 — Persistence: RDB](03-internals/022-rdb.md)
- [Lesson 023 — Persistence: AOF](03-internals/023-aof.md)
- [Lesson 024 — RDB vs AOF and Recovery](03-internals/024-recovery.md)
- [Lesson 025 — Pipelining and Round Trips](03-internals/025-pipelining.md)
- [Lesson 026 — Transactions and WATCH](03-internals/026-transactions.md)
- [Lesson 027 — Lua Scripting and Redis Functions](03-internals/027-lua-functions.md)

## Section 04 — Caching in Real Applications
- [Lesson 028 — Caching Fundamentals](04-caching/028-caching-fundamentals.md)
- [Lesson 029 — Cache-Aside Pattern](04-caching/029-cache-aside.md)
- [Lesson 030 — Read-Through and Write-Through](04-caching/030-read-write-through.md)
- [Lesson 031 — Write-Behind and Write-Around](04-caching/031-write-behind-around.md)
- [Lesson 032 — Cache Invalidation Strategies](04-caching/032-invalidation.md)
- [Lesson 033 — Cache Stampede and Hot Keys](04-caching/033-stampede-hot-keys.md)
- [Lesson 034 — Cache Penetration and Avalanche](04-caching/034-penetration-avalanche.md)
- [Lesson 035 — TTL, Jitter and Stale Data](04-caching/035-ttl-jitter-stale.md)
- [Lesson 036 — Cache Consistency and Failure Modes](04-caching/036-consistency.md)

## Section 05 — Redis with Node.js and Express
- [Lesson 037 — Node.js Redis Client Setup](05-nodejs/037-client-setup.md)
- [Lesson 038 — Connections, Errors and Reconnection](05-nodejs/038-connections.md)
- [Lesson 039 — Cache REST API Responses](05-nodejs/039-api-caching.md)
- [Lesson 040 — Redis with MongoDB and PostgreSQL](05-nodejs/040-database-caching.md)
- [Lesson 041 — Session Storage and Authentication](05-nodejs/041-sessions.md)
- [Lesson 042 — Rate Limiting Algorithms](05-nodejs/042-rate-limiting.md)
- [Lesson 043 — Counters, Idempotency and Deduplication](05-nodejs/043-counters-idempotency.md)
- [Lesson 044 — Distributed Locks and Their Limits](05-nodejs/044-distributed-locks.md)
- [Lesson 045 — Testing Redis Integrations](05-nodejs/045-testing.md)

## Section 06 — Messaging, Queues and Real Time
- [Lesson 046 — Redis Pub/Sub](06-messaging/046-pubsub.md)
- [Lesson 047 — Pub/Sub Delivery Guarantees](06-messaging/047-pubsub-guarantees.md)
- [Lesson 048 — Streams and Consumer Groups](06-messaging/048-streams-consumers.md)
- [Lesson 049 — Acknowledgment, Retry and Recovery](06-messaging/049-ack-retry.md)
- [Lesson 050 — Redis-backed Job Queues](06-messaging/050-job-queues.md)
- [Lesson 051 — BullMQ with Node.js](06-messaging/051-bullmq.md)
- [Lesson 052 — WebSockets and Socket.IO Scaling](06-messaging/052-socketio-scaling.md)
- [Lesson 053 — Choosing Pub/Sub, Streams or Queues](06-messaging/053-messaging-choices.md)

## Section 07 — Scaling and High Availability
- [Lesson 054 — Replication](07-scaling/054-replication.md)
- [Lesson 055 — Replication Lag and Data Safety](07-scaling/055-replication-lag.md)
- [Lesson 056 — Redis Sentinel](07-scaling/056-sentinel.md)
- [Lesson 057 — Redis Cluster Fundamentals](07-scaling/057-cluster.md)
- [Lesson 058 — Hash Slots and Resharding](07-scaling/058-hash-slots.md)
- [Lesson 059 — Cluster Failover and Limitations](07-scaling/059-failover.md)
- [Lesson 060 — Vertical vs Horizontal Scaling](07-scaling/060-scaling.md)

## Section 08 — Security, Observability and Production
- [Lesson 061 — Authentication, ACLs and TLS](08-production/061-security.md)
- [Lesson 062 — Secure Network Deployment](08-production/062-network-security.md)
- [Lesson 063 — Monitoring and Metrics](08-production/063-monitoring.md)
- [Lesson 064 — SLOWLOG, Latency and Troubleshooting](08-production/064-latency.md)
- [Lesson 065 — Memory Optimization and Big Keys](08-production/065-memory-optimization.md)
- [Lesson 066 — Backup, Restore and Disaster Recovery](08-production/066-backup-restore.md)
- [Lesson 067 — Docker and Docker Compose](08-production/067-docker.md)
- [Lesson 068 — Managed Redis and Deployment Choices](08-production/068-managed-redis.md)
- [Lesson 069 — Production Checklist and Anti-patterns](08-production/069-production-checklist.md)

## Section 09 — Advanced Redis Features
- [Lesson 070 — Redis JSON and Search Overview](09-advanced/070-json-search.md)
- [Lesson 071 — Probabilistic Data Structures](09-advanced/071-probabilistic.md)
- [Lesson 072 — Leaderboards and Sliding Windows](09-advanced/072-leaderboards-windows.md)
- [Lesson 073 — Distributed Systems Trade-offs](09-advanced/073-distributed-tradeoffs.md)
- [Lesson 074 — Benchmarking and Load Testing](09-advanced/074-benchmarking.md)
- [Lesson 075 — Redis Version and Feature Compatibility](09-advanced/075-version-compatibility.md)

## Section 10 — Projects and Interview Preparation
- [Lesson 076 — Build an API Cache](10-projects/076-api-cache-project.md)
- [Lesson 077 — Build a Rate Limiter](10-projects/077-rate-limiter-project.md)
- [Lesson 078 — Build a Real-Time Leaderboard](10-projects/078-leaderboard-project.md)
- [Lesson 079 — Build a Background Job System](10-projects/079-jobs-project.md)
- [Lesson 080 — Scale a Chat Application](10-projects/080-chat-project.md)
- [Lesson 081 — Redis System Design Case Studies](10-projects/081-system-design.md)
- [Lesson 082 — Redis Interview Questions](10-projects/082-interview-questions.md)
- [Lesson 083 — Revision Checklist and Next Steps](10-projects/083-revision.md)

---

## Teaching Standards
Every lesson will cover **what, why, how, when, trade-offs**, detailed but simple conceptual explanations, realistic examples from Node.js, Express, MongoDB, PostgreSQL, e-commerce and chat systems, diagrams where helpful, runnable code, mistakes to avoid, exercises, and interview questions.

We will distinguish Redis core features from optional modules/distributions, and call out version differences. **Lessons are only published when requested.**
