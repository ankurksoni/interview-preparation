# System Design & Distributed Systems – Term Definitions

Brief explanations with simple examples for common system design interview terms.

---

## API & Distributed Systems

- **Idempotency** – Calling an operation multiple times has the same effect as calling it once.
  *Example: `PUT /user/1 {name: "Ankur"}` — sending it 5 times still results in the same final state.*
- **Retry** – Re-attempting a failed request, hoping it succeeds on a later try.
  *Example: A payment API call times out, so the client retries after 1 second.*
- **Timeout** – A limit on how long you wait for a response before giving up.
  *Example: An HTTP client aborts a request if no response arrives within 5 seconds.*
- **Exponential Backoff** – Increasing the wait time between retries exponentially to avoid overwhelming a struggling system.
  *Example: Retry after 1s, then 2s, then 4s, then 8s.*
- **Jitter** – Adding randomness to retry delays so many clients don't retry at the exact same moment.
  *Example: Instead of retrying at exactly 4s, retry at 4s ± random(0-1s).*
- **Rate Limiting** – Restricting how many requests a client can make in a time window.
  *Example: "100 requests per minute per API key."*
- **Throttling** – Slowing down or delaying requests once a limit is approached (often paired with rate limiting).
  *Example: Server responds slower or queues requests once 90% of the rate limit is used.*
- **Circuit Breaker** – Stops calling a failing downstream service for a while, to prevent cascading failures.
  *Example: After 5 failed calls to a service, stop calling it for 30 seconds and return a fallback response.*
- **Bulkhead** – Isolating resources (threads, connections) per component so one failure doesn't sink the whole system.
  *Example: Giving the "recommendations" service its own thread pool so it can't exhaust threads needed by "checkout."*
- **Graceful Degradation** – Reducing functionality instead of failing completely when under stress.
  *Example: If the recommendation engine is down, show a generic "popular items" list instead of an error page.*
- **Backpressure** – Signaling upstream producers to slow down when a downstream consumer can't keep up.
  *Example: A queue tells the producer to pause sending messages because it's full.*
- **Dead Letter Queue (DLQ)** – A queue where messages that repeatedly fail processing are moved for later inspection.
  *Example: A message that fails processing 5 times gets moved to `orders-dlq` instead of blocking the main queue.*
- **Poison Message** – A message that always causes a consumer to fail/crash, no matter how many times it's retried.
  *Example: A malformed JSON message that keeps crashing the parser.*

---

## Consistency & Distributed Data

- **Strong Consistency** – Every read gets the most recent write, immediately, everywhere.
  *Example: A bank balance update is visible to all readers instantly after the write commits.*
- **Eventual Consistency** – Reads may return stale data temporarily but will "eventually" converge.
  *Example: A social media post takes a few seconds to show up on all followers' feeds.*
- **Read Your Own Writes** – A user always sees their own updates immediately, even if others don't yet.
  *Example: You post a comment and see it instantly, though your friend might see it a few seconds later.*
- **Monotonic Reads** – Once you've read a value, you never see an older value in a later read.
  *Example: You won't see your comment, refresh, and have it disappear.*
- **Replication Lag** – The delay between a write on the primary and it appearing on replicas.
  *Example: A write to the primary DB takes 200ms to appear on a read replica.*
- **Split Brain** – Two nodes both believe they are the leader, causing conflicting writes.
  *Example: A network partition causes two database nodes to both accept writes as "primary."*
- **Quorum** – The minimum number of nodes that must agree for an operation to be considered successful.
  *Example: In a 5-node cluster, a write needs acknowledgment from 3 nodes (majority) to succeed.*
- **Leader Election** – The process of choosing a coordinator node among a distributed set of nodes.
  *Example: If the Kafka controller node dies, remaining brokers elect a new controller.*
- **Consensus** – Getting multiple nodes to agree on a single value/decision despite failures (e.g., Raft, Paxos).
  *Example: Raft-based etcd cluster agreeing on the current leader.*
- **Linearizability** – The strongest consistency model — operations appear to happen instantaneously at some point between invocation and response, in real-time order.
  *Example: If write A finishes before write B starts, every reader sees A before B.*

---

## Messaging & Event-Driven Architecture

- **At-most-once Delivery** – A message is delivered 0 or 1 times — it may be lost but never duplicated.
  *Example: Fire-and-forget UDP-style logging where losing a log line is acceptable.*
