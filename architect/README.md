# Software Architect Interview Questions & Answers

A curated list of **senior / staff software architect interview questions** — medium to hard difficulty, the kind asked at top product-based companies. Covers system design trade-offs, distributed systems, scalability, resilience, data architecture, and architectural decision-making. Answers are kept concise and focused on what interviewers are actually probing for.

---

## System Design Fundamentals

### 1. **How do you approach a system design interview from the architect's chair (not the engineer's)?**

**Answer:** Start with **requirements and constraints**, not components: functional scope, scale (RPS, data volume, growth), latency/availability SLAs, consistency needs, and budget/team constraints. Then move top-down: API contract → data model → high-level components → bottleneck analysis → trade-offs. An architect is judged on **the questions asked before designing**, not the boxes drawn.

> **Interview framing:** Interviewers want to see you resist jumping to a solution — clarifying scale and SLAs first signals seniority.

---

### 2. **What's the difference between horizontal and vertical scaling, and when do you choose each?**

**Answer:**

| Aspect | Vertical (scale-up) | Horizontal (scale-out) |
|---|---|---|
| Method | Bigger machine (CPU/RAM) | More machines |
| Ceiling | Hardware limit | Near-limitless |
| Complexity | Low (no code change) | High (distributed state, coordination) |
| Downtime | Often requires restart | Rolling, zero-downtime |
| Cost curve | Exponential at high end | Linear, but adds ops overhead |

Choose vertical for quick wins on stateful systems (databases) below the hardware ceiling; horizontal when you need fault tolerance and elasticity, and are willing to pay the complexity cost (sharding, distributed consensus, statelessness).

---

### 3. **Explain the CAP theorem and how it actually applies to real system design.**

**Answer:** In a network partition (P, which *will* happen), you must choose between **Consistency** (every read gets the latest write) and **Availability** (every request gets a response). CAP is not a 3-way pick — partition tolerance is mandatory in any distributed system; the real trade-off is **C vs A during a partition**.

In practice, most systems are **PACELC**: even without a partition, you trade **Latency vs Consistency**. DynamoDB/Cassandra lean AP (tunable); Spanner/etcd/ZooKeeper lean CP.

> **Interview framing:** Say "CAP forces a choice only during partitions; PACELC is the everyday trade-off" — this shows you know the theorem is often misapplied.

---

### 4. **What is the difference between strong, eventual, and causal consistency?**

**Answer:**

- **Strong consistency** — every read sees the most recent write, globally ordered. Expensive (requires consensus/quorum), used for financial ledgers.
- **Eventual consistency** — replicas converge *eventually* if writes stop. Cheap, high availability; used for social feeds, DNS, caches.
- **Causal consistency** — operations that are causally related (a reply to a comment) are seen in order by everyone; unrelated operations can be reordered. A practical middle ground (e.g., session consistency in Cosmos DB).

Pick based on the cost of a stale read for that specific data — inventory count needs strong, "like count" tolerates eventual.

---

## Scalability & Performance

### 5. **How do you design a system to handle a 100x traffic spike (e.g., flash sale)?**

**Answer:** Layer defenses instead of relying on one:

1. **CDN/edge caching** for static and cacheable dynamic content.
2. **Queue-based load leveling** — accept requests into a queue (Kafka/SQS), process asynchronously, decouple ingestion from processing rate.
3. **Rate limiting / admission control** at the edge to protect downstream.
4. **Auto-scaling** with pre-warmed capacity (auto-scaling reacts too slowly for a sudden 100x spike — pre-warm based on known event timing).
5. **Graceful degradation** — shed non-critical features (recommendations, analytics) to protect the critical path (checkout).
6. **Read replicas / caching** to keep the primary DB off the hot path.

> **Interview framing:** Mentioning graceful degradation and queue-based leveling (not just "add more servers") is what separates architect-level answers.

---

### 6. **What is the difference between load balancing algorithms (round robin, least connections, consistent hashing)?**

**Answer:**

