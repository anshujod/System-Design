# Publisher-Subscriber Model (Pub/Sub)

## What is Pub/Sub?

**Publisher-Subscriber (Pub/Sub)** is a messaging pattern where producers of messages, called **publishers**, do not directly communicate with consumers, called **subscribers**.

Instead, publishers send messages to a **topic**, and subscribers receive messages from that topic.

```text
Publisher
    |
    ↓
  Topic
  / | \
 ↓  ↓  ↓
S1 S2 S3
```

The publisher doesn't need to know who the subscribers are.

This creates **loose coupling** between services.

---

# Why Do We Need Pub/Sub?

Consider an e-commerce application.

When an order is created, multiple services may need to react:

```text
Order Created
      |
      ├──→ Payment Service
      ├──→ Inventory Service
      ├──→ Email Service
      ├──→ Analytics Service
      └──→ Recommendation Service
```

Without Pub/Sub, the Order Service might have to call each service individually:

```text
Order Service
   ├──→ Payment
   ├──→ Inventory
   ├──→ Email
   ├──→ Analytics
   └──→ Recommendation
```

This creates tight coupling.

With Pub/Sub:

```text
                 ┌→ Payment
                 |
Order Service → Topic
                 |
                 ├→ Inventory
                 |
                 ├→ Email
                 |
                 └→ Analytics
```

The Order Service only publishes:

```text
ORDER_CREATED
```

The subscribers decide what to do with it.

---

# Core Components

A Pub/Sub system generally contains three major components:

```text
Publisher → Topic/Broker → Subscribers
```

## 1. Publisher

The publisher produces events or messages.

Example:

```json
{
  "event": "ORDER_CREATED",
  "orderId": 123,
  "userId": 42
}
```

The publisher doesn't need to know which services are consuming the event.

---

## 2. Topic

A **topic** is a logical channel to which publishers send messages.

Example:

```text
orders
payments
notifications
user-events
```

For example:

```text
Publisher
    ↓
orders topic
```

---

## 3. Subscriber

A subscriber listens to a topic and processes its messages.

```text
orders topic
    |
    ├──→ Payment Service
    ├──→ Inventory Service
    └──→ Analytics Service
```

Each subscriber can independently process the event.

---

# Basic Flow

Suppose a user places an order.

### Step 1 — Order Service publishes an event

```text
Order Service
      ↓
ORDER_CREATED
```

### Step 2 — Event is published to a topic

```text
ORDER_CREATED
      ↓
 orders topic
```

### Step 3 — Multiple subscribers receive it

```text
                orders topic
                /     |      \
               ↓      ↓       ↓
          Payment  Inventory  Email
```

Each service processes the event independently.

---

# Pub/Sub vs Point-to-Point Queue

This is an important distinction.

## Message Queue

Typically:

```text
Producer
   ↓
Queue
   ↓
Consumer
```

A message is generally processed by **one consumer**.

If we have:

```text
Queue
 ↓
C1
C2
C3
```

the consumers typically compete for messages.

For example:

```text
Message A → C1
Message B → C2
Message C → C3
```

This is useful for **work distribution**.

---

# Pub/Sub

In Pub/Sub:

```text
             Topic
           /   |   \
          ↓    ↓    ↓
         C1   C2   C3
```

Each subscriber can receive the event.

For example:

```text
ORDER_CREATED

       ↓
     Topic
   /   |    \
  ↓    ↓     ↓
Payment Inventory Email
```

This is useful for **event broadcasting**.

---

# Queue vs Pub/Sub

| Feature       | Message Queue                               | Pub/Sub                                |
| ------------- | ------------------------------------------- | -------------------------------------- |
| Main purpose  | Work distribution                           | Event broadcasting                     |
| Consumers     | Usually one consumer processes each message | Multiple subscribers can receive event |
| Communication | Producer → Queue → Consumer                 | Publisher → Topic → Subscribers        |
| Coupling      | Decoupled                                   | Highly decoupled                       |
| Example       | Image processing jobs                       | Order-created events                   |
| Typical use   | Background tasks                            | Event-driven architecture              |

---

# Consumer Groups

Consumer groups allow multiple instances of the same service to process messages in parallel.

For example:

```text
              orders topic
                   |
            Payment Group
              /    |    \
             ↓     ↓     ↓
           P1      P2     P3
```

Suppose there are three messages:

```text
Order 1 → P1
Order 2 → P2
Order 3 → P3
```

The instances share the workload.

At the same time, another subscriber can have its own consumer group:

```text
                 orders topic
                /            \
               ↓              ↓
        Payment Group     Analytics Group
        /   |   \          /    |    \
       P1   P2   P3        A1   A2   A3
```

Payment and Analytics independently consume the same events.

---

# Fan-Out

One of the biggest advantages of Pub/Sub is **fan-out**.

A single event can be delivered to many subscribers.

```text
                    EVENT
                      |
                      ↓
                    Topic
                /     |     \
               ↓      ↓      ↓
             Email  Analytics Inventory
```

For example:

```text
USER_REGISTERED
```

could trigger:

```text
Email Service       → Send welcome email
Analytics Service   → Record signup
Recommendation      → Initialize recommendations
CRM Service         → Create customer profile
```

The publisher only needs to publish one event.

---

# Push vs Pull Subscribers

Subscribers can receive events in different ways.

## Pull

The subscriber asks the broker for messages.

```text
Subscriber → "Give me messages"
      ↓
    Broker
      ↓
  Messages
```

### Advantages

* Consumer controls its processing rate
* Easier backpressure
* Consumers can batch messages

---

## Push

The broker sends messages to the subscriber.

```text
Broker
  |
  | Event
  ↓
Subscriber
```