- **At-least-once Delivery** – A message is delivered 1 or more times — never lost, but might be duplicated.
  *Example: Most message queues (SQS, Kafka default) — consumers must handle duplicates.*
- **Exactly-once Processing** – A message is processed as if delivered exactly once, even if it was technically delivered more than once (via dedup logic).
  *Example: Kafka transactions + idempotent consumers ensuring a payment is charged only once.*
- **Producer** – The component that publishes/sends messages.
  *Example: An order service publishing "OrderCreated" events.*
- **Consumer** – The component that reads/processes messages.
  *Example: An inventory service subscribing to "OrderCreated" events.*
- **Consumer Group** – A set of consumers that share the work of reading from a topic's partitions.
  *Example: 3 instances of the inventory service in one Kafka consumer group split the 6 partitions among themselves.*
- **Ordering Guarantee** – Assurance that messages are processed in the order they were sent (usually within a partition).
  *Example: All events for `orderId=123` land on the same Kafka partition, preserving order.*
- **Partitioning** – Splitting a topic/dataset into multiple independent segments for parallelism.
  *Example: A Kafka topic with 10 partitions allows 10 consumers to read in parallel.*
- **Replay** – Re-processing historical messages/events from a log.
  *Example: Resetting a Kafka consumer's offset to 0 to reprocess all events from the beginning.*
- **Event Sourcing** – Storing state as a sequence of events rather than just the current state.
  *Example: Instead of storing `balance: $100`, store events `Deposited $50`, `Deposited $50`.*
- **CQRS (Command Query Responsibility Segregation)** – Separating the write model (commands) from the read model (queries).
  *Example: Writes go to a normalized SQL DB; reads are served from a denormalized Elasticsearch index.*
- **Outbox Pattern** – Writing an event to an "outbox" table in the same DB transaction as the business change, then publishing it asynchronously — avoids dual-write inconsistency.
  *Example: An `orders` table insert and an `outbox` event insert happen in one transaction; a separate process publishes the outbox row to Kafka.*
- **Saga Pattern** – Managing a distributed transaction as a sequence of local transactions with compensating actions on failure.
  *Example: Book flight → book hotel → charge card; if charging fails, cancel the hotel and flight bookings.*
- **Compensating Transaction** – An action that undoes the effect of a previous transaction when a saga step fails.
  *Example: "CancelHotelBooking" undoes "BookHotel" if a later step fails.*

---

## Reliability & Resilience

- **Fault Tolerance** – The system continues operating correctly even when some components fail.
  *Example: A 3-node database cluster keeps serving reads/writes if one node crashes.*
- **High Availability (HA)** – The system stays up and accessible for a very high percentage of time.
  *Example: "99.99% uptime" (about 52 minutes of downtime/year).*
- **Redundancy** – Having duplicate components so failure of one doesn't take down the system.
  *Example: Running 3 replicas of a web server behind a load balancer.*
- **Failover** – Automatically switching to a backup/standby component when the primary fails.
  *Example: If the primary DB dies, a standby replica is promoted to primary automatically.*
- **Single Point of Failure (SPOF)** – A component whose failure brings down the whole system.
  *Example: A single load balancer with no backup — if it dies, the whole site goes down.*
- **Blast Radius** – The scope/extent of impact when something fails.
  *Example: Deploying a bad config to one region instead of all regions limits the blast radius.*
- **Resilience** – The system's ability to recover quickly from failures.
  *Example: Auto-restarting a crashed container within seconds via Kubernetes.*
- **Disaster Recovery (DR)** – Plans/processes to recover systems after a major outage or disaster.
  *Example: Restoring services in a secondary AWS region after the primary region goes down.*
- **RTO (Recovery Time Objective)** – The maximum acceptable time to restore service after a failure.
  *Example: "We must be back online within 1 hour of an outage."*
- **RPO (Recovery Point Objective)** – The maximum acceptable amount of data loss, measured in time.
  *Example: "We can lose at most 5 minutes of data" → backups every 5 minutes.*

---

## Scalability

- **Vertical Scaling** – Adding more power (CPU/RAM) to an existing machine.
  *Example: Upgrading a server from 8GB to 64GB RAM.*
- **Horizontal Scaling** – Adding more machines/instances to share the load.
  *Example: Going from 2 to 10 web server instances behind a load balancer.*
- **Elasticity** – Automatically scaling resources up/down based on demand.
  *Example: AWS Auto Scaling adds instances during a traffic spike and removes them afterward.*
