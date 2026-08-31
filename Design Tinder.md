# Tinder — System Design

## 1. Problem Statement

Design a dating application similar to Tinder where users can:

* Create and update profiles
* Upload profile photos
* Discover nearby users
* Swipe left or right on profiles
* Match when two users like each other
* Chat with matches
* Receive notifications
* See online/activity status

The most interesting parts of the system are:

```text
User Discovery
       ↓
Recommendation / Candidate Generation
       ↓
Swiping
       ↓
Matching
       ↓
Notifications
       ↓
Real-time Chat
```

---

# 2. Functional Requirements

## Core Requirements

### User Management

Users should be able to:

* Sign up
* Log in
* Create profiles
* Update profiles
* Upload photos
* Set preferences

Example profile:

```json
{
  "userId": "123",
  "name": "Anshu",
  "age": 23,
  "gender": "male",
  "location": {
    "lat": 25.5941,
    "lon": 85.1376
  },
  "interests": [
    "gym",
    "swimming",
    "music"
  ]
}
```

---

## Discovery

Users should receive a list of potential matches based on:

* Location
* Age preferences
* Gender preferences
* Interests
* Previous swipes
* Other recommendation signals

```text
User
 ↓
Discovery Service
 ↓
Candidate Generation
 ↓
Ranking
 ↓
Profile Cards
```

---

## Swiping

A user can:

```text
Swipe Left  → Dislike
Swipe Right → Like
```

Example API:

```http
POST /users/123/swipes
```

```json
{
  "targetUserId": 456,
  "action": "LIKE"
}
```

---

## Matching

A match occurs when:

```text
User A → LIKE → User B

AND

User B → LIKE → User A
```

Then:

```text
          LIKE
A ----------------→ B
A ←---------------- B
          LIKE

             ↓

           MATCH
```

---

## Chat

Matched users should be able to exchange messages.

```text
User A
   ↕
Chat Service
   ↕
User B
```

Messages should ideally be delivered in real time.

---

## Notifications

Users should receive notifications for:

* New match
* New message
* New like
* Other important events

---

# 3. Non-Functional Requirements

The system should provide:

### Low Latency

Swipe and discovery requests should respond quickly.

### High Availability

The application should continue working even if some servers fail.

### Scalability

The system needs to support millions of users and potentially billions of swipes.

### Consistency

Matching should be handled carefully because duplicate matches or incorrect match states are undesirable.

### Real-Time Communication

Chat messages should have low delivery latency.

---

# 4. High-Level Architecture

A simplified architecture:

```text
                         ┌──────────────┐
                         │    Client    │
                         │ iOS / Android│
                         └──────┬───────┘
                                │
                                ↓
                         ┌──────────────┐
                         │ API Gateway  │
                         └──────┬───────┘
                                │
        ┌───────────────────────┼────────────────────────┐
        ↓                       ↓                        ↓
 User Service            Discovery Service         Swipe Service
        │                       │                        │
        ↓                       ↓                        ↓
 User DB                 Recommendation             Swipe DB
                         Service
                                │
                                ↓
                           Cache / DB

                                ↓
                         Matching Service
                                │
                                ↓
                           Match DB

                                ↓
                          Message Broker
                         /      |       \
                        ↓       ↓        ↓
                 Notification  Analytics  Other Services
                        │
                        ↓
                   Push Service
```

Chat can have its own real-time infrastructure:

```text
Client
  ↕
WebSocket Gateway
  ↕
Chat Service
  ↓
Message Store
```

---

# 5. API Design

## Get Discovery Feed

```http
GET /feed
```

Query parameters:

```text
?limit=20
&cursor=abc123
```

Response:

```json
{
  "profiles": [
    {
      "userId": "456",
      "name": "User A",
      "age": 24,
      "distance": 3.2,
      "photos": [
        "photo_url"
      ]
    }
  ],
  "nextCursor": "xyz789"
}
```

Cursor-based pagination is useful because the discovery dataset can be very large.

---

## Swipe API

