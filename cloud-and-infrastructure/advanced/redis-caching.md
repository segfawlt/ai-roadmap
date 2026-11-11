# Performance — Redis Caching

**Unit 5:** Microservices Performance **Topic Code:** PERF-502 **Reading Time:** ~40 minutes

------------------------------------------------------------------------

## Learning Objectives

- Explain why caching is the most cost-effective performance lever for a microservices architecture.

- Describe Redis’s core data structures and the workloads each one fits.

- Compare the four canonical caching strategies: Cache-Aside, Read-Through, Write-Through, and Write-Behind.

- Choose an appropriate Time-To-Live (TTL) and invalidation approach for a given dataset.

- Identify the classic caching pitfalls — stampede, thundering herd, cache inconsistency — and the patterns that mitigate them.

------------------------------------------------------------------------

## Section 1: Why Cache?

### 1.1 The Cost of Every Database Round-Trip

A typical Postgres query on a well-indexed table takes 1–10 ms. A typical Redis `GET` takes under 1 ms, often under 0.2 ms. The difference sounds small until you multiply it by scale.

A checkout endpoint that fetches a user profile, a cart, and a shipping quote might issue three database queries per request, each costing ~5 ms of database time and ~50 KB of network payload. At 1,000 requests per second that is 3,000 queries/s hitting Postgres and ~150 MB/s of network inside the data center. Two thirds of the queries are repeat reads of unchanged data.

Caching the three reads in Redis collapses this to ~3 Redis calls per request (cheaper) and cuts database load by 60–80% (dramatically cheaper). The result: lower latency for users, smaller database instances, and fewer sleepless nights when traffic spikes.

### 1.2 When Not to Cache

Caching is not free. Every cached value is a potential source of stale data, a new failure mode, and a new invalidation bug. Do not cache when:

- The underlying data changes on every read (e.g., per-request randomized recommendations).

- Stale data is unacceptable (e.g., bank account balance during a withdrawal).

- The database can already serve the workload without strain.

**Rule of thumb:** introduce caching *only* when you can point to a metric that shows the database under pressure or the user waiting. Caching speculatively bloats complexity without measurable payoff.

------------------------------------------------------------------------

## Section 2: Redis as a Cache

Redis is a single-threaded, in-memory data structure server. It offers primitives far richer than a plain key-value store, and the right choice of primitive dramatically simplifies the application.

### 2.1 Core Data Structures

| Type | Use for |
|----|----|
| **String** | Simple cached blobs (serialized JSON, HTML fragments) |
| **Hash** | Object-like records where you want to read/update individual fields |
| **List** | Ordered queues, recent-activity feeds, bounded chat history |
| **Set** | Unique membership (tags, followers) |
| **Sorted Set (ZSet)** | Leaderboards, priority queues, scheduled work |
| **Stream** | Append-only event log with consumer groups |
| **Bitmap** | Compact boolean flags (daily active users via bitmap-of-user-ids) |
| **HyperLogLog** | Approximate cardinality in a fixed 12 KB |

For a cache you will mostly reach for **String** (serialized payloads) and **Hash** (when a caller often wants a single field of the cached object).

### 2.2 The Essential Commands

```
SET key value EX 600          # Set with 600s TTL
GET key                       # Retrieve
DEL key                       # Evict
EXPIRE key 600                # Set/refresh TTL on existing key
INCR key                      # Atomic counter increment
HSET key field value          # Write a single hash field
HGETALL key                   # Read all fields of a hash
LPUSH key value               # Push to the head of a list
LTRIM key 0 19                # Keep only the first 20 elements
ZADD leaderboard 1500 alice   # Sorted-set insert with score
```

------------------------------------------------------------------------

## Section 3: Caching Strategies

### 3.1 Cache-Aside (the default)

The application checks the cache first, falls through to the database on miss, and populates the cache on the way back.

``` python
def get_user(user_id: str) -> dict:
    key = f"user:{user_id}"
    cached = redis.get(key)
    if cached is not None:
        return json.loads(cached)

    user = db.query("SELECT ... FROM users WHERE id = %s", user_id)
    if user is not None:
        redis.set(key, json.dumps(user), ex=600)  # 10-minute TTL
    return user
```

