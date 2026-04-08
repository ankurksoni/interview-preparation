# Redis Interview Questions & Answers

A curated list of **Redis interview questions** — easy to medium difficulty with practical examples. Covers data structures, caching patterns, persistence, pub/sub, clustering, and real-world usage.

---

## Fundamentals

### 1. **What is Redis?**

**Answer:** Redis (Remote Dictionary Server) is an open-source, **in-memory** data structure store. It can be used as a database, cache, message broker, and streaming engine. Data lives in RAM, making it extremely fast (sub-millisecond latency).

Key characteristics:

- **Single-threaded** event loop (no lock contention)
- **Rich data structures** — not just key-value
- **Optional persistence** — RDB snapshots and AOF logs
- **Built-in replication** and clustering
- **Lua scripting** for atomic multi-step operations

---

### 2. **What data structures does Redis support?**

**Answer:**

| Type                        | Description                          | Example use case                  |
| --------------------------- | ------------------------------------ | --------------------------------- |
| **String**                  | Binary-safe string (max 512 MB)      | Cache, counters, session tokens   |
| **Hash**                    | Field-value map                      | User profiles, object storage     |
| **List**                    | Ordered collection (linked list)     | Job queues, activity feeds        |
| **Set**                     | Unordered unique values              | Tags, unique visitors             |
| **Sorted Set (ZSet)**       | Set with float scores, sorted        | Leaderboards, rate limiting       |
| **Stream**                  | Append-only log with consumer groups | Event sourcing, message queues    |
| **HyperLogLog**             | Probabilistic cardinality counter    | Unique visitor count (approx.)    |
| **Bitmap**                  | Bit-level operations on strings      | Feature flags, daily active users |
| **Geospatial**              | Lat/long indexed data                | Nearby search, geo-fencing        |
| **JSON** (RedisJSON module) | Native JSON document                 | Document storage                  |

---

### 3. **How is Redis different from Memcached?**

**Answer:**

| Feature           | Redis                                                        | Memcached            |
| ----------------- | ------------------------------------------------------------ | -------------------- |
| Data structures   | Strings, Hashes, Lists, Sets, Sorted Sets, Streams...        | Strings only         |
| Persistence       | RDB + AOF                                                    | None (pure cache)    |
| Replication       | Built-in primary-replica                                     | None                 |
| Clustering        | Redis Cluster (auto-sharding)                                | Client-side sharding |
| Pub/Sub           | ✅ Yes                                                       | ❌ No                |
| Lua scripting     | ✅ Yes                                                       | ❌ No                |
| Multithreaded I/O | I/O threads in Redis 6+ (command exec still single-threaded) | Multithreaded        |
| Max value size    | 512 MB                                                       | 1 MB                 |

**Choose Memcached** when: Simple string caching, multithreaded performance on multi-core.
**Choose Redis** when: Rich data structures, persistence, pub/sub, or anything beyond basic caching.

---

### 4. **What are the Redis persistence options?**

**Answer:**

| Option                     | How it works                             | Pros                           | Cons                         |
| -------------------------- | ---------------------------------------- | ------------------------------ | ---------------------------- |
| **RDB** (snapshotting)     | Periodic point-in-time snapshots to disk | Fast restart, compact files    | Data loss between snapshots  |
| **AOF** (Append Only File) | Logs every write command                 | Minimal data loss              | Larger files, slower restart |
| **RDB + AOF**              | Both enabled                             | Best durability + fast restart | More disk I/O                |
| **None**                   | No persistence                           | Maximum performance            | All data lost on restart     |

```
# redis.conf
save 900 1        # Snapshot if 1 key changed in 900 seconds
save 300 10       # Snapshot if 10 keys changed in 300 seconds
save 60 10000     # Snapshot if 10000 keys changed in 60 seconds

appendonly yes
appendfsync everysec   # Fsync every second (good balance)
```