```http
POST /swipes
```

Request:

```json
{
  "targetUserId": "456",
  "action": "LIKE"
}
```

Response:

```json
{
  "matched": true,
  "matchId": "match_123"
}
```

---

## Get Matches

```http
GET /matches
```

Response:

```json
{
  "matches": [
    {
      "matchId": "match_123",
      "userId": "456",
      "createdAt": "2026-08-31T12:00:00Z"
    }
  ]
}
```

---

## Send Message

For real-time chat, WebSockets can be used.

```text
WebSocket connection
        ↓
send_message
        ↓
Chat Service
```

---

# 6. User Service

The User Service manages:

```text
User profile
Photos
Preferences
Location
Interests
Account information
```

Example:

```text
User Service
     |
     ↓
User Database
```

A relational database can work well here because user profile data has a relatively structured model.

---

# 7. Profile Photos

Photos are large binary objects and should not normally be stored directly inside the database.

Instead:

```text
Client
 ↓
Photo Service
 ↓
Object Storage
 ↓
CDN
 ↓
Users
```

Example architecture:

```text
                   Object Storage
                         ↑
                         |
Client → Upload Service
                         |
                         ↓
                       CDN
                         ↓
                       Users
```

The database stores metadata:

```json
{
  "userId": 123,
  "photoId": 456,
  "url": "https://cdn.example.com/photo/456"
}
```

The actual image lives in object storage.

The CDN then serves the image from an edge location close to the user.

---

# 8. The Most Important Problem — User Discovery

The system cannot simply query:

```text
SELECT *
FROM users
WHERE age BETWEEN 20 AND 25;
```

There could be millions of users.

We need to efficiently find a smaller set of **candidate users**.

The discovery pipeline can be:

```text
User
 ↓
Candidate Generation
 ↓
Filtering
 ↓
Ranking
 ↓
Feed
```

---

# 9. Candidate Generation

Candidate generation might consider:

```text
Location
Age
Gender
Preferences
Interests
Activity
Previous interactions
```

For example:

```text
User A

Location: Patna
Age: 23
Preference: 21-26

        ↓

Find users:

Age: 21-26
Location: nearby
Not already swiped
Matching preferences
```

---

# 10. Geospatial Search

Location is one of the most important aspects of Tinder.

We don't want:

```text
User in Patna
      ↓
Search entire world
```

We want:

```text
User
 ↓
Nearby users
 ↓
Candidate pool
```

A common technique is **geospatial indexing**.

The world can be divided into geographic cells.

For example:

```text
+-----+-----+-----+
| A1  | A2  | A3  |
+-----+-----+-----+
| B1  | B2  | B3  |
+-----+-----+-----+
| C1  | C2  | C3  |
+-----+-----+-----+
```

A user belongs to a particular cell.

To find nearby users:

```text
User
 ↓
Current cell
 ↓
Neighboring cells
 ↓
Candidate users
```

Technologies/databases with geospatial capabilities can help implement this.

---

# 11. Avoiding Previously Seen Profiles

Suppose a user has already swiped on:

```text
User 101
User 102
User 103
...
```

We don't want to show these profiles again.

We can maintain a record:

```text
(userId, targetUserId) → action
```

For example:

```text
(123, 456) → LIKE
(123, 789) → DISLIKE
```

This can be stored in a high-write datastore.

---

# 12. Swipe Service

The Swipe Service handles:

```text
LIKE
DISLIKE
SUPER_LIKE
```

Example:

```text
Client
 ↓
API Gateway
 ↓
Swipe Service
 ↓
Swipe Database
```

A swipe can generate an event:

```text
SWIPE_CREATED
```

```text
Swipe Service
      ↓
SWIPE_CREATED
      ↓
Message Broker
```

This allows other services to react asynchronously.

---

# 13. Matching

Matching is the most interesting part.

Suppose:

```text
A likes B
```

We store:

```text
(A → B) = LIKE
```

When B likes A:

```text
B → A = LIKE
```

Now:

```text
A → B = LIKE
B → A = LIKE
```