This is the most common pattern and the right default. The cache is **lazy** — it only populates in response to real traffic — and the application stays in control of what lives in the cache.

### 3.2 Read-Through

The cache itself knows how to fetch from the database on miss. The application only talks to the cache. This decouples application code from cache logic but requires a cache library that supports loader callbacks (Caffeine in Java, `aiocache` in Python).

### 3.3 Write-Through

Every write goes through the cache, which synchronously updates the database. Writes are slower than cache-aside, but the cache is always consistent with the database.

### 3.4 Write-Behind (Write-Back)

Writes go to the cache and are batched asynchronously to the database. Very fast writes, but a cache crash before the batch flushes means *data loss*. Use only when loss is tolerable (e.g., analytics counters) and pair it with a durable message queue for the write stream.

### 3.5 Which to Use?

Start with **Cache-Aside**. It is simple, explicit, and works for 90% of microservice read paths. Move to Write-Through only when you have a specific consistency requirement that cache-aside cannot meet.

------------------------------------------------------------------------

## Section 4: Invalidation

> There are only two hard things in Computer Science: cache invalidation and naming things. — Phil Karlton

### 4.1 TTL-Based (Easiest)

Every cached value expires after a fixed interval. The cache self-heals on a cadence defined by the TTL. Pick the TTL based on the staleness your users can tolerate.

``` python
redis.set("product:42", payload, ex=300)  # 5 minutes stale OK
```

TTL is the simplest strategy and should be your default. It has one drawback: after the write, the cache is stale for up to one TTL.

### 4.2 Explicit Invalidation

On every write to the underlying data, delete the cached entry.

``` python
def update_user(user_id: str, changes: dict):
    db.update("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # let the next read repopulate
```

This gives near-zero staleness but requires every write path to know the full set of cache keys it invalidates. Easy to miss one; the bugs are subtle.

### 4.3 Write-Through as Invalidation

The caller writes the new value to *both* the database and the cache atomically. Read paths always see fresh data. Mostly used when reads dramatically outnumber writes and staleness is unacceptable.

------------------------------------------------------------------------

## Section 5: Pitfalls

### 5.1 Cache Stampede

When a popular key expires, every concurrent request that was relying on it misses the cache simultaneously. All of them hit the database at once. The database, already loaded, falls over.

**Mitigations:**

- **Early refresh.** Don’t wait for expiry; when TTL drops below a threshold, one request refreshes the cache in the background while others continue serving the old value.

- **Lock on miss.** On a cache miss, acquire a Redis lock for the key. Only the lock holder queries the database; others wait briefly and re-read the cache.

- **Jitter the TTL.** Add random variance to avoid many keys expiring at the same instant.

### 5.2 Thundering Herd at Cold Start

Immediately after a cache restart (or a Kubernetes rollout of a cache-enabled service), every key is a miss. The database takes the full production load cold. Pre-warm the cache as part of the deploy, or roll out gradually so only a fraction of traffic hits an empty cache at any moment.

### 5.3 Cache Inconsistency Across Services

If two services each cache the same entity with different invalidation logic, they will drift. Prefer **one owner per cache key** — one service writes, others read — or push invalidation through a shared message bus so all caches receive the “user 42 changed” event at once.

### 5.4 Memory Pressure and Eviction

Redis has a maximum memory setting. When it fills up, the `maxmemory-policy` decides what to evict. For a cache, set it to `allkeys-lru` (evict the least recently used key). For a session store, use `volatile-ttl` (evict keys closest to expiry). Never use the default (`noeviction`) — it causes writes to fail when memory fills.