- **Round robin** — simple rotation; ignores server load, fine for homogeneous stateless services.
- **Least connections** — routes to the server with fewest active connections; better for variable request durations.
- **Consistent hashing** — maps requests to servers by hash so the same key (user/session) lands on the same server, and adding/removing a node only remaps `1/N` of keys instead of all of them. Critical for caching layers and sharded systems to minimize cache-miss storms on scale events.

```
hash(key) → ring position → nearest server clockwise
Adding server D only steals keys between C and D, not a full remap
```

---

### 7. **How does consistent hashing prevent cache stampede on node changes?**

**Answer:** Without it, adding/removing a cache node re-hashes `key % N`, invalidating almost all keys at once (`N → N+1` changes nearly every mapping) — causing a stampede to the origin. Consistent hashing places nodes and keys on a hash ring; a node change only remaps the keys between it and its neighbor (~`1/N` of total keys). Virtual nodes (multiple ring points per physical node) further smooth load distribution and rebalancing.

---

### 8. **What's the difference between caching strategies: cache-aside, write-through, write-behind?**

**Answer:**

| Strategy | Write path | Read path | Risk |
|---|---|---|---|
| **Cache-aside** | App writes to DB, invalidates cache | App checks cache, falls back to DB on miss | Brief staleness window |
| **Write-through** | App writes to cache, cache syncs to DB | Cache always has data | Write latency = cache + DB |
| **Write-behind** | App writes to cache, async flush to DB | Cache always has data | Data loss risk if cache crashes before flush |

Cache-aside is the most common default (Redis + app logic); write-behind trades durability for write throughput (used in high-write analytics pipelines).

---

### 9. **How do you design for idempotency in distributed systems?**

**Answer:** Any operation that can be retried (network timeout, at-least-once delivery) must be safe to apply multiple times.

- **Idempotency keys** — client generates a unique key per logical operation; server stores the key → result mapping and returns the cached result on retry.
- **Natural idempotency** — design operations as "set to X" rather than "increment by X" where possible.
- **Conditional writes** — use `UPDATE ... WHERE version = N` or DynamoDB conditional expressions to make writes no-ops if already applied.

```ts
// Idempotency key pattern for a payment API
async function chargeCard(idempotencyKey: string, amount: number) {
  const existing = await db.get("idempotency_keys", idempotencyKey);
  if (existing) return existing.result; // safe replay
  const result = await paymentGateway.charge(amount);
  await db.put("idempotency_keys", idempotencyKey, { result });
  return result;
}
```

---

### 10. **Explain the difference between throughput and latency, and why optimizing one can hurt the other.**

**Answer:** **Latency** is the time for one request to complete; **throughput** is requests processed per unit time. Batching improves throughput (fewer round trips, better amortized overhead) but increases latency per item (waiting to fill a batch). Connection pooling, request coalescing, and async I/O are throughput optimizations that can add queuing latency under load. Architects must pick the target metric per use case — batch ETL optimizes throughput; user-facing APIs optimize p99 latency.

---

## Distributed Systems

### 11. **What is the difference between synchronous and asynchronous inter-service communication, and how do you decide?**

**Answer:**

- **Synchronous (REST/gRPC)** — simple mental model, immediate consistency, but couples availability (caller fails if callee is down) and creates cascading latency.
- **Asynchronous (message queue/event bus)** — decouples services in time and availability, enables buffering and retries, but adds eventual consistency and operational complexity (dead-letter handling, ordering).

Rule of thumb: use sync for request/response where the caller needs an immediate answer (auth check); use async for anything that can be eventually processed (order fulfillment, notifications) or crosses a failure-domain boundary.

---

### 12. **How do you prevent cascading failures between microservices?**

**Answer:**

- **Circuit breaker** — stop calling a failing downstream service after an error threshold; fail fast instead of piling up threads/timeouts.
- **Bulkheads** — isolate resource pools (thread pools, connection pools) per dependency so one slow service can't starve others.
- **Timeouts** everywhere — never call downstream without a bounded timeout.
- **Retry with backoff + jitter** — avoid retry storms synchronizing across clients.
- **Load shedding** — reject excess requests at the edge rather than degrading everyone.