> **Best practice for production:** Use AOF with `appendfsync everysec` for durability, plus RDB for fast disaster recovery restores.

---

### 5. **What is the difference between `SET` and `SETNX`?**

**Answer:**

- `SET key value` — Always sets the value (overwrites if exists)
- `SETNX key value` — **Set if Not eXists** — only sets if the key doesn't exist

```
> SET name "Alice"         # Always succeeds
OK
> SETNX name "Bob"         # Fails because "name" already exists
(integer) 0
> GET name
"Alice"

> SETNX newkey "value"     # Succeeds because "newkey" doesn't exist
(integer) 1
```

> Modern Redis: Use `SET key value NX EX 30` instead of `SETNX` — it combines set-if-not-exists + expiry atomically (used for distributed locks).

---

### 6. **How does Redis handle expiration (TTL)?**

**Answer:** Every key can have a TTL (time-to-live). Once expired, the key is automatically deleted.

```
> SET session:abc123 "user-data" EX 3600    # Expires in 1 hour
> TTL session:abc123                         # Check remaining time
(integer) 3598
> PERSIST session:abc123                     # Remove expiry (keep forever)
OK
> TTL session:abc123
(integer) -1                                 # -1 means no expiry
```

**How Redis actually deletes expired keys:**

1. **Lazy expiration** — Checked when a key is accessed
2. **Active expiration** — Redis periodically samples random keys and deletes expired ones

> Keys don't consume memory the instant they expire — there's a brief delay. Don't rely on exact-to-the-millisecond expiration for critical logic.

---

### 7. **What are Redis eviction policies?**

**Answer:** When Redis reaches `maxmemory`, it evicts keys based on the configured policy:

| Policy            | Behavior                                      |
| ----------------- | --------------------------------------------- |
| `noeviction`      | Return errors on writes (default)             |
| `allkeys-lru`     | Evict least recently used key from all keys   |
| `allkeys-lfu`     | Evict least frequently used key from all keys |
| `allkeys-random`  | Evict random key from all keys                |
| `volatile-lru`    | LRU among keys with TTL set                   |
| `volatile-lfu`    | LFU among keys with TTL set                   |
| `volatile-random` | Random among keys with TTL set                |
| `volatile-ttl`    | Evict keys with shortest TTL first            |

**Recommendation:**

- **Cache use case** → `allkeys-lru` or `allkeys-lfu`
- **Mixed (cache + persistent data)** → `volatile-lru` (only evicts keys with TTL)

---

## Caching Patterns

### 8. **What are the common caching patterns?**

**Answer:**

**Cache-Aside (Lazy Loading):**

```ts
async function getUser(userId: string): Promise<User> {
  // 1. Check cache
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);

  // 2. Cache miss → read from DB
  const user = await db.query("SELECT * FROM users WHERE id = $1", [userId]);

  // 3. Populate cache
  await redis.set(`user:${userId}`, JSON.stringify(user), "EX", 3600);

  return user;
}
```

**Write-Through:**

```ts
async function updateUser(userId: string, data: Partial<User>): Promise<void> {
  // 1. Write to DB
  await db.query("UPDATE users SET name = $1 WHERE id = $2", [
    data.name,
    userId,
  ]);

  // 2. Write to cache
  const user = await db.query("SELECT * FROM users WHERE id = $1", [userId]);
  await redis.set(`user:${userId}`, JSON.stringify(user), "EX", 3600);
}
```

**Write-Behind (Write-Back):**

- Write to cache immediately, asynchronously write to DB later
- Highest performance, but risk of data loss if cache crashes

---

### 9. **What is cache stampede and how do you prevent it?**

**Answer:** Cache stampede occurs when a popular key expires and many concurrent requests all miss the cache and hit the database simultaneously.

**Solutions:**