```
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

------------------------------------------------------------------------

## Section 6: Operational Notes

- **Connection pooling.** Never open a new TCP connection per request. Use a pool (`redis.ConnectionPool` in Python) to amortize handshake cost.

- **Pipeline multi-key operations.** One round-trip instead of ten when you need to fetch ten keys.

- **Persistence is optional.** For a pure cache, disable AOF/RDB — you’re trading durability you don’t need for faster writes and lower ops burden.

- **Monitor hit rate.** A cache with a \<80% hit rate is providing less value than it costs. Check with `INFO`` ``stats` and Grafana.

- **Avoid `KEYS`` ``*` in production.** It’s O(n) over the entire keyspace and blocks the single Redis thread. Use `SCAN` instead.

------------------------------------------------------------------------

## Section 7: TTL as Cost Optimization

Redis stores data in **RAM**, which is significantly more expensive than disk storage (SSD/HDD). TTL is not just about data freshness; it is a critical strategy for **cost optimization**.

If you cache everything without expiration:

1.  **Memory leak.** Redis will eventually run out of RAM.

2.  **High expense.** Scaling Redis to 100 GB+ of RAM is very costly compared to 100 GB of database disk.

3.  **Connection saturation.** Old, unused keys clog the system, potentially slowing connections for active users.

4.  **Cloud costs (AWS ElastiCache, Azure Cache, GCP Memorystore).** Managed services charge by the **hour/node**. If you don’t expire data, you are forced to upgrade to larger instances (vertical scaling) just to store stale data, significantly increasing your hourly bill.

### The “Lease” Mental Model

Think of caching as *leasing* memory space. You do not own it forever; you rent it for a specific purpose.

- **Short lease (seconds/minutes)** — volatile data that changes fast or is only needed instantly: real-time analytics, user session active state.

- **Medium lease (hours)** — user-specific content that might be viewed multiple times in a session but isn’t permanent: conversation history.

- **Explicit TTLs help Redis delete the *right* data** (expired) instead of randomly evicting useful data when memory fills.

------------------------------------------------------------------------

## Section 8: Redis for RAG and LLM Applications

In the era of GenAI, Redis is critical for reducing LLM costs and latency.

### 8.1 Caching Chat History (Conversation Memory)

LLMs are stateless. To have a multi-turn conversation, you must send the entire chat history with every prompt. Fetching this from Postgres every time is inefficient.

**Strategy:** store the rolling context window in a Redis **list**.

- Key: `chat:{conversation_id}`

- Value: list of JSON-encoded messages `[{"role":`` ``"user",`` ``"content":`` ``"..."},`` ``...]`

- Optimization: use `LTRIM` to keep only the last N messages (typically 20).

``` python
def add_message(conv_id: str, role: str, content: str) -> None:
    key = f"chat:{conv_id}"
    msg = json.dumps({"role": role, "content": content})

    pipe = redis_client.pipeline()
    pipe.rpush(key, msg)
    pipe.ltrim(key, -20, -1)   # keep only last 20 items
    pipe.expire(key, 86400)    # 24h TTL
    pipe.execute()
