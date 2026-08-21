# Caching in Distributed Systems

## What is Caching?

Caching is the process of storing frequently accessed data in a **faster storage layer** so that future requests can be served more quickly.

Instead of repeatedly querying the database:

```text
Client
  ↓
Application
  ↓
Database
```

we introduce a cache:

```text
Client
  ↓
Application
  ↓
Cache
  ↓
Database
```

The cache is usually much faster than the primary database.

Common caching systems include:

* Redis
* Memcached
* CDN caches
* Application-level caches
* Browser caches

---

# Why Do We Need Caching?

Consider an application receiving:

```text
10,000 requests/sec
```

If every request queries the database:

```text
10,000 requests
       ↓
   Database
```

The database can become the bottleneck.

With caching:

```text
10,000 requests
       ↓
      Cache
       ↓
   Database
```

If 90% of requests are cache hits, only around 1,000 requests need to reach the database.

### Benefits

* Lower latency
* Reduced database load
* Higher throughput
* Better scalability
* Lower infrastructure cost

---

# Cache Hit vs Cache Miss

These are the two most important concepts.

## Cache Hit

The requested data exists in the cache.

```text
Client
  ↓
Application
  ↓
Cache ──→ Data found
  ↓
Response
```

The database is not contacted.

## Cache Miss

The requested data is not present in the cache.

```text
Client
  ↓
Application
  ↓
Cache ──→ Data not found
  ↓
Database
  ↓
Cache
  ↓
Response
```

The application fetches the data from the database and usually stores it in the cache for future requests.

### Cache Hit Ratio

A useful metric is:

```text
Cache Hit Ratio = Cache Hits / Total Requests
```

For example:

```text
9,000 cache hits
1,000 cache misses

Hit Ratio = 90%
```

A higher hit ratio generally means the cache is doing its job effectively.

---

# Where Can We Cache?

Caching can happen at different layers.

```text
Client
  ↓
Browser Cache
  ↓
CDN
  ↓
Application
  ↓
Distributed Cache
  ↓
Database
```

## 1. Browser Cache

Data is cached on the user's device.

Examples:

* Images
* CSS
* JavaScript
* Static assets

## 2. CDN Cache

A CDN stores content close to users geographically.

```text
User → CDN Edge → Origin Server
```

Useful for:

* Images
* Videos
* Static files
* Web pages

## 3. Application Cache

The application itself stores frequently accessed data.

## 4. Distributed Cache

A separate caching cluster is used by multiple application servers.

Examples:

```text
Redis
Memcached
```

---

# Distributed Cache

Suppose we have multiple application servers:

```text
             Load Balancer
              /    |    \
             ↓     ↓     ↓
           App1   App2   App3
             \     |     /
              \    |    /
               ↓   ↓   ↓
             Redis Cluster
                   ↓
               Database
```

All application servers can access the same distributed cache.

This is generally preferable to keeping important cached state only inside individual application servers.

---

# Cache-Aside Pattern

The most common caching pattern is **Cache-Aside**.

The application is responsible for reading from and writing to the cache.

### Read

```text
1. Check cache
       ↓
2. Cache hit?
    /       \
  Yes        No
   ↓          ↓
Return      Database
data          ↓
           Update cache
               ↓
           Return data
```

Example:

```text
GET user:123

Cache → Miss

Database → User 123

Cache ← Store User 123

Return User 123
```

### Why is it popular?

Because the application has explicit control over what gets cached.

---

# Write Strategies

Caching becomes more complicated when data changes.

There are several common strategies.

## 1. Write-Through

Every write goes to the cache and database.

```text
Application
   ↓
 Cache
   ↓
Database
```

The cache is always updated when the database is updated.

### Advantage

Cache remains relatively fresh.

### Disadvantage

Every write has additional cache overhead.

---

# 2. Write-Back

The application writes to the cache first.

```text
Application
     ↓
   Cache
     ↓
  Database
  (later)
```

The database is updated asynchronously.

### Advantage

Very fast writes.

### Disadvantage

If the cache fails before data reaches the database, data can potentially be lost.

---

# 3. Write-Around

Writes go directly to the database.

```text
Application
   ↓
Database
```

The cache is populated only when data is later read.

This avoids caching data that may never be read.

---

# Cache Eviction

A cache has limited memory.

Eventually, old entries need to be removed.

Common eviction policies include:

### LRU — Least Recently Used

Remove the item that hasn't been accessed for the longest time.

```text
A → recently used
B → recently used
C → old
D → oldest

Evict D
```

Very commonly used.

### LFU — Least Frequently Used

Remove the item accessed the fewest times.

Useful when frequently accessed data should remain cached.

### FIFO — First In, First Out

Remove the item that entered the cache first.

### TTL — Time To Live

An item automatically expires after a specified duration.

```text
user:123
TTL = 300 seconds
```

After 300 seconds, the entry expires.

---

# Cache Invalidation

One of the hardest problems in caching is:

> **How do we keep cached data consistent with the database?**

Suppose:

```text
Database:
User name = "Alice"

Cache:
User name = "Alice"
```

The database is updated:

```text
Database:
User name = "Bob"
```