```ts
// 1. Mutex lock (only one request rebuilds cache)
async function getWithLock(
  key: string,
  buildFn: () => Promise<string>,
): Promise<string> {
  const cached = await redis.get(key);
  if (cached) return cached;

  const lockKey = `lock:${key}`;
  const acquired = await redis.set(lockKey, "1", "NX", "EX", 10);

  if (acquired) {
    try {
      const value = await buildFn();
      await redis.set(key, value, "EX", 3600);
      return value;
    } finally {
      await redis.del(lockKey);
    }
  }

  // Didn't get lock — wait and retry
  await new Promise((r) => setTimeout(r, 100));
  return getWithLock(key, buildFn);
}
```

```ts
// 2. Stale-while-revalidate (serve stale data while refreshing)
// Store expiry inside the value, actual Redis TTL is longer
await redis.set(
  key,
  JSON.stringify({
    data: value,
    logicalExpiry: Date.now() + 3600_000, // 1 hour
  }),
  "EX",
  7200,
); // Actual TTL: 2 hours (buffer)
```

---

### 10. **What is cache invalidation? Why is it hard?**

**Answer:** Cache invalidation is removing or updating cached data when the source data changes. It's hard because:

1. **Knowing what to invalidate** — A user profile change might affect multiple cached pages
2. **Race conditions** — Stale data can overwrite fresh data
3. **Distributed systems** — Multiple app servers may have inconsistent views

**Strategies:**

- **TTL-based** — Simplest. Acceptable staleness = TTL duration
- **Event-driven** — Publish events on data change, subscribers invalidate
- **Versioned keys** — `user:123:v5` — increment version on update

```ts
// ❌ Bad: Delete then set (race condition window)
await redis.del(`user:${id}`);
// Another request reads stale data from DB and sets old value here!
await redis.set(`user:${id}`, newData);

// ✅ Good: Atomic set with new value
await redis.set(`user:${id}`, JSON.stringify(newData), "EX", 3600);
```

---

## Data Structure Use Cases

### 11. **How do you implement a leaderboard with Redis?**

**Answer:** Use a **Sorted Set** — members have scores, automatically sorted.

```
> ZADD leaderboard 1500 "alice"
> ZADD leaderboard 2300 "bob"
> ZADD leaderboard 1800 "charlie"

# Top 3 players (highest score first)
> ZREVRANGE leaderboard 0 2 WITHSCORES
1) "bob"      2) "2300"
3) "charlie"  4) "1800"
5) "alice"    6) "1500"

# Alice's rank (0-based, highest first)
> ZREVRANK leaderboard "alice"
(integer) 2

# Increment alice's score by 500
> ZINCRBY leaderboard 500 "alice"
"2000"

# Count players with score between 1000 and 2000
> ZCOUNT leaderboard 1000 2000
(integer) 2
```

> Sorted Sets handle millions of members efficiently. `ZREVRANGE` is O(log(N) + M) where M is the number of returned elements.

---

### 12. **How do you implement rate limiting with Redis?**

**Answer:** Several approaches:

```ts
// Fixed window counter (simple)
async function isRateLimited(userId: string, limit: number, windowSec: number): Promise<boolean> {
  const key = `ratelimit:${userId}:${Math.floor(Date.now() / 1000 / windowSec)}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, windowSec);
  return count > limit;
}

// Sliding window with sorted set (more accurate)
async function slidingWindowRateLimit(userId: string, limit: number, windowMs: number): Promise<boolean> {
  const key = `ratelimit:${userId}`;
  const now = Date.now();
  const windowStart = now - windowMs;

  await redis
    .multi()
    .zremrangebyscore(key, 0, windowStart)           // Remove old entries
    .zadd(key, now, `${now}:${crypto.randomUUID()}`) // Add current request
    .zcard(key)                                       // Count requests in window
    .expire(key, Math.ceil(windowMs / 1000))          // Auto-cleanup
    .exec();

  const count = /* result of zcard */;
  return count > limit;
}
```

---

### 13. **How do you use Redis for session storage?**

**Answer:** Store session data as a Hash with TTL:

```ts
// Store session
await redis.hset(`session:${sessionId}`, {
  userId: "u-123",
  role: "admin",
  loginAt: Date.now().toString(),
  ip: "192.168.1.1",
});
await redis.expire(`session:${sessionId}`, 86400); // 24-hour TTL