Therefore:

```text
MATCH CREATED
```

---

# 14. Match Detection

There are two broad approaches.

## Synchronous Matching

When the user swipes:

```text
Swipe
 ↓
Store swipe
 ↓
Check reverse swipe
 ↓
Match?
```

For example:

```text
A likes B
   ↓
Check:
Does B like A?
   ↓
YES
   ↓
Create Match
```

This gives the user an immediate match response.

---

## Asynchronous Matching

The swipe generates an event:

```text
SWIPE_CREATED
       ↓
Message Broker
       ↓
Matching Service
       ↓
Check reverse swipe
       ↓
Create Match
```

This is more decoupled but may introduce slight delay.

---

# 15. Preventing Duplicate Matches

Imagine two requests arrive simultaneously:

```text
A likes B
B likes A
```

Both requests might attempt to create a match.

We need to guarantee:

```text
A + B → Only ONE Match
```

A common solution is a unique constraint on the pair.

For example, normalize the pair:

```text
min(userA, userB)
max(userA, userB)
```

So:

```text
A = 100
B = 200

match key = (100, 200)
```

Whether the request comes as:

```text
100 → 200
```

or:

```text
200 → 100
```

the same unique key is used.

This prevents duplicate matches.

---

# 16. Event-Driven Matching

A good architecture can combine synchronous and asynchronous processing.

```text
Swipe Request
     ↓
Swipe Service
     ↓
Store Swipe
     ↓
Check Match
     ↓
Return Result

     ↓
SWIPE_CREATED event
     ↓
Message Broker
     ├──→ Analytics
     ├──→ Recommendation
     └──→ Other consumers
```

This keeps the user-facing operation fast while allowing background systems to process the event.

---

# 17. Recommendation System

The discovery feed should not simply return random nearby users.

A ranking system can assign a score:

```text
Score =
    Location Score
  + Preference Score
  + Interest Score
  + Activity Score
  + Recommendation Score
```

Conceptually:

```text
Candidate Pool
      ↓
Feature Generation
      ↓
Ranking Model
      ↓
Top N Profiles
      ↓
User Feed
```

The recommendation system can learn from:

```text
Likes
Dislikes
Matches
Messages
Profile interactions
```

---

# 18. Caching

Discovery is read-heavy.

The system can cache candidate profiles.

```text
Client
 ↓
Discovery Service
 ↓
Cache
 ↓
Database
```

For example:

```text
feed:user:123
```

could contain a precomputed list of candidate IDs.

```text
feed:user:123
→ [456, 789, 321, 555, ...]
```

This reduces expensive candidate-generation work.

---

# 19. Precomputing Feeds

Instead of generating the feed every time the user opens the app:

```text
User opens app
       ↓
Generate feed
       ↓
Return
```

we can generate candidates asynchronously:

```text
Recommendation Service
       ↓
Generate candidates
       ↓
Store candidate IDs
       ↓
Cache
```

Then:

```text
User opens app
       ↓
Fetch precomputed feed
       ↓
Very fast response
```

This is a common technique for reducing latency.

---

# 20. Message Queue / Event Streaming

Events can flow through a broker:

```text
Swipe
  ↓
SWIPE_CREATED
  ↓
Message Broker
  ├──→ Matching
  ├──→ Recommendation
  ├──→ Analytics
  └──→ Notification
```

For example:

```text
LIKE_RECEIVED
```

could trigger:

```text
Notification Service
       ↓
Push notification
```

---

# 21. Notifications

When a match occurs:

```text
Match Service
     ↓
MATCH_CREATED
     ↓
Message Broker
     ↓
Notification Service
     ↓
Push Notification
     ↓
User
```

The notification doesn't need to block the matching request.

This is a good use case for asynchronous processing.

---

# 22. Real-Time Chat

Once users match, they can chat.

HTTP polling would be inefficient:

```text
Client → "Any new messages?"
Client → "Any new messages?"
Client → "Any new messages?"
```

Instead, use a persistent WebSocket connection.