```
Circuit breaker states:
CLOSED (normal) --failures > threshold--> OPEN (fail fast)
OPEN --after cooldown--> HALF_OPEN (test with limited traffic)
HALF_OPEN --success--> CLOSED   |   HALF_OPEN --failure--> OPEN
```

---

### 13. **What is the Saga pattern and when would you use it over distributed transactions (2PC)?**

**Answer:** 2PC (two-phase commit) requires a coordinator to lock resources across services until all participants ack — this blocks under partition and doesn't scale across service/team boundaries. **Saga** breaks a transaction into a sequence of local transactions, each with a **compensating action** to undo it on failure.

- **Choreography** — services react to each other's events (no central coordinator); simple but hard to trace at scale.
- **Orchestration** — a central saga orchestrator calls each step and triggers compensations; easier to reason about and monitor, adds a coordination component.

Use Saga for cross-service business transactions (order → payment → inventory) where eventual consistency is acceptable; use 2PC only within a tightly coupled, low-latency boundary (rare in modern architectures).

---

### 14. **How does leader election work in distributed systems (e.g., Raft/ZooKeeper)?**

**Answer:** Nodes agree on a single leader to serialize writes/decisions, avoiding split-brain. Raft: nodes start as followers, a randomized election timeout triggers a candidate to request votes; majority quorum wins leadership for a term. If the leader fails (missed heartbeats), a new election starts. Quorum-based majority (`N/2 + 1`) guarantees at most one leader per term even under partition, at the cost of requiring a majority of nodes to be reachable to make progress.

---

### 15. **What is the outbox pattern and what problem does it solve?**

**Answer:** Solves the **dual-write problem** — writing to a DB and publishing an event to a message broker are not atomic; a crash between the two causes inconsistency. The outbox pattern writes the event to an **outbox table in the same DB transaction** as the business change, then a separate relay process (or CDC like Debezium) tails the outbox and publishes to the broker, marking rows as sent.

```sql
BEGIN;
UPDATE orders SET status = 'CONFIRMED' WHERE id = 123;
INSERT INTO outbox (event_type, payload) VALUES ('OrderConfirmed', '{"orderId":123}');
COMMIT;
-- separate relay process polls/streams outbox and publishes to Kafka
```

---

### 16. **Explain exactly-once processing — is it really achievable?**