// Read session
const session = await redis.hgetall(`session:${sessionId}`);
// { userId: 'u-123', role: 'admin', loginAt: '...', ip: '...' }

// Extend session on activity
await redis.expire(`session:${sessionId}`, 86400);

// Logout (delete session)
await redis.del(`session:${sessionId}`);
```

> **Why Hash over String?** You can read/update individual fields without serializing/deserializing the entire session object.

---

### 14. **How do you implement pub/sub in Redis?**

**Answer:** Redis Pub/Sub delivers messages to all subscribers in real-time. Messages are **fire-and-forget** — not persisted.

```ts
// Subscriber
const sub = redis.duplicate();
await sub.subscribe("notifications", (message) => {
  console.log("Received:", message);
});

// Pattern subscribe (wildcard)
await sub.psubscribe("order.*", (message, channel) => {
  console.log(`${channel}: ${message}`);
  // "order.created: {orderId: 123}"
  // "order.shipped: {orderId: 123}"
});

// Publisher
await redis.publish(
  "notifications",
  JSON.stringify({ type: "alert", text: "Server restarting" }),
);
await redis.publish("order.created", JSON.stringify({ orderId: 123 }));
```

> **Limitation:** If a subscriber is offline, messages are lost. For durable messaging, use **Redis Streams** instead.

---

### 15. **What are Redis Streams?**

**Answer:** Streams are an append-only log data structure (like Kafka) with consumer groups for reliable message processing.

```
# Producer: add events to stream
> XADD orders * action created orderId ORD-123 amount 99.99
"1712345678901-0"

> XADD orders * action paid orderId ORD-123
"1712345678902-0"

# Create consumer group
> XGROUP CREATE orders order-processors $ MKSTREAM

# Consumer: read new messages
> XREADGROUP GROUP order-processors worker-1 COUNT 5 BLOCK 5000 STREAMS orders >
1) "orders"
2) 1) 1) "1712345678901-0"
      2) 1) "action" 2) "created" 3) "orderId" 4) "ORD-123" 5) "amount" 6) "99.99"

# Acknowledge processing
> XACK orders order-processors 1712345678901-0
```

**Streams vs Pub/Sub:**
| Feature | Pub/Sub | Streams |
|---------|---------|---------|
| Persistence | ❌ Fire-and-forget | ✅ Stored on disk |
| Consumer groups | ❌ No | ✅ Yes |
| Replay | ❌ No | ✅ Read from any point |
| Acknowledgment | ❌ No | ✅ XACK per message |

---

## Clustering & Replication

### 16. **How does Redis replication work?**

**Answer:** Redis uses **asynchronous primary-replica replication**. Replicas maintain a copy of the primary's data.

```
Primary (read/write) ──async──▶ Replica 1 (read-only)
                      ──async──▶ Replica 2 (read-only)