### Advantages

* Low latency
* No continuous polling
* Simple event delivery

However, the system needs mechanisms to prevent a slow subscriber from being overwhelmed.

---

# Message Retention

Some Pub/Sub systems retain messages for a period of time.

For example:

```text
Event
 ↓
Topic
 ↓
Retained for 7 days
```

This allows a new or recovering subscriber to consume older events.

This is different from a simple transient message delivery system where the event disappears after being consumed.

---

# Delivery Guarantees

Pub/Sub systems commonly deal with three delivery models.

## At-Most-Once

The event is delivered zero or one time.

```text
Event → Subscriber
```

The event may be lost, but duplicates are avoided.

---

## At-Least-Once

The system retries delivery until the subscriber acknowledges the message.

```text
Event
 ↓
Subscriber crashes
 ↓
Retry
 ↓
Subscriber
```

The event won't normally be intentionally lost, but duplicates can occur.

Therefore, subscribers should ideally be **idempotent**.

---

## Exactly-Once

The system attempts to ensure that an event is processed exactly once.

This is difficult in distributed systems and often requires additional coordination or deduplication mechanisms.

---

# Ordering

Sometimes event order matters.

Consider:

```text
1. CREATE_ACCOUNT
2. UPDATE_ACCOUNT
3. DELETE_ACCOUNT
```

Processing:

```text
DELETE
CREATE
UPDATE
```

could result in an incorrect state.

Therefore, Pub/Sub systems may provide ordering guarantees within a particular topic partition, key, or ordering boundary.

However, maintaining global ordering across a highly distributed system can reduce scalability.

---

# Retry and Dead Letter Topics

If a subscriber repeatedly fails to process an event:

```text
Topic
 ↓
Subscriber
 ↓
Failure
 ↓
Retry
 ↓
Retry
 ↓
Retry limit reached
 ↓
Dead Letter Topic
```

The failed event can then be inspected separately.

This prevents a single problematic event from continuously blocking normal processing.

---

# Pub/Sub and Event-Driven Architecture

Pub/Sub is one of the foundations of **event-driven architecture**.

Instead of services directly calling each other:

```text
Service A → Service B → Service C
```

services communicate through events:

```text
Service A
   ↓
Event
   ↓
Broker
   ├──→ Service B
   ├──→ Service C
   └──→ Service D
```

This makes services more independent.

---

# Advantages

## 1. Loose Coupling

Publishers don't need to know about subscribers.

## 2. Scalability

Subscribers can scale independently.

```text
Topic
 ↓
C1 C2 C3 C4 C5
```

## 3. Fan-Out

One event can trigger multiple independent workflows.

## 4. Fault Isolation

If one subscriber is down, other subscribers can potentially continue processing.

## 5. Extensibility

You can add a new subscriber without modifying the publisher.

For example:

```text
Existing:

Order → Topic → Payment
             → Inventory

Later:

Order → Topic → Payment
             → Inventory
             → Fraud Detection
```

The Order Service doesn't need to change.

---

# Disadvantages

## 1. Eventual Consistency

Services may process events at different times.

Therefore, different services may temporarily have different views of the system.

## 2. Duplicate Events

At-least-once delivery can cause duplicate processing.

## 3. Ordering Complexity

Maintaining strict global ordering can reduce scalability.

## 4. Debugging Complexity

A single event can trigger many asynchronous operations.

Tracing:

```text
Order Created
   ↓
Payment
   ↓
Email
   ↓
Analytics
```

across multiple services can be difficult.

## 5. Operational Complexity

You need to monitor:

* Topic throughput
* Consumer lag
* Failed messages
* Retry rates
* Dead-letter messages
* Subscriber health

---

# Real-World Use Cases

Pub/Sub is commonly used for:

* Microservice communication
* Order processing
* Notifications
* Analytics pipelines
* Log processing
* User activity tracking
* Fraud detection
* Recommendation systems
* Data synchronization
* Event-driven architectures

---

# Example: E-Commerce System

Suppose an order is placed.

```text
                     ORDER_CREATED
                           |
                           ↓
                      Orders Topic
                    /      |       \
                   ↓       ↓        ↓
              Payment  Inventory   Email
                 ↓         ↓          ↓
             Process     Reserve    Send
             Payment      Stock     Email
```

Later, we add fraud detection:

```text
                     Orders Topic
                          |
        ┌─────────┬───────┼────────┬──────────┐
        ↓         ↓       ↓        ↓          ↓
     Payment  Inventory  Email  Analytics   Fraud
```

The Order Service doesn't need to know about the new Fraud Service.

This is the main power of Pub/Sub.

---

# Pub/Sub vs Message Queue — Mental Model

Think of a **queue** as:

> "Who can do this job?"

```text
                Queue
              /   |   \
             ↓    ↓    ↓
            C1   C2   C3

One message → One consumer
```

Think of **Pub/Sub** as:

> "Who cares about this event?"

```text
                Topic
              /   |   \
             ↓    ↓    ↓
           Email Analytics Fraud

One event → Multiple subscribers
```

---

# Key Takeaways

```text
Publisher
    ↓
  Topic
    ↓
Subscribers
```

The most important concepts are:

* **Publisher** → Produces events
* **Topic** → Logical channel for events
* **Subscriber** → Consumes events
* **Fan-out** → One event → Multiple subscribers
* **Consumer group** → Multiple instances share workload
* **At-least-once delivery** → Duplicates are possible
* **Idempotency** → Important for safe duplicate processing
* **Dead-letter topic** → Handles repeatedly failed events
* **Eventual consistency** → Common consequence of asynchronous processing

> **Pub/Sub is primarily about decoupling event producers from multiple independent consumers.**