- **Load Balancing** – Distributing incoming requests across multiple servers.
  *Example: Nginx routing traffic round-robin across 5 backend servers.*
- **Statelessness** – Servers don't store session/client state between requests, making them easily replaceable/scalable.
  *Example: A REST API storing session data in Redis instead of in server memory.*
- **Sharding** – Splitting a large dataset across multiple databases/nodes.
  *Example: Users A-M on DB1, users N-Z on DB2.*
- **Partitioning** – Same concept as sharding, generically dividing data or workload into segments.
  *Example: Partitioning a table by date range (2024-Q1, 2024-Q2, ...).*
- **Hot Partition** – A partition/shard that receives disproportionately more traffic than others, becoming a bottleneck.
  *Example: A celebrity's user ID gets so much traffic it overwhelms its shard while others sit idle.*
- **Fan-out** – One request/event triggers multiple downstream calls/events.
  *Example: A new post triggers notifications to all 10,000 followers.*
- **Fan-in** – Multiple sources send data into a single consumer/aggregator.
  *Example: Logs from 100 servers all flowing into one centralized logging pipeline.*

---

## Performance

- **Latency** – Time taken for a single operation/request to complete.
  *Example: An API responds in 150ms.*
- **Throughput** – The number of operations/requests processed per unit time.
  *Example: A server handles 5,000 requests per second.*
- **Concurrency** – Multiple tasks making progress during overlapping time periods (not necessarily simultaneously).
  *Example: A single-core server handling multiple requests by interleaving them.*
- **Parallelism** – Multiple tasks executing literally at the same time (needs multiple cores/machines).
  *Example: Splitting a large computation across 8 CPU cores.*
- **Little's Law** – A queuing theory formula: `L = λ × W` (avg items in system = arrival rate × avg time in system).
  *Example: If 10 requests/sec arrive and each takes 2 sec to process, ~20 requests are in the system on average.*
- **P50, P95, P99** – Percentile latency metrics — P99 means 99% of requests were faster than this value.
  *Example: P99 latency of 800ms means only 1% of requests took longer than 800ms.*
- **Tail Latency** – The latency experienced by the slowest requests (the "tail" of the distribution, e.g., P99/P999).
  *Example: Most requests take 50ms, but tail latency (P99) is 2 seconds due to GC pauses.*
- **Bottleneck** – The component limiting overall system performance.
  *Example: A single-threaded database connection pool limiting throughput even though app servers scale fine.*
- **Contention** – Multiple processes/threads competing for the same limited resource.
  *Example: Many threads trying to acquire the same database row lock simultaneously.*
- **Connection Pooling** – Reusing a fixed set of open connections instead of creating new ones per request.
  *Example: A DB connection pool of 20 reused connections instead of opening/closing a connection per query.*
- **Caching** – Storing frequently accessed data in fast storage to avoid recomputation/refetching.
  *Example: Storing API responses in Redis for 60 seconds.*
- **Cache Hit Ratio** – The percentage of requests served from cache vs. going to the source.
  *Example: 95% cache hit ratio means only 5% of requests hit the actual database.*
- **Cache Stampede** – Many requests simultaneously miss the cache (e.g., on expiry) and hammer the backend at once.
  *Example: A popular cached item expires, and 1,000 concurrent requests all hit the DB at the same instant.*
- **Thundering Herd** – Many processes/threads waking up simultaneously to compete for a resource, overwhelming it.
  *Example: 500 workers all polling a queue at the exact same interval, spiking load every second.*
- **N+1 Problem** – Making 1 query to fetch a list, then N additional queries (one per item) instead of a single batched query.
  *Example: Fetching 100 blog posts, then running a separate query for each post's author (101 queries instead of 2).*

---

## Caching

- **Cache-aside** – Application checks cache first; on miss, reads from DB and populates the cache.
  *Example: `if not in cache: fetch from DB; cache.set(key, value)`.*
- **Write-through** – Writes go to the cache and the database at the same time (synchronously).
  *Example: Updating a user's profile updates both Redis and the DB before returning success.*
- **Write-behind (write-back)** – Writes go to the cache immediately, and the DB is updated asynchronously later.
  *Example: Cache is updated instantly; a background job flushes changes to the DB every few seconds.*
- **TTL (Time To Live)** – How long a cached item stays valid before expiring.
  *Example: `cache.set(key, value, ttl=300)` — expires after 5 minutes.*