```

- Replicas can serve **read** traffic (scale reads horizontally)
- Replication is **asynchronous** — a write acknowledged by the primary may not yet be on replicas
- If the primary fails, a replica can be **promoted** (manually or via Sentinel)

```
# Replica configuration
replicaof 192.168.1.100 6379
```

> **WAIT command:** For critical writes, `WAIT 1 5000` blocks until at least 1 replica acknowledges the write (within 5s timeout). This gives **synchronous-like** behavior.

---

### 17. **What is Redis Sentinel?**

**Answer:** Sentinel provides **high availability** — automatic failover when the primary goes down.

**What it does:**

1. **Monitoring** — Checks if primary and replicas are reachable
2. **Notification** — Alerts when something goes wrong
3. **Automatic failover** — Promotes a replica to primary
4. **Configuration provider** — Clients ask Sentinel for the current primary address

```
# sentinel.conf
sentinel monitor mymaster 192.168.1.100 6379 2    # 2 = quorum (sentinels that must agree)
sentinel down-after-milliseconds mymaster 5000      # 5s before marking as down
sentinel failover-timeout mymaster 60000             # 60s failover timeout
```

> **Production:** Run at least 3 Sentinel instances on separate machines for reliable quorum.

---

### 18. **What is Redis Cluster?**

**Answer:** Redis Cluster provides **automatic sharding** across multiple nodes. Data is distributed using **16,384 hash slots**.

```
Node A: slots 0–5460
Node B: slots 5461–10922
Node C: slots 10923–16383
```

Each key is hashed to a slot: `CRC16(key) % 16384`

```
# Keys go to different nodes
> SET user:1 "data"   → CRC16("user:1") % 16384 = slot 5649 → Node B
> SET user:2 "data"   → CRC16("user:2") % 16384 = slot 1337 → Node A
```

**Hash tags** — Force related keys to the same slot:

```
# Both go to the same slot because {order:123} is the hash tag
> SET {order:123}:details "..."
> SET {order:123}:items "..."
```

> **Important:** Multi-key commands (MGET, SUNION) only work if all keys are on the same node. Use hash tags to co-locate related keys.

---

### 19. **What is a Redis transaction?**

**Answer:** Redis transactions use `MULTI`/`EXEC` to execute commands atomically (all or nothing from other clients' perspective).

```
> MULTI
OK
> DECRBY account:alice 100
QUEUED
> INCRBY account:bob 100
QUEUED
> EXEC
1) (integer) 900
2) (integer) 1100
```

> **Important:** Redis transactions are NOT like SQL transactions — there's no rollback. If one command fails, others still execute. For conditional transactions, use `WATCH` (optimistic locking).

```
> WATCH account:alice       # Watch for changes
> GET account:alice
"1000"
> MULTI
> DECRBY account:alice 100
> INCRBY account:bob 100
> EXEC                      # Fails if account:alice was modified by another client
```

---

### 20. **What is Lua scripting in Redis?**

**Answer:** Lua scripts execute atomically on the server — no other commands run during script execution.

```ts
// Atomic rate limiter using Lua
const luaScript = `
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])

  local current = redis.call('INCR', key)
  if current == 1 then
    redis.call('EXPIRE', key, window)
  end

  if current > limit then
    return 0  -- Rate limited
  end
  return 1    -- Allowed
`;

// Use EVALSHA for performance (script is cached)
const sha = await redis.scriptLoad(luaScript);
const allowed = await redis.evalsha(sha, 1, `rate:${userId}`, 100, 60);
```

> **Why Lua over MULTI/EXEC?** Lua can read values and make decisions (if/else) within the atomic block. MULTI/EXEC only queues predefined commands.

---

## Practical Scenarios

### 21. **How do you use Redis as a distributed lock?**

**Answer:** Use the `SET NX EX` pattern (simplified Redlock):

```ts
async function acquireLock(
  resource: string,
  ttlMs: number,
): Promise<string | null> {
  const token = crypto.randomUUID();
  const acquired = await redis.set(
    `lock:${resource}`,
    token,
    "PX",
    ttlMs, // Expiry in milliseconds
    "NX", // Only if not exists
  );
  return acquired ? token : null;
}

async function releaseLock(resource: string, token: string): Promise<boolean> {
  // Lua script to delete only if we own the lock
  const script = `
    if redis.call('GET', KEYS[1]) == ARGV[1] then
      return redis.call('DEL', KEYS[1])
    end
    return 0
  `;
  const result = await redis.eval(script, 1, `lock:${resource}`, token);
  return result === 1;
}

