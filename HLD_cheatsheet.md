# HLD (High-Level Design) Cheatsheet

Quick reference for system design interviews. Source: systemdesignschool.io — this is a condensed companion, not a replacement.

---

## Interview Approach (follow this order every time)

1. **Clarify requirements** — functional (what must it do) and non-functional (scale, latency, consistency needs). Ask about read/write ratio, expected users/QPS, data size.
2. **Back-of-envelope estimation** — QPS, storage, bandwidth. Order-of-magnitude only, don't over-precision.
3. **High-level design** — draw the major components (client, load balancer, API layer, cache, DB, queue). Get the shape right before details.
4. **Deep dive** — pick 1-2 components the interviewer cares about (usually the hardest part of the problem) and go deep.
5. **Identify bottlenecks & tradeoffs** — single points of failure, scaling limits; state tradeoffs explicitly ("I'm choosing eventual consistency here because availability matters more for this use case").

---

## Back-of-Envelope Numbers (memorize these)

| Operation | Latency |
|---|---|
| L1 cache reference | ~1 ns |
| Main memory reference | ~100 ns |
| SSD random read | ~100 μs |
| Round trip within same datacenter | ~500 μs |
| Disk seek | ~10 ms |
| Round trip between continents (e.g. US↔Europe) | ~150 ms |

| Scale reference | Value |
|---|---|
| 1 day | ~86,400 seconds |
| 1 million requests/day | ~12 QPS average |
| 1 KB | 1,000 bytes (approx) |
| 1 million users, 1 KB/user | 1 GB |

Rule of thumb: if a number needs more than 1 significant figure of precision in an interview, you're overengineering the estimate.

---

## Scalability

- **Vertical scaling**: bigger machine. Simple, but has a ceiling and single point of failure.
- **Horizontal scaling**: more machines. Requires statelessness — no server-local session state; push session data to a shared store (Redis, DB).
- **Load balancer algorithms**:
  - Round robin — simple, even distribution, ignores server load
  - Least connections — routes to server with fewest active connections
  - IP hash — same client always hits same server (useful for sticky sessions)
  - Consistent hashing — minimizes redistribution when servers are added/removed (see below)

---

## Caching

| Pattern | How it works | Tradeoff |
|---|---|---|
| Cache-aside (lazy loading) | App checks cache first; on miss, reads DB and populates cache | Simple, but first request after eviction is slow; cache can go stale |
| Write-through | Write goes to cache and DB synchronously | Cache always consistent with DB; write latency higher |
| Write-back (write-behind) | Write goes to cache, DB updated asynchronously | Fast writes; risk of data loss if cache fails before flush |
| Write-around | Write goes directly to DB, bypassing cache | Avoids cache pollution from write-heavy, rarely-read data |

**Eviction policies**: LRU (evict least recently used — good general default), LFU (evict least frequently used — good when access frequency is more predictive than recency), FIFO (simplest, rarely optimal).

**CDN**: caches static content geographically close to users. Push CDN (you upload) vs Pull CDN (CDN fetches on first request, then caches).

---

## Databases

- **SQL vs NoSQL**: SQL for strong consistency, complex queries/joins, well-defined schema. NoSQL for horizontal scale, flexible schema, high write throughput.
- **NoSQL types**: key-value (Redis, DynamoDB), document (MongoDB), column-family (Cassandra), graph (Neo4j).
- **Indexing**: speeds reads, slows writes (index must be updated). B-tree indexes for range queries, hash indexes for exact-match lookups.
- **Replication**:
  - Leader-follower: one write node, multiple read replicas. Simple, but leader is a bottleneck/SPOF until failover.
  - Multi-leader: writes accepted at multiple nodes; conflict resolution needed.
- **Sharding strategies**:
  - Range-based — simple, but risk of hotspots (e.g. all recent data in one shard)
  - Hash-based — even distribution, but range queries become expensive (must query all shards)
  - Directory-based — a lookup service maps keys to shards; flexible but adds a dependency

---

## Consistency & Availability

- **CAP theorem**: under a network partition, you must choose Consistency or Availability (Partition tolerance is not optional in a distributed system). In practice this is a spectrum, not a binary choice.
- **Strong consistency**: every read sees the latest write. Higher latency, harder to scale.
- **Eventual consistency**: reads may return stale data temporarily, but will converge. Higher availability, lower latency.
- **Quorum reads/writes**: with N replicas, W = write quorum, R = read quorum. If W + R > N, you get strong consistency. Tune W/R to trade off latency vs consistency.

---

## Message Queues

- **Kafka**: log-based, high throughput, consumers track their own offset, good for event streaming and replay.
- **RabbitMQ**: traditional message broker, supports complex routing (exchanges), good for task queues.
- **Pub/sub vs point-to-point**: pub/sub = multiple consumers all get the message; point-to-point = one consumer per message (competing consumers).
- **Delivery guarantees**: at-most-once (may lose messages, never duplicates), at-least-once (never loses, may duplicate — requires idempotent consumers), exactly-once (hardest, usually built on at-least-once + deduplication).
- **Backpressure**: when consumers can't keep up with producers — handle via bounded queues, rate limiting producers, or autoscaling consumers.

---

## Rate Limiting

| Algorithm | How it works | Tradeoff |
|---|---|---|
| Token bucket | Bucket refills at fixed rate; each request consumes a token | Allows bursts up to bucket size; simple to implement |
| Leaky bucket | Requests processed at fixed rate regardless of arrival rate | Smooths bursts completely; can add latency |
| Fixed window counter | Count requests per fixed time window | Simple, but allows 2x burst at window boundary |
| Sliding window log | Store timestamp of every request, count within rolling window | Accurate, but memory-heavy at scale |
| Sliding window counter | Weighted average of current + previous window | Good balance of accuracy and memory — most commonly used in practice |

---

## Consistent Hashing

Used to minimize data redistribution when nodes are added/removed. Nodes and keys are hashed onto a ring; a key belongs to the next node clockwise. Adding/removing a node only affects its immediate neighbors, not the whole dataset. Virtual nodes (multiple points per physical node) improve load distribution.

---

## Classic Design Problems — Quick Approach

- **URL shortener**: hash-based ID generation (base62 encoding of an auto-increment ID, or a hash + collision check) → key-value store → redirect via 301/302. Discuss custom aliases, expiration, analytics.
- **Rate limiter**: pick an algorithm (see above), discuss distributed rate limiting (shared state in Redis vs per-node approximate limiting).
- **Chat system**: WebSocket for real-time delivery, message queue for offline users, separate services for presence/typing indicators, message storage sharded by conversation ID.
- **News feed**: push model (fan-out on write — precompute feeds, fast reads, expensive for high-follower accounts) vs pull model (fan-out on read — cheap writes, slower reads) vs hybrid (push for most users, pull for celebrity accounts).
- **Ride-sharing matching**: geospatial indexing (geohash or quad-tree) to find nearby drivers, matching algorithm balancing proximity/wait-time/driver rating.
- **Job scheduler**: priority queue + worker pool, handle retries/dead-letter queues, idempotent job execution.

---

## Common Pitfalls

- Jumping to a detailed design before clarifying requirements.
- Not stating tradeoffs explicitly — every design decision should come with a "because."
- Over-indexing on one component and running out of time for the rest.
- Ignoring failure modes (what happens when the cache goes down? the DB leader fails?).
- Forgetting to revisit the numbers from step 2 when discussing scaling.