- **Cache Invalidation** – Explicitly removing/updating stale cache entries when the underlying data changes.
  *Example: Deleting the cached user profile when the user updates their email.*
- **Cache Eviction** – Removing entries from a full cache based on a policy (LRU, LFU, etc.).
  *Example: LRU (Least Recently Used) evicts the item that hasn't been accessed in the longest time.*
- **Stale Data** – Cached data that no longer matches the current source of truth.
  *Example: A cached product price still shows $10 after it was updated to $8 in the DB.*
- **Distributed Cache** – A cache shared across multiple application instances/servers.
  *Example: A shared Redis cluster used by all 20 instances of a web app.*
- **Consistent Hashing** – A hashing technique that minimizes redistribution of keys when nodes are added/removed.
  *Example: Adding a new cache node only remaps ~1/N of keys instead of nearly all of them.*

---

## Database & Transactions

- **ACID** – Atomicity, Consistency, Isolation, Durability — guarantees for reliable transactions.
  *Example: A bank transfer either fully completes (debit + credit) or not at all.*
- **BASE** – Basically Available, Soft state, Eventually consistent — the tradeoff model used by many NoSQL systems.
  *Example: A DynamoDB table that's always available for writes but may return stale reads briefly.*
- **Isolation Levels** – Rules defining how visible one transaction's changes are to concurrent transactions (Read Uncommitted, Read Committed, Repeatable Read, Serializable).
  *Example: PostgreSQL defaults to "Read Committed" isolation.*
- **Dirty Read** – Reading uncommitted changes from another transaction that might later be rolled back.
  *Example: Transaction A reads a balance updated by Transaction B, but B then rolls back — A read invalid data.*
- **Non-repeatable Read** – Reading the same row twice within a transaction and getting different values because another transaction modified it in between.
  *Example: You read a price as $10, then read it again in the same transaction and it's now $12.*
- **Phantom Read** – A query run twice in the same transaction returns a different set of rows because another transaction inserted/deleted rows.
  *Example: `SELECT COUNT(*) WHERE status='pending'` returns 5, then 7, within the same transaction.*
- **Optimistic Locking** – Assume conflicts are rare; check a version number at write time and reject if it changed.
  *Example: Updating a row with `WHERE id=1 AND version=3` — fails if another update already bumped the version.*
- **Pessimistic Locking** – Lock the row upfront before reading/updating to prevent others from touching it.
  *Example: `SELECT ... FOR UPDATE` locks the row until the transaction commits.*
- **MVCC (Multi-Version Concurrency Control)** – Keeping multiple versions of data so readers don't block writers and vice versa.
  *Example: PostgreSQL lets a long-running read see a consistent snapshot while writes continue concurrently.*
- **Index Selectivity** – How well an index narrows down rows — higher selectivity (fewer matching rows per value) means a more useful index.
  *Example: An index on `email` (mostly unique) is highly selective; an index on `gender` (few distinct values) is not.*

---

## System Design Fundamentals

- **CAP Theorem** – A distributed system can only guarantee 2 of 3: Consistency, Availability, Partition Tolerance.
  *Example: During a network partition, a system must choose to stay consistent (reject some requests) or available (serve possibly stale data).*
- **PACELC** – Extension of CAP: even without a Partition, there's a tradeoff between Latency and Consistency.
  *Example: A system might sacrifice consistency for lower latency even when the network is healthy.*
- **Availability** – The system responds successfully to requests (even if data might be stale).
  *Example: The site stays up and returns responses even during a partial outage.*
- **Consistency** (in CAP) – All nodes see the same data at the same time.
  *Example: A read after a write always returns the latest value, everywhere.*
- **Partition Tolerance** – The system continues to operate despite network failures between nodes.
  *Example: The system keeps working even if two data centers can't communicate with each other.*
- **Durability** – Once a transaction is committed, it stays committed even if the system crashes.
  *Example: A confirmed order isn't lost even if the server crashes 1 second after commit.*
- **Atomicity** – A transaction either fully completes or fully fails — no partial changes.
  *Example: A money transfer doesn't debit one account without crediting the other.*
- **Isolation** (ACID) – Concurrent transactions don't interfere with each other's intermediate state.
  *Example: Two people booking the last seat simultaneously — only one succeeds, cleanly.*
- **Eventual Consistency** – (see above) Replicas converge to the same value over time.