// Usage
const token = await acquireLock("order:ORD-123", 10000);
if (token) {
  try {
    await processOrder("ORD-123");
  } finally {
    await releaseLock("order:ORD-123", token);
  }
}
```

> **Why use Lua for release?** Without it, another client could acquire the lock between your GET and DEL, and you'd delete their lock.

---

### 22. **How do you count unique visitors efficiently?**

**Answer:** Use **HyperLogLog** — probabilistic data structure that counts unique elements using ~12 KB regardless of cardinality.

```
> PFADD visitors:2024-04-08 "user-1" "user-2" "user-3"
> PFADD visitors:2024-04-08 "user-1" "user-4"     # user-1 is duplicate, not counted

> PFCOUNT visitors:2024-04-08
(integer) 4

# Merge multiple days
> PFMERGE visitors:week visitors:2024-04-08 visitors:2024-04-09
> PFCOUNT visitors:week
(integer) 42
```

> **Trade-off:** ~0.81% error rate. For exact counts, use a Set (`SADD`/`SCARD`), but that uses much more memory.

---

### 23. **What is pipelining in Redis?**

**Answer:** Pipelining sends multiple commands without waiting for each response, reducing network round-trips.

```ts
// ❌ Bad: 100 sequential round trips
for (const key of keys) {
  await redis.get(key); // Each waits for response
}

// ✅ Good: Single round trip with pipeline
const pipeline = redis.pipeline();
for (const key of keys) {
  pipeline.get(key);
}
const results = await pipeline.exec();
// [[null, 'value1'], [null, 'value2'], ...]
```

> Pipelining can improve throughput by **5–10x** for bulk operations. It's not atomic — for atomicity use `MULTI`/`EXEC` or Lua.

---

### 24. **What are the Redis memory optimization techniques?**

**Answer:**

1. **Use appropriate data structures** — Hash is more memory-efficient than separate keys for small objects
2. **Set maxmemory and eviction policy** — Prevent OOM crashes
3. **Use short key names in high-volume scenarios** — `u:123:n` vs `user:123:name`
4. **Compress values** — gzip/snappy before storing large strings
5. **Use TTL on everything ephemeral** — Don't let stale data accumulate
6. **Use OBJECT ENCODING** to check internal representation

```
> SET counter 12345
> OBJECT ENCODING counter
"int"                    # Stored as integer (8 bytes), not string

> HSET user:1 name Alice age 30 city NYC
> OBJECT ENCODING user:1
"listpack"               # Compact encoding for small hashes (< hash-max-listpack-entries)
```

---

### 25. **What is Redis ACL (Access Control List)?**

**Answer:** Redis 6+ introduced ACLs for fine-grained user authentication and command restriction.

```
# Create user with limited permissions
> ACL SETUSER app-readonly on >secretpassword ~cache:* +get +mget +hgetall -@dangerous

# Breakdown:
# on           → user is active
# >password    → set password
# ~cache:*     → can only access keys matching cache:*
# +get +mget   → allowed commands
# -@dangerous  → deny all commands in the "dangerous" category
```

> **Production:** Always disable the `default` user or set a password. Use separate ACL users for different applications/services.

---

## Error Handling, Indexing & Performance Deep Dive

### 26. **How do you handle connection errors and build resilient Redis clients?**

**Answer:** Redis connections can drop due to network issues, failovers, or server restarts. A production client must handle reconnection gracefully.

```js
import Redis from "ioredis";

const redis = new Redis({
  host: "redis-primary.example.com",
  port: 6379,
  retryStrategy(times) {
    const delay = Math.min(times * 200, 5000); // 200ms, 400ms, 600ms... cap at 5s
    console.warn(`Redis reconnecting... attempt ${times}, delay ${delay}ms`);
    return delay; // return null to stop retrying
  },
  maxRetriesPerRequest: 3, // fail individual commands after 3 retries
  enableReadyCheck: true, // wait for Redis to be fully loaded
  lazyConnect: false, // connect immediately
  reconnectOnError(err) {
    // Reconnect only on specific errors
    return err.message.includes("READONLY"); // happens during failover
  },
});