```text
User A
  ↕
WebSocket
  ↕
Chat Gateway
  ↕
Chat Service
  ↕
Message Store
```

When A sends:

```text
Hello!
```

the Chat Service can deliver it to B over their WebSocket connection.

---

# 23. WebSocket Scaling

Suppose we have multiple WebSocket servers:

```text
             Load Balancer
              /    |    \
             ↓     ↓     ↓
           WS1    WS2    WS3
```

User A may be connected to:

```text
WS1
```

while User B is connected to:

```text
WS3
```

How does WS1 deliver a message to WS3?

A shared messaging system can be used:

```text
WS1
 ↓
Redis Pub/Sub / Message Broker
 ↓
WS3
 ↓
User B
```

So:

```text
User A
  ↓
WS1
  ↓
Message Broker
  ↓
WS3
  ↓
User B
```

---

# 24. Message Storage

Messages need persistent storage.

A message could look like:

```json
{
  "messageId": "msg123",
  "conversationId": "conv456",
  "senderId": "123",
  "receiverId": "456",
  "text": "Hello!",
  "timestamp": "2026-08-31T12:30:00Z"
}
```

The access pattern is usually:

```text
Get messages for conversation
ordered by timestamp
```

A database optimized for high-volume writes and predictable key-based queries can be useful here.

---

# 25. Database Architecture

We don't necessarily need one database for everything.

A possible architecture:

```text
User Service
     ↓
SQL Database

Swipe Service
     ↓
Distributed NoSQL DB

Match Service
     ↓
SQL / NoSQL

Chat Service
     ↓
Distributed Message Store

Cache
     ↓
Redis

Photos
     ↓
Object Storage
```

This is an example of **polyglot persistence**.

Different workloads use different storage technologies.

---

# 26. Sharding

At Tinder-scale, a single database may not be enough.

We can shard data based on `userId`.

```text
hash(userId)
      ↓
Shard
```

For example:

```text
Users 1–1M     → Shard 1
Users 1M–2M    → Shard 2
Users 2M–3M    → Shard 3
```

Consistent hashing can also be used when distributing keys across nodes.

---

# 27. Why Consistent Hashing?

Suppose swipe data is distributed across:

```text
Node 1
Node 2
Node 3
```

A hash function determines where a user's data lives.

```text
hash(userId)
      ↓
Node
```

When adding a node, traditional:

```text
hash(key) % N
```

can cause many keys to move.

Consistent hashing minimizes the amount of data that needs to be remapped.

This is particularly useful in distributed caches and distributed storage systems.

---

# 28. CDN

Profile photos are excellent candidates for CDN caching.

```text
User
 ↓
CDN Edge
 ↓
Photo Cache
```

If the photo isn't cached:

```text
CDN
 ↓
Object Storage
 ↓
CDN
 ↓
User
```

This prevents every photo request from reaching the origin storage.

---

# 29. Handling High Traffic

Suppose:

```text
10 million active users
```

and each user performs multiple actions per day.

The system may experience huge numbers of:

```text
Feed requests
Swipes
Messages
Photo requests
Notifications
```

We can scale horizontally:

```text
                 Load Balancer
                /      |      \
               ↓       ↓       ↓
             Server  Server  Server
```

Stateless services make horizontal scaling easier.

---

# 30. Rate Limiting

Users shouldn't be able to generate unlimited requests.

For example:

```text
100 requests / minute
```

could be enforced at the API Gateway.

```text
Client
 ↓
Rate Limiter
 ↓
API Gateway
 ↓
Services
```

Rate limiting protects the system from:

* Abuse
* Bots
* Accidental traffic spikes
* Denial-of-service attacks

---

# 31. Handling Hot Users

Some profiles may become extremely popular.

Suppose one user receives millions of likes.

Their data can become a **hot key**.

```text
          Millions of requests
                  ↓
             User 123
                  ↓
              Database
```

This can overload one shard.

Possible solutions include:

* Caching hot profiles
* Replication
* Request distribution
* Avoiding single-key bottlenecks

---

# 32. High-Level Final Architecture

A more complete design:

```text
                           Clients
                              |
                              ↓
                       ┌─────────────┐
                       │ API Gateway │
                       └──────┬──────┘
                              |
          ┌───────────────────┼────────────────────┐
          ↓                   ↓                    ↓
    User Service       Discovery Service       Swipe Service
          |                   |                    |
       User DB          Recommendation        Swipe DB
                              |
                            Cache
                              |
                              ↓
                       Matching Service
                              |
                          Match DB
                              |
                              ↓
                         Event Broker
                       /      |       \
                      ↓       ↓        ↓
               Notification Analytics Recommendation
                      |
                      ↓
                Push Notification


        ─────────────── CHAT ───────────────

Client A ←→ WebSocket Gateway ←→ Client B
                  |
                  ↓
             Chat Service
                  |
                  ↓
            Message Store


        ───────────── PHOTOS ─────────────

Client
  ↓
Object Storage
  ↓
CDN
  ↓
Users
```

---

# 33. Main Design Decisions

| Problem             | Solution                        |
| ------------------- | ------------------------------- |
| API communication   | REST/HTTP                       |
| Real-time chat      | WebSockets                      |
| Profile photos      | Object Storage                  |
| Fast photo delivery | CDN                             |
| Frequent reads      | Cache                           |
| Nearby users        | Geospatial indexing             |
| Feed generation     | Candidate generation + ranking  |
| Large datasets      | Sharding                        |
| Distributed cache   | Consistent hashing              |
| Async processing    | Message Broker                  |
| Event communication | Pub/Sub                         |
| Notifications       | Event-driven                    |
| Duplicate matches   | Unique constraint / idempotency |
| High traffic        | Horizontal scaling              |
| Abuse protection    | Rate limiting                   |

---

# 34. Most Important System Design Concepts

This problem connects almost everything we've studied:

```text
                    Tinder
                      |
       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
    APIs          Databases       Caching
       ↓              ↓              ↓
  API Gateway     SQL/NoSQL       Redis
                      |
                 Sharding
                      |
              Consistent Hashing


       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
    Pub/Sub       Message Queue   Event Driven
       ↓              ↓              ↓
 Notifications    Async Work     Analytics


       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
      CDN          WebSockets     Geo Search
       ↓              ↓              ↓
     Photos          Chat        Discovery
```

---

# 35. Interview Flow

If asked to design Tinder in an interview, don't immediately start drawing databases.

A good approach is:

```text
1. Clarify requirements
        ↓
2. Define APIs
        ↓
3. Estimate scale
        ↓
4. High-level architecture
        ↓
5. Design user discovery
        ↓
6. Design swipe + matching
        ↓
7. Design chat
        ↓
8. Design notifications
        ↓
9. Database + sharding
        ↓
10. Caching + CDN
        ↓
11. Failure handling
        ↓
12. Bottlenecks + trade-offs
```

The **three most important components to deep-dive** are:

### 1. Discovery

How do we efficiently find and rank nearby candidates?

### 2. Matching

How do we detect mutual likes while preventing duplicate matches?

### 3. Chat

How do we maintain millions of real-time connections and reliably deliver messages?

---

# Key Takeaway

A simplified mental model of Tinder is:

```text
                    USER
                     |
                     ↓
               DISCOVERY
                     |
                     ↓
                   SWIPE
                     |
                     ↓
                 MATCHING
                     |
              ┌──────┴──────┐
              ↓             ↓
          NOTIFICATION     CHAT
```

Underneath these features:

```text
APIs
 ↓
Load Balancers
 ↓
Stateless Services
 ↓
Caches + Databases
 ↓
Message Brokers
 ↓
Object Storage + CDN
 ↓
WebSockets
```

> **The core challenge in designing Tinder isn't the swipe itself. The difficult problems are efficiently generating nearby candidates, handling billions of swipes, detecting matches reliably, and supporting real-time chat at massive scale.**
