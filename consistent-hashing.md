# Consistent Hashing

## What is Consistent Hashing?

Consistent hashing is a technique used in **distributed systems** to distribute data across multiple servers while minimizing the amount of data that needs to be moved when servers are added or removed.

It is commonly used in:

* Distributed caches
* Distributed databases
* Load balancing
* CDNs
* Sharding systems

### The Problem with Normal Hashing

Suppose we have 4 servers:

```text
Server 0
Server 1
Server 2
Server 3
```

A simple approach is:

```text
server = hash(key) % N
```

where `N` is the number of servers.

For example:

```text
hash("user_123") % 4 = 2
```

So `user_123` goes to Server 2.

The problem occurs when a server is added.

If we add Server 4:

```text
hash(key) % 5
```

Now the result changes for **most keys**.

This means a huge amount of data has to be redistributed.

### Consistent Hashing Solution

Consistent hashing maps both:

* Servers
* Data keys

onto the same **hash ring**.

```text
                 Server A
                    ↓
              ┌───────────┐
           ┌───┘           └───┐
         Key 1               Key 2
         ↑                     ↑
      Server C              Server B
           └───┐           ┌───┘
               └───────────┘
```

The hash space is treated as a circular ring.

For example:

```text
0 ------------------------------ 2^32
 \                                /
  \______________________________/
```

### How It Works

#### 1. Hash the servers

Each server is assigned a position on the ring:

```text
hash(Server A) → position 100
hash(Server B) → position 400
hash(Server C) → position 700
```

#### 2. Hash the key

Suppose:

```text
hash("user_123") → 450
```

We move clockwise on the ring until we find the first server.

```text
450 → Server C at 700
```

Therefore:

```text
user_123 → Server C
```

### Adding a Server

Suppose we add Server D between Server B and Server C.

Only the keys between Server B and Server D need to move.

```text
Before:

B -------------------- C


After:

B -------- D --------- C
```

Instead of redistributing **all keys**, only a portion of the keys are affected.

This is the main advantage of consistent hashing.

### Removing a Server

If Server D fails, its keys are simply assigned to the next server clockwise.

```text
D → removed

Keys previously handled by D
        ↓
Next server clockwise
```

Again, only a subset of the data needs to move.

---

## Virtual Nodes

A problem with placing each physical server at only one point on the ring is **uneven distribution**.

For example:

```text
Server A -------- Server B ---------------- Server C
```

Server C might end up responsible for a much larger portion of the ring.

### Solution: Virtual Nodes

Instead of assigning one position to each server, assign multiple positions.

```text
Server A → A1, A2, A3, A4
Server B → B1, B2, B3, B4
Server C → C1, C2, C3, C4
```

These virtual nodes are distributed around the ring.

```text
A1 → B2 → C1 → A3 → B4 → C3 → A2 → ...
```

This provides better load distribution.

It also makes the system more resilient when a server joins or leaves.

---

## Advantages

### 1. Minimal Data Movement

When servers are added or removed, only a small portion of the keys need to be redistributed.

### 2. Horizontal Scalability

New servers can be added without completely reshuffling the dataset.

### 3. Fault Tolerance

When a server fails, its keys can be assigned to another server.

### 4. Better Distribution with Virtual Nodes

Virtual nodes reduce the possibility of uneven load distribution.

---

## Disadvantages

### 1. More Complexity

It is more complicated than:

```text
hash(key) % N
```

### 2. Uneven Distribution Without Virtual Nodes

A poor distribution of server positions can cause some servers to receive significantly more traffic.

### 3. Rebalancing Still Exists

Consistent hashing reduces data movement; it does **not** eliminate it completely.

---

## Consistent Hashing vs Modulo Hashing

| Feature             | Modulo Hashing     | Consistent Hashing      |
| ------------------- | ------------------ | ----------------------- |
| Algorithm           | `hash(key) % N`    | Hash ring               |
| Adding server       | Many keys remapped | Small portion remapped  |
| Removing server     | Many keys remapped | Small portion remapped  |
| Complexity          | Simple             | More complex            |
| Load distribution   | Can be good        | Good with virtual nodes |
| Distributed systems | Less suitable      | Very suitable           |

---

## Real-World Applications

Consistent hashing is useful when a distributed system needs to decide **which node should own a particular piece of data**.

Examples:

* Distributed caches
* Database sharding
* CDN routing
* Distributed storage
* Load balancing

---

## Key Takeaway

> Consistent hashing solves the problem of excessive data redistribution when nodes are added or removed from a distributed system.

The key ideas are:

```text
Hash Ring
    ↓
Map servers onto ring
    ↓
Map keys onto ring
    ↓
Assign key to next server clockwise
    ↓
Use virtual nodes for better distribution
```