redis.on("connect", () => console.log("Redis connected"));
redis.on("ready", () => console.log("Redis ready to accept commands"));
redis.on("error", (err) => console.error("Redis error:", err.message));
redis.on("close", () => console.warn("Redis connection closed"));
redis.on("reconnecting", (delay) => console.warn(`Reconnecting in ${delay}ms`));

// Graceful degradation — cache miss is NOT a fatal error
async function getCached(key, fallbackFn) {
  try {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
  } catch (error) {
    console.warn(`Cache read failed for ${key}:`, error.message);
    // DON'T throw — fall through to the source
  }

  const fresh = await fallbackFn();
  try {
    await redis.set(key, JSON.stringify(fresh), "EX", 300);
  } catch (error) {
    console.warn(`Cache write failed for ${key}:`, error.message);
  }
  return fresh;
}
```

---

### 27. **How does Redis handle errors during Pub/Sub and Streams consumption?**

**Answer:**

```js
// Pub/Sub — subscriber must handle disconnections
const subscriber = new Redis();

subscriber.on("error", (err) => {
  console.error("Subscriber error:", err);
  // ioredis auto-reconnects and re-subscribes
});

subscriber.subscribe("orders", (err) => {
  if (err) console.error("Subscribe failed:", err);
});

subscriber.on("message", async (channel, message) => {
  try {
    const order = JSON.parse(message);
    await processOrder(order);
  } catch (error) {
    console.error("Message processing failed:", error);
    // Pub/Sub has NO acknowledgment — message is lost if processing fails
    // For reliable processing, use Streams instead
  }
});

// Streams — with consumer groups and acknowledgment
async function consumeStream() {
  while (true) {
    try {
      const results = await redis.xreadgroup(
        "GROUP",
        "order-processors",
        "consumer-1",
        "COUNT",
        10,
        "BLOCK",
        5000,
        "STREAMS",
        "order-stream",
        ">",
      );

      if (!results) continue;

      for (const [, messages] of results) {
        for (const [id, fields] of messages) {
          try {
            await processOrder(Object.fromEntries(fields));
            await redis.xack("order-stream", "order-processors", id); // ✅ acknowledged
          } catch (error) {
            console.error(`Failed to process ${id}:`, error);
            // NOT acknowledged — will be re-delivered via XPENDING + XCLAIM
          }
        }
      }
    } catch (error) {
      console.error("Stream read error:", error);
      await new Promise((r) => setTimeout(r, 2000)); // back off
    }
  }
}
```

> **Pub/Sub vs Streams for error handling:**
> | Feature | Pub/Sub | Streams |
> |---------|---------|---------|
> | Message persistence | ❌ Fire-and-forget | ✅ Stored on disk |
> | Acknowledgment | ❌ None | ✅ XACK per message |
> | Retry failed messages | ❌ Impossible | ✅ XPENDING + XCLAIM |
> | Consumer groups | ❌ All get all messages | ✅ Each message to one consumer |

---

### 28. **How does RediSearch provide secondary indexing in Redis?**

**Answer:** Native Redis has no secondary indexes (only key-based lookup). **RediSearch** module adds full indexing and query capabilities.

```sh
# Create an index on hash keys with prefix "user:"
FT.CREATE idx:users ON HASH PREFIX 1 "user:"
  SCHEMA
    name TEXT SORTABLE
    email TAG
    age NUMERIC SORTABLE
    city TAG
    bio TEXT

# Index is automatically updated when you SET hash fields
HSET user:1 name "Alice" email "alice@example.com" age 30 city "NYC" bio "Backend engineer"
HSET user:2 name "Bob" email "bob@example.com" age 25 city "LA" bio "Frontend developer"

