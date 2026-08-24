# SQL vs NoSQL

## What are SQL Databases?

**SQL (Structured Query Language) databases** are relational databases that store data in **tables consisting of rows and columns**.

Example:

```text
Users

+----+--------+-------------------+
| ID | Name   | Email             |
+----+--------+-------------------+
| 1  | Anshu  | anshu@gmail.com   |
| 2  | Rahul  | rahul@gmail.com   |
+----+--------+-------------------+
```

Common SQL databases:

* PostgreSQL
* MySQL
* Oracle
* Microsoft SQL Server

---

# What are NoSQL Databases?

**NoSQL** databases are non-relational databases designed for flexible schemas, horizontal scalability, and specific data-access patterns.

Common types include:

### Document

Stores data as JSON-like documents.

Examples:

* MongoDB
* Couchbase

```json
{
  "id": 123,
  "name": "Anshu",
  "email": "anshu@gmail.com"
}
```

### Key-Value

Stores data as:

```text
key → value
```

Examples:

* Redis
* Amazon DynamoDB

```text
"user:123" → "{name: Anshu, age: 22}"
```

### Wide-Column

Stores data in column families.

Examples:

* Apache Cassandra
* HBase

### Graph

Stores nodes and relationships.

Examples:

* Neo4j
* Amazon Neptune

---

# SQL vs NoSQL: Core Difference

The fundamental difference is the **data model and scaling approach**.

```text
SQL

Application
     ↓
Relational Database
     ↓
Tables + Relationships


NoSQL

Application
     ↓
NoSQL Database
     ↓
Documents / Key-Value / Columns / Graph
```

SQL databases generally prioritize **strong relationships, structured schemas, and transactions**.

NoSQL databases generally prioritize **flexible data models, horizontal scalability, and high-throughput access patterns**.

---

# Schema

## SQL — Structured Schema

SQL databases typically define the schema before inserting data.

```sql
CREATE TABLE users (
    id INT,
    name VARCHAR(100),
    email VARCHAR(255)
);
```

Every row follows the defined structure.

```text
ID | Name | Email
```

Changing the schema may require a migration.

---

## NoSQL — Flexible Schema

A document database can store documents with different fields.

```json
{
  "name": "Anshu",
  "email": "anshu@gmail.com"
}
```

Another document could contain:

```json
{
  "name": "Rahul",
  "email": "rahul@gmail.com",
  "age": 25,
  "location": "Delhi"
}
```

This flexibility is useful when the data structure evolves frequently.

---

# Relationships

SQL databases are designed around relationships between entities.

Suppose we have:

```text
Users
Orders
Products
```

We can model:

```text
User
  ↓
Orders
  ↓
Products
```

Using foreign keys:

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

This makes SQL particularly strong for highly relational data.

---

# Joins

SQL databases support powerful joins.

For example:

```sql
SELECT users.name, orders.id
FROM users
JOIN orders
ON users.id = orders.user_id;
```

This allows related data to be queried across tables.

NoSQL databases generally avoid heavy joins and instead encourage data to be structured around the queries the application needs.

---

# Normalization

SQL databases commonly use **normalization** to reduce duplicate data.

For example:

```text
Users
+----+-------+
| ID | Name  |
+----+-------+
| 1  | Anshu |
+----+-------+

Orders
+----+---------+
| ID | User_ID |
+----+---------+
| 10 | 1       |
+----+---------+
```

Instead of repeatedly storing the user's information inside every order, we reference the user through `User_ID`.

### Advantages

* Less duplication
* Better consistency
* Easier updates

### Disadvantage

Queries may require joins.

---

# Denormalization

NoSQL systems often favor **denormalization**.

Instead of:

```text
Users
Orders
Products
```

we may store related information together:

```json
{
  "orderId": 123,
  "user": {
    "id": 42,
    "name": "Anshu"
  },
  "items": [
    {
      "productId": 10,
      "name": "Laptop",
      "price": 80000
    }
  ]
}
```

Now a single query can retrieve most of the required information.

### Advantage

Fast reads.

### Disadvantage

Data duplication and more difficult updates.

---

# ACID Transactions