```

### 8.2 Semantic Caching (Embedding-based)

Users often ask similar questions (“Reset password?” vs “How to change password?”). Standard cache misses these because the strings are different.

**Strategy:**

1.  Vectorize the user query (embedding).

2.  Search a Redis vector index for **similar** past queries (cosine similarity \> 0.95).

3.  If found, return the cached answer. This saves an expensive LLM call.

Redis Stack (with RediSearch + RedisJSON) supports vector similarity search natively, so you can run both your chat history cache and your semantic cache on the same instance.

### 8.3 When the LLM Workload Is Heavy

Chat history caching alone can cut LLM token costs by 30-50% in long conversations by keeping the hot context in Redis instead of re-fetching from Postgres on every turn. Semantic caching on top of that can deflect 10-20% of requests entirely, cutting LLM API spend proportionally.

------------------------------------------------------------------------

## Summary

Caching is the highest-leverage performance tool in most microservice architectures, and Redis is the incumbent default for a reason: it is fast, its primitives are rich, and its operational story is well understood. The hard problems are not the commands — they are deciding *what* to cache, picking the right invalidation strategy, and surviving cache stampedes under load. Start with Cache-Aside and TTL-based invalidation, measure hit rate, and add sophistication only when a metric tells you to.

For LLM-backed services, Redis is also the go-to layer for conversation memory and semantic caching — both of which turn directly into reduced LLM API spend.

## Practice

### 🌐 Base URL

```
https://yourapp.com/backend-api/
```

### 1. Redis Caching Strategy

The goal of this practice is to integrate Redis caching into the chatbot API to improve performance and reduce database load.

#### Target Endpoints to Cache

| Endpoint | Cache Strategy | Cache Key | TTL |
|----|----|----|----|
| **Get Conversation Detail** | **Cache-Aside** | `conv:{id}:history` | 10 min |
| **List Conversations** | **Cache-Aside** | `user:{id}:conversation_list` | 10 min |

#### 1.1 Cache Conversation History

> **GET** `/backend-api/conversation/{conversation_id}` Retrieve message history. Try to get from Redis first; if missing, fetch from DB and set to Redis.

**Implementation logic:**

1.  **Check cache:** `GET`` ``conv:{conversation_id}:history`

2.  **Cache hit:** return JSON immediately (latency \< 5 ms).

3.  **Cache miss:**

    - Query the database for messages.

    - **Write to cache:** `SETEX`` ``conv:{conversation_id}:history`` ``600`` ``<json_data>`

      - *Note:* 600s TTL. After 10 minutes, the entry is deleted to free RAM. On cloud providers (e.g., AWS ElastiCache), saving RAM prevents the need to scale up to expensive larger nodes.

    - Return data.

**Cache key structure:**

```
Key: conv:690d5b6c-02d8-8321-a91e-65ea55b781f7:history
Value: [JSON String of Messages]
TTL: 600 seconds
```

#### 1.2 Cache User Conversation List

> **GET** `/backend-api/conversations?offset=0&limit=20` List all conversations. This query can be heavy on the DB if the user has many chats.

**Implementation logic:**

1.  **Check cache:** `GET`` ``user:{user_id}:conversation_list`

2.  **Cache hit:** return cached list.

3.  **Cache miss:**

    - Query database (`SELECT`` ``*`` ``FROM`` ``conversations`` ``WHERE`` ``user_id`` ``=`` ``...`).

    - **Write to cache:** `SETEX`` ``user:{user_id}:conversation_list`` ``600`` ``<json_data>`

    - Return data.

4.  **Invalidation strategy:**

    - When a **new** conversation is created (`POST`` ``/backend-api/f/conversation`), **delete** this cache key so the list updates immediately.

    - Command: `DEL`` ``user:{user_id}:conversation_list`

### 2. Advanced: Caching for LLM Context

> **POST** `/backend-api/f/conversation` Chatting with AI requires sending previous context.

**Optimization challenge:** instead of fetching the full history from PostgreSQL for *every* message sent:

1.  **Store context in a Redis list:** use `RPUSH` to append new user/assistant messages to `chat:{id}:context`.

2.  **Limit context window:** use `LTRIM` to keep only the last 20 messages. This ensures you don’t exceed the LLM’s token limit and keeps Redis memory usage low.

``` python
# Context caching
def add_message_to_context(conversation_id: str, message: dict) -> None:
    key = f"chat:{conversation_id}:context"
    redis.rpush(key, json.dumps(message))
    redis.ltrim(key, -20, -1)   # keep last 20
    redis.expire(key, 86400)    # expires in 24h
