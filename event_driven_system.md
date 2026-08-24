# Event-Driven Systems

## What is an Event-Driven System?

An **event-driven system** is an architecture where components communicate by producing and reacting to **events**.

An event represents something that **has already happened**.

Examples:

```text
ORDER_CREATED
PAYMENT_COMPLETED
USER_REGISTERED
VIDEO_UPLOADED
PAYMENT_FAILED
```

Instead of services directly calling each other, one service publishes an event and other services react to it.

```text
Service A
   |
   | Event
   ↓
Message Broker
   |
   ├──→ Service B
   ├──→ Service C
   └──→ Service D
```

---

# What is an Event?

An event is a record of something that happened in the system.

For example:

```json
{
  "event": "ORDER_CREATED",
  "orderId": 123,
  "userId": 42,
  "timestamp": "2026-08-24T10:00:00Z"
}
```

The important idea is:

> An event describes a **fact**, not a command.

Compare:

```text
Command:
"Send an email"

Event:
"Order was created"
```

A command tells another component what to do.

An event tells other components what happened.

---

# Traditional Synchronous Architecture

Suppose a user places an order.

A traditional approach might be:

```text
Client
  ↓
Order Service
  ↓
Payment Service
  ↓
Inventory Service
  ↓
Notification Service
```

The Order Service directly calls other services.

This creates **tight coupling**.

If the Notification Service is down:

```text
Order Service
      ↓
Notification Service ❌
```

The failure may affect the entire request flow.

---

# Event-Driven Architecture

Instead:

```text
Client
  ↓
Order Service
  ↓
ORDER_CREATED
  ↓
Message Broker
  ├──→ Payment Service
  ├──→ Inventory Service
  ├──→ Notification Service
  └──→ Analytics Service
```

The Order Service doesn't directly call all of these services.

It simply publishes:

```text
ORDER_CREATED
```

Other services decide whether they care about that event.

---

# Core Components

A typical event-driven system contains:

```text
Producer
    ↓
Event
    ↓
Event Broker
    ↓
Consumers
```

## 1. Producer

The producer generates an event.

Example:

```text
Order Service
      ↓
ORDER_CREATED
```

---

## 2. Event Broker

The broker receives and distributes events.

Examples of technologies used for event-driven architectures include:

* Apache Kafka
* Amazon SNS
* Google Pub/Sub
* RabbitMQ
* Apache Pulsar

The broker acts as the communication layer between producers and consumers.

---

## 3. Consumer

A consumer listens for events and performs some action.

```text
ORDER_CREATED
      ↓
Payment Service
      ↓
Process payment
```

Another consumer might do:

```text
ORDER_CREATED
      ↓
Analytics Service
      ↓
Record order metric
```

---

# Basic Flow

Consider a video platform.

A user uploads a video.

### Step 1

The Upload Service stores the video.

```text
User
 ↓
Upload Service
```

### Step 2

It publishes:

```text
VIDEO_UPLOADED
```

### Step 3

Multiple services react:

```text
                  VIDEO_UPLOADED
                        |
                        ↓
                      Broker
                    /   |    \
                   ↓    ↓     ↓
              Transcode Thumbnail Analytics
```

The services work independently.

---

# Event-Driven vs Request-Driven

## Request-Driven

One service explicitly asks another service to perform an operation.

```text
Service A
   |
   | HTTP Request
   ↓
Service B
```

Service A waits for a response.

---

## Event-Driven

A service publishes an event.

```text
Service A
   |
   | Event
   ↓
Broker
   |
   ↓
Service B
```

Service A doesn't necessarily wait for Service B.

---

# Synchronous vs Asynchronous Events

Event-driven systems are often asynchronous.

```text
Order Service
     ↓
ORDER_CREATED
     ↓
Broker
     ↓
Payment Service
```

The Order Service can return to the client without waiting for every downstream service.

This improves responsiveness and decoupling.

However, it also introduces **eventual consistency**.

---

# Eventual Consistency

Suppose:

```text
Order Created
      ↓
ORDER_CREATED
      ↓
Inventory Service
```

The inventory system may process the event a few milliseconds or seconds later.

During that period:

```text
Order Service → Updated
Inventory     → Not yet updated
```

The system is temporarily inconsistent.

Eventually:

```text
Inventory processes event
          ↓
System becomes consistent
```

This is called **eventual consistency**.

---

# Pub/Sub and Event-Driven Systems

Pub/Sub is a common mechanism for implementing event-driven architecture.

```text
Publisher
    ↓
   Topic
  /  |  \
 ↓   ↓   ↓
 C1  C2  C3
```

For example:

```text
Order Service
      ↓
orders topic
      |
      ├──→ Payment
      ├──→ Inventory
      ├──→ Email
      └──→ Analytics
```

So:

> **Pub/Sub is a messaging pattern; event-driven architecture is a broader architectural style.**

---

# Event Queue vs Event Stream

There are two important approaches.

## Event Queue

Messages are generally consumed as work.

```text
Producer
   ↓
Queue
   ↓
Consumer
```

Once successfully processed, the message may be removed.

Useful for:

* Background jobs
* Email processing
* Image processing
* Task distribution

---

## Event Stream

Events are retained for some period and consumers can process them independently.

```text
Producer
   ↓
Event Stream
   ├──→ Consumer A
   ├──→ Consumer B
   └──→ Consumer C
```

This is useful for:

* Analytics
* Event sourcing
* Activity tracking
* Data pipelines
* Real-time processing

---

# Event Ordering

Events may sometimes need to be processed in order.

For example:

```text
ACCOUNT_CREATED
ACCOUNT_UPDATED
ACCOUNT_DELETED
```

If processed as:

```text
ACCOUNT_DELETED
ACCOUNT_CREATED
ACCOUNT_UPDATED
```

the resulting state could be incorrect.

Distributed event systems therefore often provide ordering guarantees within a particular:

* Partition
* Key
* Topic
* Consumer group

Global ordering is much harder and can reduce scalability.

---

# Event Delivery Guarantees

## At-Most-Once

```text
Event → Consumer
```

The event is delivered zero or one time.

Possible problem:

```text
Event lost ❌
```

---

## At-Least-Once

The system retries until the event is acknowledged.

```text
Event
 ↓
Consumer crashes
 ↓
Retry
 ↓
Consumer
```

The downside is that duplicate events can occur.

Therefore:

> Consumers should ideally be **idempotent**.

---

## Exactly-Once

The system attempts to ensure that an event is processed exactly once.

This is difficult to guarantee in distributed systems and usually requires additional mechanisms.

---

# Idempotency

Suppose a payment event is delivered twice:

```text
PAYMENT_COMPLETED
PAYMENT_COMPLETED
```

If the consumer charges the customer twice, that's a serious problem.

Instead, the consumer can track processed event IDs:

```json
{
  "eventId": "abc123",
  "status": "processed"
}
```

If the same event arrives again:

```text
eventId = abc123
       ↓
Already processed
       ↓
Ignore
```

This makes the consumer idempotent.

---

# Failure Handling

Distributed systems fail.

A consumer might be unavailable:

```text
Broker
  ↓
Payment Service ❌
```

A good event-driven system should support:

* Retries
* Exponential backoff
* Dead-letter queues/topics
* Idempotency
* Monitoring
* Error handling

Example:

```text
Event
 ↓
Consumer
 ↓
Failure
 ↓
Retry
 ↓
Retry
 ↓
Retry limit
 ↓
Dead Letter Queue
```

---

# Advantages

## 1. Loose Coupling

Services don't need direct knowledge of each other.

```text
Service A → Event → Broker
```

Service A doesn't need to know which services consume the event.

---

## 2. Scalability

Consumers can scale independently.

```text
             Topic
           /   |   \
          ↓    ↓    ↓
         C1   C2   C3
```

---

## 3. Fault Isolation

One consumer failing doesn't necessarily stop the producer.