---

## Distributed Systems Coordination

- **Distributed Lock** – A lock that coordinates access to a resource across multiple machines/processes.
  *Example: Using Redis (`SETNX`) or Zookeeper to ensure only one worker processes a job at a time.*
- **Lease** – A time-bound lock/grant of ownership that automatically expires if not renewed.
  *Example: A worker holds a 30-second lease on a task; if it crashes, the lease expires and another worker picks it up.*
- **Heartbeat** – Periodic signal sent to indicate a node/process is still alive.
  *Example: A worker pings a coordinator every 5 seconds; missing 3 heartbeats marks it as dead.*
- **Fencing Token** – A monotonically increasing token used to prevent a "zombie" client (with an expired lock) from making unsafe writes.
  *Example: Client A gets lock token 5, gets paused, lock expires, Client B gets token 6; storage rejects A's write with the older token 5.*
- **Leader Election** – (see above) Choosing a single coordinator among peers.
- **Gossip Protocol** – Nodes periodically exchange state info with random peers, spreading information across the cluster over time.
  *Example: Cassandra nodes use gossip to learn about cluster membership and node health.*
- **Vector Clock** – A mechanism to track causality/ordering of events across distributed nodes without relying on synchronized clocks.
  *Example: `[A:2, B:1]` shows node A has seen 2 of its own events and 1 from B, helping detect conflicting concurrent updates.*
- **Lamport Clock** – A simpler logical clock providing a partial ordering of events in a distributed system.
  *Example: Each event increments a counter; if event X causes event Y, X's counter is guaranteed to be lower.*
- **Clock Skew** – The difference in time between clocks on different machines.
  *Example: Server A's clock is 3 seconds ahead of Server B's, causing timestamp-based ordering issues.*

---

## Data Architecture

- **Database Replication** – Copying data from one database to one or more others.
  *Example: A primary DB streams its write-ahead log to 2 replicas.*
- **Read Replica** – A replica used only to serve read queries, offloading the primary.
  *Example: Analytics queries run against a read replica instead of the primary production DB.*
- **Primary-Replica** – Architecture where one node (primary) accepts writes and others (replicas) mirror it for reads.
  *Example: MySQL primary handles writes; 3 replicas handle read traffic.*
- **Multi-Leader Replication** – Multiple nodes can accept writes, and changes are synced between them.
  *Example: Two data centers each accept local writes and replicate changes to each other (can cause conflicts).*
- **Multi-Region Replication** – Data is replicated across geographically distant regions for availability/latency.
  *Example: A database replicated across US, EU, and Asia regions so users read from the nearest copy.*
- **Data Locality** – Keeping computation close to where the data resides to reduce network overhead.
  *Example: Running a Spark job on the same node that stores the data partition it needs.*
- **Data Partitioning** – (see Partitioning above) Dividing data across nodes/storage.
- **Data Skew** – Uneven distribution of data/load across partitions.
  *Example: 80% of records belong to one customer, overloading that customer's partition.*
- **Hot Key** – A specific key that receives disproportionately high traffic, causing a bottleneck.
  *Example: A "trending" product ID gets hit by millions of cache reads per second.*

---

## Microservices Patterns

- **API Gateway** – A single entry point that routes client requests to the appropriate backend microservices.
  *Example: `api.example.com/orders` routes to the Order Service; `/users` routes to the User Service.*
- **Service Discovery** – Mechanism for services to find each other's network locations dynamically.
  *Example: A service registers itself with Consul/Eureka, and other services look it up by name instead of hardcoded IP.*
- **Service Mesh** – Infrastructure layer (often sidecars) handling service-to-service communication, security, and observability.
  *Example: Istio managing retries, mTLS, and metrics between microservices transparently.*
- **Sidecar Pattern** – Deploying a helper process alongside the main service container to handle cross-cutting concerns.
  *Example: An Envoy proxy container running next to the app container to handle networking/logging.*
- **Strangler Fig Pattern** – Gradually replacing a legacy system by routing pieces of functionality to a new system over time.
  *Example: New checkout flow is built as a microservice while the old monolith still handles everything else, until it's fully replaced.*
- **Ambassador Pattern** – A proxy sidecar that handles outgoing network calls on behalf of the main application (retries, monitoring).
  *Example: An ambassador container manages TLS and retries for calls the app makes to an external API.*