# Query by name (full-text search)
FT.SEARCH idx:users "Alice"

# Filter by numeric range
FT.SEARCH idx:users "@age:[25 35]"

# Tag filter (exact match)
FT.SEARCH idx:users "@city:{NYC}"

# Combined query with sorting
FT.SEARCH idx:users "@city:{NYC|LA} @age:[20 40]" SORTBY age ASC LIMIT 0 10

# Aggregation
FT.AGGREGATE idx:users "*"
  GROUPBY 1 @city
  REDUCE COUNT 0 AS user_count
  SORTBY 2 @user_count DESC
```

| Feature               | Native Redis          | RediSearch          |
| --------------------- | --------------------- | ------------------- |
| Lookup by key         | ✅ O(1)               | ✅ O(1)             |
| Search by field value | ❌ Must SCAN all keys | ✅ Indexed O(log n) |
| Full-text search      | ❌                    | ✅                  |
| Numeric range queries | ❌                    | ✅                  |
| Aggregations          | ❌                    | ✅                  |
| Auto-index updates    | N/A                   | ✅                  |

---

### 29. **What are the key Redis performance factors and how do you optimize them?**

**Answer:**

| Factor               | Impact                                | Optimization                                                           |
| -------------------- | ------------------------------------- | ---------------------------------------------------------------------- |
| Key size             | Large keys waste memory               | Keep key names short: `u:123:s` not `user:123:session`                 |
| Value size           | Large values → slow reads             | Compress with gzip, split into hashes                                  |
| Number of keys       | More keys = more memory overhead      | Use hashes for small objects (ziplist encoding)                        |
| Command complexity   | O(n) commands block the single thread | Avoid `KEYS *`, `SMEMBERS` on huge sets — use `SCAN`                   |
| Network roundtrips   | Latency × count = slow                | Use pipelining, MGET/MSET, Lua scripts                                 |
| Persistence          | RDB fork + AOF rewrite use memory     | Schedule snapshots during low-traffic periods                          |
| Memory fragmentation | Wasted RAM                            | Monitor with `INFO memory`, restart if `mem_fragmentation_ratio` > 1.5 |

```sh
# Check slow commands
SLOWLOG GET 10

# Memory analysis
MEMORY DOCTOR
INFO memory
# Look for: used_memory_peak, mem_fragmentation_ratio

# Find big keys (run in background, won't block)
redis-cli --bigkeys

# Monitor commands in real-time (USE CAREFULLY in prod)
MONITOR  # shows every command — high overhead

# Latency diagnostics
redis-cli --latency         # continuous ping
redis-cli --latency-history # over time
redis-cli --intrinsic-latency 5  # measure system latency baseline
```

---

### 30. **How does Redis Cluster handle failures and maintain availability?**

**Answer:** Redis Cluster uses a **gossip protocol** where every node monitors every other node. Failover is automatic.

```
Failure detection flow:
1. Node A can't reach Node B → marks B as PFAIL (possibly failed)
2. Majority of masters mark B as PFAIL → promoted to FAIL
3. B's replica is elected as new master (replica with most data wins)
4. Cluster configuration updated, hash slots reassigned
5. Total failover time: 1-2 seconds (configurable)
```

| Configuration                   | Default | Recommended (HA)                             |
| ------------------------------- | ------- | -------------------------------------------- |
| `cluster-node-timeout`          | 15000ms | 5000ms (faster detection)                    |
| `cluster-require-full-coverage` | yes     | **no** (partial availability > total outage) |
| `cluster-allow-reads-when-down` | no      | **yes** (stale reads > no reads)             |
| Replicas per master             | 1       | 2 (survive 2 simultaneous failures)          |

> **`cluster-require-full-coverage: no`** is critical. By default, if even ONE hash slot is uncovered (master + all its replicas down), the **entire cluster stops accepting writes**. Setting this to `no` keeps the rest of the cluster working.