But the cache still contains:

```text
Cache:
User name = "Alice"
```

Now users receive stale data.

This is why **cache invalidation** is critical.

Common approaches:

### TTL-Based Invalidation

Let the cache entry expire automatically.

```text
Cache Entry
   ↓
TTL expires
   ↓
Entry removed
```

### Explicit Invalidation

When data changes:

```text
Update Database
      ↓
Delete Cache Entry
```

### Event-Based Invalidation

A database change generates an event:

```text
Database
   ↓
Event
   ↓
Message Queue
   ↓
Cache Invalidation
```

---

# Cache Stampede

A cache stampede occurs when a popular cache entry expires and many requests simultaneously try to rebuild it.

Suppose:

```text
1,000 requests/sec
       ↓
Cache MISS
       ↓
1,000 database queries
```

The database suddenly receives a huge spike.

### Solutions

* Locking
* Request coalescing
* Randomized TTLs
* Background refresh
* Early refresh
* Stale-while-revalidate

---

# Cache Penetration

Cache penetration occurs when requests repeatedly ask for data that **does not exist**.

For example:

```text
GET user:999999999
```

If the user doesn't exist:

```text
Cache → Miss
Database → Miss
```

Every request hits the database.

### Solution: Negative Caching

Cache the fact that the data doesn't exist.

```text
Cache:
user:999999999 → NULL
```

Future requests can be rejected without hitting the database.

---

# Cache Avalanche

A cache avalanche occurs when a large number of cache entries expire at approximately the same time.

```text
Cache
████████████████
       ↓
Many entries expire
       ↓
Huge DB traffic
       ↓
Database overloaded
```

### Solutions

* Randomize TTLs
* Stagger expiration
* Use background refresh
* Rate limiting
* Multiple cache layers

---

# Distributed Cache and Consistent Hashing

This connects directly to **consistent hashing**.

Suppose we have multiple Redis servers:

```text
Redis 1
Redis 2
Redis 3
Redis 4
```

We need to decide which Redis node stores:

```text
user:123
user:456
user:789
```

A common approach is hashing the key:

```text
hash(user:123)
       ↓
Redis node
```

Consistent hashing helps minimize key movement when cache nodes are added or removed.

```text
Cache Cluster

        Consistent Hash Ring
       /         |          \
    Redis 1    Redis 2    Redis 3
```

If Redis 4 is added, only a portion of the keys need to move.

---

# Cache Consistency

There is a fundamental trade-off between:

```text
Freshness
   vs
Performance
```

A cache can be:

### Strongly Consistent

Cached data closely tracks the database.

More expensive and complex.

### Eventually Consistent

Cached data may temporarily be stale.

Usually simpler and faster.

Many distributed caching systems favor eventual consistency because it provides better scalability and performance.

---

# Important Metrics

When designing a caching system, monitor:

### Cache Hit Ratio

How often requests are served from the cache.

### Cache Miss Ratio

How often requests need to access the underlying data store.

### Latency

How quickly the cache responds.

### Eviction Rate

How frequently entries are removed because of capacity or policy.

### Memory Usage

How much cache capacity is being consumed.

### Database Load

One of the main reasons for introducing caching.

---

# When Should You Use Caching?

Caching works especially well when:

* Data is read frequently
* Data doesn't change frequently
* Low latency is important
* Database queries are expensive
* The same data is requested repeatedly

Examples:

```text
User profiles
Product information
Session data
Popular posts
Configuration
API responses
Leaderboard data
```

Caching is less useful when data:

* Changes extremely frequently
* Is rarely accessed
* Must always be strongly consistent
* Is very difficult to invalidate correctly

---

# Key Trade-offs

| Benefit            | Cost                   |
| ------------------ | ---------------------- |
| Lower latency      | Stale data             |
| Less DB load       | Cache invalidation     |
| Higher throughput  | Extra infrastructure   |
| Better scalability | More system complexity |
| Faster reads       | Memory cost            |

---

# Interview Cheat Sheet

### What is caching?

Storing frequently accessed data in a faster storage layer to reduce latency and backend load.

### What is a cache hit?

Requested data exists in the cache.

### What is a cache miss?

Requested data is not in the cache and must be fetched from the underlying data store.

### What is cache-aside?

Application checks the cache first and fetches from the database on a miss.

### What is cache invalidation?

Removing or updating cached data when the underlying data changes.

### What is cache stampede?

Many requests simultaneously regenerate an expired cache entry, causing a sudden database load spike.

### LRU vs LFU?

```text
LRU → Evict least recently used
LFU → Evict least frequently used
```

### Why use consistent hashing with distributed caches?

To distribute keys across cache nodes while minimizing key movement when nodes are added or removed.

---

# Mental Model

Think of caching as:

```text
              ┌──────────────┐
Request ─────→│    CACHE     │
              └──────┬───────┘
                 Hit │ Miss
                     │
                     ↓
              ┌──────────────┐
              │   DATABASE   │
              └──────────────┘
                     │
                     ↓
              Update Cache
```

The core goal is:

> **Serve frequently accessed data from a fast layer while reducing load on slower backend systems.**