- **Backend for Frontend (BFF)** – A dedicated backend tailored to a specific frontend client's needs.
  *Example: A separate BFF API for the mobile app vs. one for the web app, each shaping data differently.*
- **Anti-Corruption Layer** – A translation layer that isolates a system from a legacy or external system's data model.
  *Example: A adapter service converts legacy XML SOAP responses into a clean JSON model for new services.*

---

## Transactions & Consistency Patterns

- **Two-Phase Commit (2PC)** – A protocol where a coordinator asks all participants to "prepare" then "commit" a distributed transaction.
  *Example: A payment DB and inventory DB both must agree to commit before either finalizes the change.*
- **Three-Phase Commit (3PC)** – An extension of 2PC adding an extra phase to reduce the chance of blocking on coordinator failure.
  *Example: Adds a "pre-commit" acknowledgment step between prepare and commit to avoid indefinite blocking.*
- **Saga Choreography** – Each service listens for events and reacts independently — no central coordinator.
  *Example: Order Service emits "OrderCreated" → Payment Service reacts and emits "PaymentCompleted" → Shipping Service reacts, etc.*
- **Saga Orchestration** – A central orchestrator explicitly tells each service what to do next.
  *Example: An Orchestrator service calls "ChargePayment", then "ReserveInventory", then "ShipOrder" in sequence.*
- **Transactional Outbox** – (see Outbox Pattern above) Ensures DB update and event publish happen atomically.
- **Inbox Pattern** – Tracking already-processed message IDs on the consumer side to prevent duplicate processing.
  *Example: Before processing a message, check if its ID is already in the `processed_messages` table; skip if so.*
- **Idempotent Consumer** – A consumer designed so processing the same message multiple times has no extra effect.
  *Example: A "charge card" consumer checks if `paymentId` was already charged before charging again.*

---

## Observability

- **Logging** – Recording discrete events for later debugging/auditing.
  *Example: `INFO: User 123 logged in at 10:04:32`.*
- **Metrics** – Numeric measurements collected over time, often aggregated.
  *Example: `requests_per_second`, `cpu_usage_percent`.*
- **Tracing** – Tracking a single request's journey across multiple services.
  *Example: Seeing that a request took 20ms in the API Gateway, 150ms in the Order Service, and 80ms in the DB.*
- **Distributed Tracing** – Tracing extended across many microservices to visualize the full call chain.
  *Example: Jaeger/Zipkin showing a request's path through 8 different microservices.*
- **Correlation ID** – A unique ID attached to a request that's passed through all services it touches, to correlate logs.
  *Example: `X-Correlation-ID: abc-123` included in every log line related to that request across all services.*
- **Trace ID** – The unique identifier for an entire distributed trace (one user request end-to-end).
  *Example: One `traceId` ties together all spans across all services for a single checkout request.*
- **Span** – A single unit of work within a trace (e.g., one service call or DB query).
  *Example: The "DB query" span shows it took 12ms within the larger request trace.*
- **SLIs (Service Level Indicators)** – Actual measured metrics of service behavior.
  *Example: "99.95% of requests succeeded in the last 30 days" (measured value).*
- **SLOs (Service Level Objectives)** – Target goals for SLIs.
  *Example: "We aim for 99.9% availability per month."*
- **SLA (Service Level Agreement)** – A contractual commitment (often with penalties) based on SLOs.
  *Example: "If uptime falls below 99.9%, the customer gets a service credit."*
- **Error Budget** – The allowed amount of failure/downtime before violating an SLO.
  *Example: A 99.9% SLO allows ~43 minutes of downtime per month — that's the error budget.*

---

## Security Architecture

- **Authentication** – Verifying who a user/system is.
  *Example: Logging in with a username and password.*
- **Authorization** – Verifying what an authenticated user is allowed to do.
  *Example: A regular user can view their own orders but not delete other users' accounts.*
- **OAuth 2.0** – An authorization framework allowing third-party apps limited access to a user's resources without sharing passwords.
  *Example: "Sign in with Google" granting an app access to your email without giving it your Google password.*
- **OpenID Connect (OIDC)** – An identity layer built on top of OAuth 2.0 for authentication (proving who you are).
  *Example: Logging into an app and receiving an ID Token confirming your identity via Google.*
- **JWT (JSON Web Token)** – A compact, signed token used to securely transmit claims (e.g., user identity) between parties.
  *Example: A JWT containing `{userId: 123, role: "admin"}`, signed so the server can verify it wasn't tampered with.*