**Answer:** True exactly-once *delivery* over an unreliable network is impossible (you can't distinguish a lost ack from a lost message). What's achievable is **exactly-once processing effect** via idempotent consumers + at-least-once delivery: dedupe by message ID, use idempotent writes, or transactional offset commits (Kafka's transactional producer commits the output write and the offset atomically). Frame it as "effectively-once," not literal exactly-once.

---

## Data Architecture

### 17. **How do you decide between SQL and NoSQL for a new service?**

**Answer:** Start from access patterns, not the label:

- Need **flexible/ad-hoc queries, joins, strong transactional integrity** → relational (Postgres/MySQL).
- **Known, fixed access patterns at massive scale**, need predictable low-latency reads/writes → key-value/wide-column (DynamoDB/Cassandra) — but you must design the schema around queries upfront.
- **Deeply nested, evolving document structure**, moderate scale → document store (MongoDB).
- **Graph traversal-heavy** (social, recommendations) → graph DB (Neo4j).

> **Interview framing:** The strongest answer rejects "NoSQL for scale, SQL for structure" as a myth — modern Postgres scales very far, and DynamoDB requires upfront query modeling, not less design.

---

### 18. **What is database sharding, and what are the common sharding strategies?**

**Answer:** Sharding splits data horizontally across multiple database instances to scale beyond a single node's capacity.

- **Range-based** — shard by key ranges (e.g., user_id 1–1M). Simple, but risks hot shards for sequential/skewed keys.
- **Hash-based** — hash the shard key to distribute evenly. Even distribution, but range queries become scatter-gather.
- **Directory-based** — a lookup service maps keys to shards. Flexible rebalancing, but the directory is a new single point of failure/bottleneck.

Key challenges after sharding: cross-shard joins, cross-shard transactions, and rebalancing when a shard outgrows capacity (mitigated by consistent hashing).

---

### 19. **How do you handle schema evolution in event-driven systems without breaking consumers?**

**Answer:** Treat events like a public API contract:

- **Backward-compatible changes only on a given schema version**: add optional fields, never remove/rename/retype fields in place.
- Use a **schema registry** (Confluent Schema Registry, AWS Glue Schema Registry) with compatibility checks enforced at publish time (`BACKWARD`/`FORWARD`/`FULL` compatibility modes).
- For breaking changes, **version the event type** (`OrderCreatedV2`) and run both versions in parallel until consumers migrate.

---

### 20. **What is CQRS and when is it worth the added complexity?**

**Answer:** **Command Query Responsibility Segregation** splits the write model (commands, normalized, optimized for consistency) from the read model (queries, denormalized, optimized for the specific read pattern), often backed by different stores kept in sync via events.

Worth it when: read and write workloads have very different scaling/shape needs (e.g., high-write IoT ingestion vs. complex read dashboards), or when you need multiple read representations of the same data (search index + cache + analytics store). Not worth it for simple CRUD — it adds eventual consistency between models and real operational overhead.

---

### 21. **What is Event Sourcing, and what are its trade-offs vs. storing current state?**

**Answer:** Instead of storing current state, store the **immutable sequence of events** that led to it; current state is derived by replaying events (or from a periodic snapshot + replay of the tail).

- **Pros:** full audit trail, temporal queries ("state as of last Tuesday"), natural fit with event-driven architectures, enables rebuilding new read models retroactively.
- **Cons:** query complexity (must project state), storage growth (mitigated by snapshotting), schema evolution of old events, and a steep learning curve for the team.

Use for domains where audit/history is a first-class requirement (finance, order lifecycle); avoid for simple entities with no audit need.

---

### 22. **How would you design a globally distributed, multi-region database strategy?**

**Answer:** Trade-off is again consistency vs. latency:

- **Active-passive** — one region is primary (writes), others are read replicas with async replication; simplest, but failover has RPO/RTO risk and non-primary regions serve stale reads.
- **Active-active** — multiple regions accept writes; needs conflict resolution (last-write-wins, CRDTs, or app-level merge) or a globally consistent store (Spanner, CockroachDB) using synchronized clocks/consensus — at the cost of write latency for cross-region consensus.
- **Data locality/partitioning by region** — shard by geography (EU data stays in EU) when regulatory (GDPR) constraints exist, avoiding the conflict problem entirely for most rows.

Choose based on RPO/RTO requirements and whether the business tolerates conflict resolution complexity.

---

## Resilience & Operations

### 23. **How do you design a system's SLA/SLO/error budget strategy?**

**Answer:** **SLI** (indicator, e.g., p99 latency), **SLO** (internal target, e.g., 99.9% of requests < 200ms), **SLA** (external contractual commitment, usually looser than the SLO with penalties). The gap between 100% and the SLO is the **error budget** — it quantifies how much unreliability is acceptable and gives teams permission to ship risk (deploys, experiments) as long as the budget isn't exhausted. When the budget is burned, feature work pauses for reliability work. This turns "reliability" from a vague goal into a measurable, negotiable resource.

---

### 24. **What is the difference between RPO and RTO, and how do they drive DR architecture?**

**Answer:** **RPO (Recovery Point Objective)** — max acceptable data loss, measured in time (e.g., 5 min → need replication/backup at least every 5 min). **RTO (Recovery Time Objective)** — max acceptable downtime to restore service. Low RPO/RTO (near-zero) demands active-active multi-region with synchronous or near-real-time replication (expensive); higher tolerance allows cheaper backup-and-restore or pilot-light DR. Always design DR to the *business-stated* RPO/RTO, not a default "as available as possible."

---

### 25. **How do you approach capacity planning for a new system?**

**Answer:** Work backward from expected load: estimate peak RPS (not average — design for peak × safety margin), data growth rate, and per-request resource cost (CPU, memory, I/O). Model bottlenecks per tier (DB connections, cache hit ratio, network bandwidth) and identify the first component to saturate. Add headroom (typically 30–50%) for traffic spikes and load-test to validate the model rather than trusting theoretical math alone — real systems rarely bottleneck where you predict.

---

### 26. **What is backpressure and how do you implement it in a streaming/queue-based system?**

**Answer:** Backpressure signals a slow consumer's limits back to the producer so the producer slows down instead of overwhelming the consumer (unbounded queue growth → OOM/latency blowup).

- **Pull-based systems** (Kafka) — consumers pull at their own pace; natural backpressure.
- **Push-based systems** — need explicit signaling: bounded queues that block/reject on full, reactive streams (`Flow`/`Reactor`) with demand signaling, or HTTP 429 + client-side rate limiting.

Without backpressure, a slow downstream service causes unbounded memory growth upstream — a common root cause of cascading outages.

---

### 27. **How do you design zero-downtime deployments for a stateful, schema-driven service?**

**Answer:** The hard part isn't the app — it's the schema. Use the **expand-contract (parallel change) pattern**:

1. **Expand** — add new column/table, deploy code that writes to both old and new (dual write), backfill historical data.
2. **Migrate** — deploy code that reads from the new schema, still writing both.
3. **Contract** — once verified, deploy code that only uses the new schema, then drop the old column.

Combine with **blue-green or canary deployment** at the app layer, feature flags to decouple deploy from release, and always design migrations to be backward-compatible with the *currently running* previous version (since rolling deploys run old and new code simultaneously).

---

## Architectural Decision-Making

### 28. **How do you evaluate build vs. buy for a core piece of infrastructure?**

**Answer:** Frame it around: (1) **is this a differentiator** for the business, or undifferentiated heavy lifting — build only what creates competitive advantage; (2) **total cost of ownership** — not just license/service cost, but the ongoing engineering time to build, operate, patch, and scale a homegrown solution; (3) **time-to-market** pressure; (4) **lock-in risk and exit cost** of the vendor option. Default bias for most teams: buy/use managed services for commodity infra (queues, DBs, auth), build only the core domain logic that is the actual product.

---

### 29. **How do you communicate and document an architectural decision so it survives team turnover?**

**Answer:** Use **Architecture Decision Records (ADRs)** — short, immutable documents capturing: context (the problem/forces), the decision, alternatives considered and why rejected, and consequences (trade-offs accepted). Stored in-repo, versioned with the code. This prevents the classic failure mode where a past constraint (now resolved) still shapes design because nobody remembers *why* a decision was made — the ADR makes the reasoning, not just the outcome, durable.

```markdown
# ADR-012: Use event-driven architecture for order fulfillment
## Status: Accepted
## Context: Order processing needs to scale independently of inventory/payment...
## Decision: Adopt Kafka-based choreography between services...
## Alternatives considered: Synchronous REST orchestration (rejected: tight coupling)...
## Consequences: Eventual consistency in order status; requires DLQ monitoring...
```

---

### 30. **How do you balance technical debt against feature velocity as an architect?**

**Answer:** Not all debt is equal — classify it: **deliberate & prudent** (a known shortcut taken to hit a deadline, tracked to be repaid) vs. **inadvertent & reckless** (design flaws from lack of foresight). Make debt **visible** (tech-debt backlog with business impact, not just "code smells"), tie repayment to when it starts costing measurable velocity (slower delivery, rising incident rate) rather than repaying on a fixed schedule, and negotiate a standing allocation (e.g., 20% of capacity) rather than one-off asks — this turns it into a continuous, defensible trade-off instead of a recurring argument.

> **Interview framing:** Interviewers listen for whether you treat debt as a business trade-off with cost/impact framing, not just an engineering purity concern.