SQL databases are well known for strong **ACID** transaction guarantees.

## Atomicity

A transaction either completes completely or doesn't happen.

```text
Transfer ₹100

Debit → Success
Credit → Success

Transaction complete
```

If the credit fails:

```text
Debit → Success
Credit → Failed

Rollback
```

---

## Consistency

A transaction moves the database from one valid state to another valid state while maintaining defined constraints.

---

## Isolation

Concurrent transactions should not incorrectly interfere with each other.

---

## Durability

Once a transaction is committed, the data should survive failures.

---

# NoSQL and Transactions

Modern NoSQL databases are not simply "non-transactional."

Many support transactions and consistency guarantees.

However, their transaction model and scalability trade-offs vary by database.

So avoid the outdated rule:

> "SQL has transactions, NoSQL doesn't."

A better statement is:

> SQL databases traditionally emphasize relational integrity and strong ACID transactions, while NoSQL databases often make different trade-offs to achieve flexible schemas and distributed scalability.

---

# Scaling

This is one of the most important system-design differences.

## SQL — Vertical Scaling

A traditional approach is to make the database server more powerful.

```text
        Database
           ↓
   Bigger CPU / RAM / SSD
```

For example:

```text
8 CPU  → 32 CPU
32 GB  → 128 GB RAM
```

This is called **vertical scaling**.

SQL databases can also scale horizontally, but doing so can introduce additional architectural complexity.

---

# NoSQL — Horizontal Scaling

NoSQL databases are often designed with horizontal scaling in mind.

Instead of one huge machine:

```text
        Database
```

use multiple machines:

```text
      Database Cluster
      /      |      \
    Node 1  Node 2  Node 3
```

Need more capacity?

```text
      Database Cluster
      /      |      |      \
    Node 1  Node 2  Node 3  Node 4
```

This is called **horizontal scaling**.

---

# Sharding

Large datasets can be divided across multiple database nodes.

For example:

```text
Users 1–1M
      ↓
   Shard 1

Users 1M–2M
      ↓
   Shard 2

Users 2M–3M
      ↓
   Shard 3
```

The database uses a **partition key** to determine where the data belongs.

NoSQL databases commonly make distributed partitioning a core part of their architecture.

SQL databases can also use sharding, but it may require more application-level or infrastructure-level design.

---

# Replication

Both SQL and NoSQL databases can use replication.

```text
             Primary
             /     \
            ↓       ↓
       Replica 1  Replica 2
```

Replication can provide:

* High availability
* Read scalability
* Fault tolerance

For example:

```text
Writes
  ↓
Primary

Reads
  ↓
Replicas
```

---

# CAP Theorem

CAP is particularly important when discussing distributed NoSQL systems.

A distributed system can provide guarantees around:

* **Consistency**
* **Availability**
* **Partition Tolerance**

The key idea is:

> When a network partition occurs, a distributed system must trade off between consistency and availability.

```text
          CAP
        /  |  \
Consistency | Availability
        \   |  /
      Partition
      Tolerance
```

### Consistency

Every read receives the latest write according to the system's consistency model.

### Availability

Every request receives a response, even if some nodes are unavailable.

### Partition Tolerance

The system continues operating despite network failures between nodes.

In real distributed systems, **partition tolerance is generally unavoidable**, so the practical trade-off is often between consistency and availability during a partition.

---

# SQL vs NoSQL Example

Consider an online banking system.

Data:

```text
Users
Accounts
Transactions
```

There are strong relationships and financial transactions need strict correctness.

A relational database is often a strong choice:

```text
PostgreSQL / MySQL
```

Now consider a social media feed.

You may need:

```text
Millions of users
Millions of posts
Huge write throughput
Flexible post metadata
Horizontal scaling
```

A NoSQL system may be a better fit for some parts of the architecture.

---

# When Should You Choose SQL?

SQL is generally a strong choice when:

* Data has complex relationships
* Strong consistency is important
* ACID transactions are important
* Complex queries are required
* Data structure is relatively stable
* Joins are important
* Referential integrity matters

Examples:

```text
Banking
Payments
ERP systems
Accounting
Inventory management
Order management
```

---

# When Should You Choose NoSQL?