- **RBAC (Role-Based Access Control)** – Access is granted based on a user's assigned role.
  *Example: "Admin" role can delete users; "Viewer" role can only read data.*
- **ABAC (Attribute-Based Access Control)** – Access decisions based on attributes (user, resource, environment), not just roles.
  *Example: "Allow access only if user.department == resource.department AND time is business hours."*
- **Least Privilege** – Granting only the minimum permissions necessary to perform a task.
  *Example: A logging service can only write to the logs bucket, not read/delete other S3 buckets.*
- **Zero Trust** – Never automatically trust any request, even from inside the network — verify every request.
  *Example: Every internal service call requires mTLS authentication, even between services in the same data center.*
- **Encryption at Rest** – Data is encrypted while stored on disk.
  *Example: An S3 bucket with server-side encryption enabled (AES-256).*
- **Encryption in Transit** – Data is encrypted while being transmitted over the network.
  *Example: HTTPS/TLS encrypting API traffic between client and server.*
- **Key Rotation** – Periodically replacing cryptographic keys to limit the impact of a compromised key.
  *Example: Automatically rotating an API's signing key every 90 days.*

---

## Cloud & Infrastructure

- **Auto Scaling** – Automatically adjusting the number of running instances based on load.
  *Example: AWS EC2 Auto Scaling adds instances when CPU usage exceeds 70%.*
- **Infrastructure as Code (IaC)** – Managing infrastructure through machine-readable config files instead of manual setup.
  *Example: Defining an AWS VPC and EC2 instances in a Terraform `.tf` file.*
- **Immutable Infrastructure** – Servers/instances are never modified after deployment — replaced entirely instead.
  *Example: Instead of patching a running server, deploy a brand-new instance with the updated image and terminate the old one.*
- **Blue-Green Deployment** – Running two identical environments (blue = current, green = new); switch traffic all at once.
  *Example: Deploy v2 to the "green" environment, test it, then flip the load balancer from blue to green instantly.*
- **Canary Deployment** – Gradually rolling out a new version to a small subset of users before full rollout.
  *Example: Route 5% of traffic to v2; if no errors after an hour, increase to 25%, then 100%.*
- **Rolling Deployment** – Gradually replacing old instances with new ones, one (or a few) at a time.
  *Example: Updating servers 1-by-1 out of a fleet of 20, so the app stays available throughout.*
- **Multi-AZ (Availability Zone)** – Deploying resources across multiple isolated data centers within a region for fault tolerance.
  *Example: An RDS database with a standby replica in a different AZ within the same AWS region.*
- **Multi-Region** – Deploying resources across multiple geographic regions for disaster recovery/latency.
  *Example: Running the application in both `us-east-1` and `eu-west-1`.*

---

## Queues & Streaming

- **Message Queue** – A buffer that holds messages between producers and consumers, typically each message consumed once.
  *Example: SQS holding "SendEmail" tasks until a worker picks them up.*
- **Pub/Sub** – Publish-subscribe model where messages are broadcast to all interested subscribers.
  *Example: An "OrderCreated" event is published, and both the Email Service and Analytics Service subscribe and each get a copy.*
- **Event Stream** – A continuous, ordered log of events (e.g., Kafka topic) that can be read/replayed.
  *Example: A Kafka topic recording every "click" event on a website.*
- **Consumer Offset** – The position/pointer indicating how far a consumer has read in a stream/partition.
  *Example: Consumer has processed up to offset 4,502 in a partition.*
- **Consumer Lag** – The gap between the latest message produced and the last message a consumer has processed.
  *Example: Producer is at offset 5,000, consumer is at offset 4,502 → lag of 498 messages.*
- **Partition Key** – A value used to determine which partition a message is routed to (ensures related messages stay ordered).
  *Example: Using `userId` as the partition key so all events for a user land in the same partition, in order.*
- **Ordering** – (see above) Guarantee on the sequence in which messages are processed.
- **Retention** – How long messages/events are kept before being deleted.
  *Example: A Kafka topic configured to retain messages for 7 days.*
- **Compaction** – Keeping only the latest value per key in a log, discarding older values for the same key.
  *Example: A Kafka compacted topic keeps only the most recent "UserProfileUpdated" event per `userId`, discarding earlier ones.*
- **Replay** – (see above) Re-reading historical events from a stream.

---

## Failure & Recovery

