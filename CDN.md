# Content Delivery Network (CDN)

## What is a CDN?

A **Content Delivery Network (CDN)** is a geographically distributed network of servers that caches and delivers content from a location close to the user.

Instead of every user requesting content from a single origin server:

```text
User ───────────────→ Origin Server
```

a CDN places **edge servers** around the world:

```text
                    ┌── Edge Server
                    │
User ──→ CDN ───────┼── Edge Server
                    │
                    └── Edge Server
                          │
                          ↓
                    Origin Server
```

The goal is to reduce **latency**, improve **availability**, and reduce load on the origin server.

---

# Why Do We Need a CDN?

Suppose your application is hosted in Mumbai and a user is in New York.

Without a CDN:

```text
New York User
      │
      │ Long distance
      ↓
Mumbai Server
```

Every request has to travel a long distance.

With a CDN:

```text
New York User
      │
      ↓
New York Edge Server
      │
      ↓
Origin Server
```

If the requested content is already cached at the edge server, the origin server doesn't need to be contacted at all.

---

# CDN Architecture

A simplified CDN looks like this:

```text
                         Origin Server
                              │
                              ↓
                    ┌──────────────────┐
                    │       CDN        │
                    │                  │
                    │ Edge Locations   │
                    └──────────────────┘
                      ↙       ↓       ↘
                    Edge     Edge     Edge
                   Mumbai   London   New York
                     ↑        ↑        ↑
                     │        │        │
                   Users    Users    Users
```

### Origin Server

The origin server is the source of truth for the content.

It might be:

* Application server
* Object storage
* Web server
* API server

### Edge Server

An edge server is a CDN server located close to users.

It caches content from the origin and serves subsequent requests locally.

### Edge Location / Point of Presence

A **PoP (Point of Presence)** is a physical location containing CDN infrastructure.

Large CDNs have many PoPs around the world.

---

# How Does a CDN Request Work?

Suppose a user requests:

```text
GET /images/product.jpg
```

### Step 1 — User Sends Request

```text
User
 ↓
CDN
```

The CDN determines an appropriate edge location.

### Step 2 — Check Edge Cache

The edge server checks whether the content exists in its cache.

```text
Cache?
 /   \
Yes   No
 ↓     ↓
Return  Origin
        ↓
      Content
        ↓
      Cache
        ↓
      Return
```

---

# Cache Hit

If the content exists:

```text
User
 ↓
CDN Edge
 ↓
Cache HIT
 ↓
Content
```

The request never reaches the origin.

This provides:

* Lower latency
* Lower origin load
* Higher throughput

---

# Cache Miss

If the content doesn't exist:

```text
User
 ↓
CDN Edge
 ↓
Cache MISS
 ↓
Origin Server
 ↓
Content
 ↓
CDN Edge caches it
 ↓
User
```

Future users requesting the same resource can receive it directly from the edge.

---

# CDN and Caching

A CDN is essentially a **large geographically distributed caching system**.

You can think of it as:

```text
Traditional Cache:

Application → Redis → Database


CDN:

User → Edge Cache → Origin
```

The major difference is **where the cache lives and what it caches**.

A Redis cache is typically used for application data.

A CDN is primarily used to cache and deliver content close to users.

---

# What Does a CDN Cache?

CDNs are particularly effective for content that is:

* Frequently requested
* Relatively static
* Expensive to fetch from the origin

Examples:

```text
Images
Videos
CSS
JavaScript
Fonts
HTML
Software downloads
Static API responses
```

For example:

```text
/images/logo.png
/videos/movie.mp4
/css/style.css
/js/app.js
```

---

# Dynamic Content

CDNs aren't limited to completely static content.

Modern CDNs can also help with dynamic content and APIs using techniques such as:

* Dynamic caching
* Edge computing
* API caching
* Request routing
* Compression
* Connection optimization

However, highly personalized or rapidly changing data is generally harder to cache.

For example:

```text
GET /user/123/profile
```

may be personalized and therefore require more careful caching rules.

---

# CDN Request Routing

A major challenge is:

> Which edge server should handle the user's request?

CDNs use techniques such as:

* DNS-based routing
* Anycast
* Geographic routing
* Latency-based routing
* Load balancing

The goal is generally to send the user to a suitable nearby/healthy edge location.

---

# DNS-Based CDN Routing

A simplified flow:

```text
User
 ↓
DNS Resolver
 ↓
CDN DNS
 ↓
Choose Edge Location
 ↓
Edge Server
```

For example:

```text
User in India
      ↓
CDN DNS
      ↓
Mumbai Edge

User in USA
      ↓
CDN DNS
      ↓
New York Edge
```

This allows users to connect to a nearby CDN location.

---

# Cache-Control

HTTP headers are important for controlling CDN caching behavior.

For example:

```http
Cache-Control: max-age=3600
```

This tells caches that the response can be considered fresh for a specified period.

Other useful concepts include:

```text
max-age
s-maxage
no-cache
no-store
public
private
```

### Example

For a static image:

```http
Cache-Control: public, max-age=86400
```

The CDN can cache the resource for approximately one day.

---

# CDN Cache Invalidation

One of the biggest challenges is updating cached content.

Suppose:

```text
Origin:
logo.png → Version 2
```

But the CDN still has:

```text
Edge Cache:
logo.png → Version 1
```

Users may receive the old version.

There are several approaches.

## TTL-Based Expiration

Wait for the cached object to expire.

```text
Cache
 ↓
TTL expires
 ↓
Next request → Origin
```

## Explicit Purging

Tell the CDN to remove cached content.

```text
Purge /images/logo.png
```

The next request will fetch the latest version.

## Cache Busting

Change the URL whenever content changes.