NoSQL can be a strong choice when:

* Data structure changes frequently
* Very large scale is required
* High write/read throughput is needed
* Horizontal scaling is important
* Data is naturally document/key-value/graph based
* Access patterns are well-defined
* Joins aren't central to the workload

Examples:

```text
Caching
Real-time analytics
Large-scale user activity
IoT data
Gaming
Social feeds
Session storage
```

---

# Important: Don't Choose Based Only on Scale

A common misconception is:

> "If the application is large, use NoSQL."

That's not necessarily true.

Large systems frequently use **both** SQL and NoSQL databases.

For example:

```text
                    Application
                   /           \
                  ↓             ↓
             PostgreSQL       Redis
                  ↓             ↓
             Transactions     Cache
```

Another component might use:

```text
Application
    ↓
Cassandra
    ↓
High-volume events
```

Database selection should be based on:

* Data model
* Access patterns
* Consistency requirements
* Transaction requirements
* Scale
* Query patterns
* Operational complexity

---

# SQL + NoSQL Together

A real system might look like:

```text
                         Application
                       /      |       \
                      ↓       ↓        ↓
                 PostgreSQL Redis   MongoDB
                    ↓        ↓         ↓
               Transactions Cache   Documents
```

Each database is used for the workload it handles best.

This is sometimes called **polyglot persistence**.

---

# SQL vs NoSQL Comparison

| Feature           | SQL                                         | NoSQL                                       |
| ----------------- | ------------------------------------------- | ------------------------------------------- |
| Data model        | Relational                                  | Document / Key-Value / Wide-Column / Graph  |
| Schema            | Usually fixed/structured                    | Usually flexible                            |
| Relationships     | Strong                                      | Often limited or application-managed        |
| Joins             | Excellent                                   | Usually avoided                             |
| Transactions      | Strong ACID support                         | Varies by database                          |
| Scaling           | Traditionally vertical; horizontal possible | Often designed for horizontal scaling       |
| Data consistency  | Often strong                                | Varies                                      |
| Query flexibility | High                                        | Depends on database                         |
| Schema changes    | Usually require migrations                  | Usually easier                              |
| Best for          | Relational, transactional workloads         | Large-scale, flexible/distributed workloads |

---

# Common Mistakes

### ❌ "SQL can't scale horizontally"

False.

SQL databases can scale horizontally using:

* Read replicas
* Sharding
* Partitioning
* Distributed SQL databases

---

### ❌ "NoSQL doesn't support transactions"

False.

Many modern NoSQL databases support transactions.

---

### ❌ "NoSQL is always faster"

False.

Performance depends heavily on:

* Query patterns
* Indexes
* Data model
* Hardware
* Workload
* Architecture

---

### ❌ "SQL = small systems, NoSQL = large systems"

False.

Both can power extremely large systems.

---

# Interview Cheat Sheet

### SQL vs NoSQL?

```text
SQL
→ Relational
→ Structured schema
→ Joins
→ Strong transactions
→ Complex queries

NoSQL
→ Non-relational
→ Flexible data models
→ Horizontal scalability
→ Often optimized for specific access patterns
```

### What is normalization?

Structuring data to reduce duplication and improve consistency.

### What is denormalization?

Intentionally duplicating data to optimize reads and reduce expensive joins.

### What is sharding?

Splitting data across multiple database nodes.

### What is replication?

Maintaining copies of data on multiple nodes.

### What is polyglot persistence?

Using different database technologies for different workloads within the same system.

---

# Mental Model

Don't ask:

> **"Should I use SQL or NoSQL?"**

Ask:

> **"What are my data, queries, consistency requirements, transaction requirements, and scaling requirements?"**

Then choose the database that best fits those requirements.

```text
                    Database Choice
                          |
          ┌───────────────┼───────────────┐
          ↓               ↓               ↓
       Data Model     Consistency       Scale
          ↓               ↓               ↓
       Relational?     Strong?        Horizontal?
          ↓               ↓               ↓
       SQL / NoSQL based on workload
```

> **SQL is not the "old" choice and NoSQL is not the "modern" choice. They are tools optimized for different system requirements.**