- **Fail Fast** – Detecting and reporting failures immediately rather than continuing in a broken state.
  *Example: Validating input at the start of a function and throwing an error immediately instead of silently continuing.*
- **Failover** – (see above) Automatic switch to a backup component on failure.
- **Fallback** – A default/alternative response used when the primary operation fails.
  *Example: If the recommendation service times out, fall back to showing a static list of popular items.*
- **Retry Storm** – A surge of retries from many clients simultaneously overwhelming a recovering system.
  *Example: 10,000 clients all retry at once right when a service comes back up, immediately crashing it again.*
- **Cascading Failure** – A failure in one component triggers failures in dependent components, spreading through the system.
  *Example: The DB slows down → API servers queue up requests → they exhaust threads → the whole app becomes unresponsive.*
- **Partial Failure** – Only some parts of a distributed operation fail while others succeed.
  *Example: A fan-out request to 5 services — 4 succeed but 1 times out.*
- **Chaos Engineering** – Deliberately injecting failures into a system to test its resilience.
  *Example: Netflix's "Chaos Monkey" randomly kills production instances to verify the system recovers gracefully.*
- **Disaster Recovery** – (see above) Plans to recover from major outages.
- **Recovery Point** – (see RPO above) The data state you can recover to.
- **Recovery Time** – (see RTO above) The time needed to restore service.

---

## Architecture Principles

- **Separation of Concerns** – Different parts of a system should each address a distinct responsibility.
  *Example: Keeping database access code separate from business logic and UI rendering code.*
- **Loose Coupling** – Components depend on each other as little as possible, so changes in one don't ripple into others.
  *Example: Services communicate via well-defined APIs/events rather than sharing a database directly.*
- **High Cohesion** – Related functionality is grouped together within a single module/component.
  *Example: All order-related logic (create, update, cancel) lives in the Order Service, not scattered across services.*
- **Single Responsibility** – A class/module/service should have only one reason to change.
  *Example: A `PasswordValidator` class only validates passwords — it doesn't also send emails.*
- **Dependency Inversion** – High-level modules shouldn't depend on low-level modules directly; both should depend on abstractions.
  *Example: A `PaymentService` depends on a `PaymentGateway` interface, not directly on `StripeClient`.*
- **DRY (Don't Repeat Yourself)** – Avoid duplicating logic; extract shared code into reusable components.
  *Example: Writing one `formatCurrency()` function instead of copy-pasting formatting logic in 10 places.*
- **KISS (Keep It Simple, Stupid)** – Prefer simple solutions over unnecessarily complex ones.
  *Example: Using a simple `if/else` instead of a complex design pattern for two conditions.*
- **YAGNI (You Aren't Gonna Need It)** – Don't build functionality until it's actually needed.
  *Example: Not adding a plugin system "just in case" when no current requirement needs it.*

---

## Design & Scalability Laws

- **Amdahl's Law** – The speedup from parallelizing a task is limited by its non-parallelizable (sequential) portion.
  *Example: If 10% of a task must run sequentially, even infinite parallel processors won't get you more than a 10x speedup.*
- **Little's Law** – (see above) `L = λ × W` — relates queue length, arrival rate, and wait time.
- **CAP Theorem** – (see above) Consistency, Availability, Partition Tolerance — pick 2 of 3 under a partition.
- **PACELC Theorem** – (see above) Extends CAP: even without partitions, trade Latency vs. Consistency.
- **FLP Impossibility** – In an asynchronous network, it's impossible to guarantee consensus if even one node can fail (no bound on message delay).
  *Example: A theoretical proof of why practical consensus protocols (like Raft) use timeouts/randomization to work around this in practice.*
- **Two Generals Problem** – A thought experiment showing it's impossible to guarantee two parties agree on an action via an unreliable network with 100% certainty.
  *Example: Two armies need to attack simultaneously but can only communicate via messengers who might get captured — neither can ever be 100% sure the other got the message.*
- **Byzantine Generals Problem** – Extends the Two Generals Problem to cases where some participants may act maliciously/send conflicting information.
  *Example: Blockchain consensus (e.g., proof-of-work) must reach agreement even when some nodes are actively lying.*
- **CALM Theorem** – "Consistency As Logical Monotonicity" — a program can be consistent without coordination if its logic is monotonic (only ever adds, never retracts information).
  *Example: A distributed "set union" operation (only adding elements) can be computed consistently without coordination, unlike a "delete" operation.*