```text
Order Service
     ↓
   Event
     ↓
  Broker
   /   \
  ↓     ↓
Payment  Email ❌
```

Payment can still continue even if Email is temporarily unavailable.

---

## 4. Easy Extensibility

You can add new consumers without modifying the producer.

```text
Existing:

Order → Event → Payment
              → Inventory

Later:

Order → Event → Payment
              → Inventory
              → Fraud Detection
```

The Order Service doesn't need to change.

---

## 5. Asynchronous Processing

Slow operations can be performed in the background.

```text
User Request
     ↓
Order Service
     ↓
Return Response

        ↓
   Event Processing
        ↓
 Email / Analytics / etc.
```

---

# Disadvantages

## 1. Eventual Consistency

Data isn't necessarily updated everywhere immediately.

## 2. Debugging Is Harder

A single event may trigger many services.

```text
Event
 ├──→ Service A
 ├──→ Service B
 ├──→ Service C
 └──→ Service D
```

Tracing the complete flow requires good observability.

---

## 3. Duplicate Events

At-least-once delivery means consumers must handle duplicates.

## 4. Ordering Problems

Parallel processing makes global ordering difficult.

## 5. Operational Complexity

You need to manage:

* Brokers
* Topics
* Partitions
* Consumer groups
* Retries
* Dead-letter queues
* Monitoring
* Consumer lag

---

# Event-Driven Architecture Example

Consider a food delivery application.

```text
                   User
                    ↓
               Order Service
                    ↓
               ORDER_CREATED
                    ↓
                  Broker
             /      |       \
            ↓       ↓        ↓
       Payment   Restaurant  Notification
         ↓          ↓           ↓
      Process     Prepare      Send
      Payment      Food        Alert
                    |
                    ↓
              DELIVERY_READY
                    ↓
                  Broker
                    ↓
              Delivery Service
```

The services communicate primarily through events rather than direct synchronous calls.

---

# Event-Driven vs Microservices

These concepts are related but **not the same**.

### Microservices

Describes how an application is split into independently deployable services.

```text
Order Service
Payment Service
Inventory Service
```

### Event-Driven Architecture

Describes how those services communicate.

```text
Order
  ↓
Event
  ↓
Broker
  ↓
Payment / Inventory / etc.
```

You can have:

```text
Microservices + REST
```

or:

```text
Microservices + Event-Driven Communication
```

---

# Event-Driven vs Pub/Sub

A useful distinction:

```text
Event-Driven Architecture
        |
        ├── Pub/Sub
        ├── Event Streams
        ├── Message Queues
        └── Event Brokers
```

**Event-driven architecture** is the overall design approach.

**Pub/Sub** is one communication pattern used to implement it.

---

# When Should You Use Event-Driven Architecture?

It works particularly well when:

* Services need to operate independently
* Multiple services react to the same event
* Asynchronous processing is acceptable
* High scalability is required
* Loose coupling is important
* Real-time/event processing is needed

Examples:

* E-commerce
* Banking
* Food delivery
* Ride sharing
* Video platforms
* Analytics systems
* Notification systems

---

# When Should You NOT Use It?

Event-driven architecture can be unnecessary when:

* The system is small
* Operations require immediate synchronous responses
* Strong consistency is required everywhere
* There are only a few simple service interactions
* Operational complexity isn't justified

Don't introduce Kafka or a message broker just because the architecture looks more "distributed."

---

# Mental Model

Think of an event-driven system as:

```text
        Something Happens
                ↓
             EVENT
                ↓
             BROKER
          /     |     \
         ↓      ↓      ↓
      Service Service Service
         ↓      ↓      ↓
       React  React   React
```

The key idea is:

> **Services communicate by publishing and reacting to events rather than directly depending on each other.**

### Core concepts to remember

```text
Event
  ↓
Producer
  ↓
Broker
  ↓
Consumer
  ↓
Action
```

And the main benefits are:

```text
Loose Coupling
      +
Asynchronous Processing
      +
Scalability
      +
Fault Isolation
```