```

### 3. Request Flow with Caching

1.  **Frontend** sends `GET`` ``/conversation/{id}`.

2.  **Backend** calls `CacheService.get(f"conv:{id}:history")`.

    - *Found?* Return immediately.

    - *Not found?* Fetch DB → `CacheService.set(...,`` ``ttl=600)` → return.

3.  **Frontend** sends `POST`` ``/conversation` (new message).

    - **Backend** saves message to DB.

    - **Backend** invalidates cache: `CacheService.delete(f"conv:{id}:history")`.

    - **Backend** updates Redis list context (optional).

## Review Questions

1.  **What is the primary difference between Redis and a traditional relational database like PostgreSQL?**

    - A. Redis supports SQL queries, while PostgreSQL does not.

    - B. Redis stores data on disk by default, while PostgreSQL stores data in RAM.

    - C. Redis stores data in-memory (RAM) for high speed, while PostgreSQL primarily stores data on disk.

    - D. Redis is slower but more reliable than PostgreSQL.

2.  **Which Redis data structure is best suited for implementing a “Leaderboard” or “Ranking” system?**

    - A. Strings

    - B. Sets

    - C. Sorted Sets (ZSets)

    - D. Lists

3.  **In the “Cache-Aside” pattern, what happens when a “Cache Miss” occurs?**

    - A. The application returns an error to the user immediately.

    - B. The cache automatically fetches data from the database in the background.

    - C. The application queries the database, writes the result to the cache with a TTL, and then returns the data.

    - D. The application waits for the cache to be manually updated by an admin.

4.  **Why is it critical to set a “Time To Live” (TTL) for cached data?**

    - A. To ensure the data is encrypted.

    - B. To prevent the cache from filling up with old/stale data and causing memory leaks or high costs.

    - C. To make the data load faster.

    - D. To allow multiple users to access the same key.

5.  **What is the main benefit of “Redis Serverless” compared to provisioning fixed-size nodes?**

    - A. It is always free.

    - B. You pay based on actual usage (storage + requests) rather than idle instance time.

    - C. It runs on your own local computer.

    - D. It does not support TTLs.

6.  **When caching “Chat History” for an LLM application, why might we use the `LTRIM` command?**

    - A. To delete the entire conversation.

    - B. To keep only the most recent N messages (context window) and remove older ones to save memory/tokens.

    - C. To encrypt the messages.

    - D. To translate the messages into another language.

7.  **How does “Connection Pooling” optimize Redis performance?**

    - A. It creates a new connection for every single request to ensure freshness.

    - B. It reuses a set of established connections, reducing the overhead of constantly opening and closing handshakes.

    - C. It allows unlimited connections to the server.

    - D. It disconnects users who are idle for too long.

8.  **What is a “Cache Hit”?**

    - A. When the requested data is NOT found in the cache.

    - B. When the cache server crashes.

    - C. When the requested data IS found in the cache and returned immediately.

    - D. When the user manually clears their browser cache.

9.  **Which scenario is a good candidate for a “Long TTL” (e.g., hours or days)?**

    - A. Real-time stock prices.

    - B. User session active status.

    - C. Reference data like “Country Codes” or “Product Categories” that rarely change.

    - D. The exact location of a delivery driver.

10. **What is the risk of the “Write-Behind” caching strategy?**

    - A. It is too slow for real-time applications.

    - B. If the cache crashes before the data is asynchronously synced to the database, data loss can occur.

    - C. It blocks the main thread.

    - D. It requires more RAM than Write-Through.

11. **In a RAG application, what does “Semantic Caching” store?**

    - A. The exact string match of the user’s question.

    - B. The vector embedding of the query, allowing retrieval of answers to *similar* (not just identical) questions.

    - C. The raw PDF documents.

    - D. The user’s password.

12. **For AWS ElastiCache, why does failing to use TTLs result in higher costs?**

    - A. AWS charges extra for keys without TTLs.

    - B. You are charged for every `GET` request.

    - C. Stale data consumes RAM, forcing you to vertically scale to larger, more expensive node types (e.g., moving from `cache.t3.micro` to `cache.r6g.large`).

    - D. It slows down the internet connection.

**View Answer Key**

1.  **C** — Speed vs. persistence trade-off.

2.  **C** — ZSets store a score with each member, perfect for ranking.

3.  **C** — The application “lazily” loads data only when needed.

4.  **B** — Memory is expensive; cleaning up old data is essential.

5.  **B** — Serverless scales down to (near) zero cost when idle.

6.  **B** — Optimizes for the limited context window of LLMs.

7.  **B** — Handshakes are expensive; reuse is efficient.

8.  **C** — The happy path!

9.  **C** — Static data doesn’t degrade quickly.

10. **B** — Durability is compromised for write speed.

11. **B** — Semantic = meaning. Vector search finds similar meanings.

12. **C** — You pay for provisioned capacity (RAM/CPU), not just data size.