```text
app.v1.js
app.v2.js
```

or:

```text
app.js?v=2
```

This is extremely common for static assets.

---

# CDN Cache Hierarchy

Large CDNs may have multiple cache layers.

For example:

```text
User
 ↓
Edge Cache
 ↓
Regional Cache
 ↓
Origin
```

If the edge doesn't have the object, it can potentially retrieve it from a higher-level cache instead of directly contacting the origin.

This reduces origin traffic.

---

# CDN for Video Streaming

CDNs are especially important for video platforms.

Without a CDN:

```text
Millions of users
       ↓
Single Origin
       ↓
Huge bandwidth requirement
```

With a CDN:

```text
                  Origin
                 /  |  \
                /   |   \
             Edge  Edge  Edge
              ↑     ↑     ↑
           Users  Users  Users
```

Popular video segments can be cached at edge locations.

This significantly reduces the load on the origin.

---

# CDN and Availability

CDNs can also improve availability.

Suppose one edge location fails:

```text
Mumbai Edge ❌
```

Traffic can potentially be routed to another healthy edge:

```text
Mumbai Edge ❌
      ↓
Another Edge
      ↓
User
```

This provides an additional layer of resilience.

However, a CDN does **not** automatically make the entire application highly available. The origin and other infrastructure still need proper redundancy.

---

# CDN vs Load Balancer

These are often confused.

### CDN

Primarily focuses on:

* Content delivery
* Caching
* Geographic distribution
* Reducing latency

```text
User → CDN → Origin
```

### Load Balancer

Primarily focuses on distributing traffic across application servers.

```text
Client
  ↓
Load Balancer
 /    |    \
App1 App2 App3
```

They are often used together:

```text
                    ┌→ App Server 1
User → CDN → LB ────┼→ App Server 2
                    └→ App Server 3
```

---

# CDN vs Redis Cache

| Feature                 | CDN                                 | Redis                               |
| ----------------------- | ----------------------------------- | ----------------------------------- |
| Main purpose            | Content delivery                    | Application data caching            |
| Location                | Globally distributed edge locations | Usually centralized/clustered       |
| Main users              | End users                           | Application servers                 |
| Typical data            | Images, JS, CSS, video              | Sessions, objects, counters         |
| Geographic distribution | Very high                           | Depends on architecture             |
| Primary goal            | Reduce network latency              | Reduce database/application latency |

---

# CDN Trade-offs

## Advantages

### 1. Lower Latency

Users receive content from a nearby edge.

### 2. Reduced Origin Load

Cached requests don't reach the origin.

### 3. Higher Scalability

CDNs can handle huge amounts of traffic.

### 4. Better Availability

Traffic can potentially be served from alternative edge locations.

### 5. Reduced Bandwidth Costs

Less traffic reaches the origin infrastructure.

---

## Disadvantages

### 1. Cache Invalidation Complexity

Updating cached content can be difficult.

### 2. Stale Data

Users may temporarily receive outdated content.

### 3. Additional Complexity

You now need to manage:

* Cache rules
* TTLs
* Purging
* Headers
* Routing

### 4. Not Everything Can Be Cached

Personalized or rapidly changing data may not benefit from CDN caching.

---

# CDN in a System Design

A typical web architecture might look like:

```text
                         ┌───────────────┐
                         │   Database    │
                         └───────┬───────┘
                                 │
                                 ↓
                         ┌───────────────┐
                         │ Application   │
                         │   Servers     │
                         └───────┬───────┘
                                 │
                                 ↓
User → DNS → CDN → Load Balancer
                    │
                    ├── App Server 1
                    ├── App Server 2
                    └── App Server 3
```

For static content:

```text
User → CDN → Cache HIT → Response
```

For uncached content:

```text
User → CDN → Cache MISS → Load Balancer → App Server
```

---

# CDN + Caching + Consistent Hashing

These three concepts connect nicely:

```text
              CDN
               ↓
        Distributed Cache
               ↓
      Consistent Hashing
               ↓
       Cache Node Selection
```

For example, a CDN may distribute content across many edge servers, while a distributed cache can use consistent hashing to decide which cache node stores a particular object.

---

# Key Metrics

When evaluating a CDN, monitor:

### Cache Hit Ratio

Percentage of requests served directly from the CDN cache.

### Cache Miss Ratio

Percentage of requests that require fetching from the origin.

### Edge Latency

Time taken to serve a request from the edge.

### Origin Load

How much traffic reaches the origin.

### Bandwidth

Amount of data transferred.

### Error Rate

Percentage of failed requests.

---

# Interview Cheat Sheet

### What is a CDN?

A geographically distributed network of servers that caches and delivers content close to users.

### Why use a CDN?

To reduce latency, reduce origin load, improve scalability, and improve availability.

### What happens during a cache miss?

The CDN retrieves the content from the origin, returns it to the user, and can cache it for subsequent requests.

### What is cache invalidation?

Removing or updating stale content stored at CDN edge locations.

### CDN vs Load Balancer?

```text
CDN → Distributes/caches content geographically

Load Balancer → Distributes application traffic across servers
```

### What determines which CDN server handles a request?

Typically DNS, Anycast, geographic routing, latency, and health/load information.

---

# Mental Model

Think of a CDN as:

```text
                 ┌─────────────┐
                 │   ORIGIN    │
                 └──────┬──────┘
                        │
              Distribute content
                        │
       ┌────────────────┼────────────────┐
       ↓                ↓                ↓
   Edge India       Edge Europe      Edge USA
       ↑                ↑                ↑
       │                │                │
    Users            Users            Users
```

The core idea:

> **Move frequently requested content closer to users so it can be served faster without repeatedly hitting the origin server.**
