# APIs and API Design

## What is an API?

**API (Application Programming Interface)** is a set of rules that allows two software components to communicate with each other.

For example:

```text
Frontend
   ↓
   API
   ↓
Backend
   ↓
Database
```

The frontend doesn't need to know how the backend works internally. It only needs to know:

* Which endpoint to call
* What parameters to provide
* What response it will receive
* What errors can occur

---

# Example

Suppose we are building a social media application.

The frontend wants to fetch a user's profile.

It can call:

```http
GET /users/123
```

The server might return:

```json
{
  "id": 123,
  "name": "Anshu",
  "email": "anshu@example.com"
}
```

The API acts as the **contract between the client and server**.

---

# What is REST?

**REST (Representational State Transfer)** is a common architectural style for designing HTTP APIs.

A REST API generally represents entities as **resources**.

For example:

```text
/users
/users/123
/products
/products/123
/orders
/orders/123
```

The HTTP method describes the operation.

---

# HTTP Methods

## GET

Used to retrieve data.

```http
GET /users/123
```

Response:

```json
{
  "id": 123,
  "name": "Anshu"
}
```

GET should generally not modify server state.

---

## POST

Used to create a new resource or trigger an operation.

```http
POST /users
```

Request:

```json
{
  "name": "Anshu",
  "email": "anshu@example.com"
}
```

Response:

```json
{
  "id": 123,
  "name": "Anshu",
  "email": "anshu@example.com"
}
```

---

## PUT

Used to replace/update a resource.

```http
PUT /users/123
```

```json
{
  "name": "Anshu",
  "email": "new@example.com"
}
```

PUT is generally considered **idempotent**.

---

## PATCH

Used to partially update a resource.

```http
PATCH /users/123
```

```json
{
  "email": "new@example.com"
}
```

Only the specified field needs to change.

---

## DELETE

Used to delete a resource.

```http
DELETE /users/123
```

---

# HTTP Status Codes

Status codes communicate the result of an API request.

## 2xx — Success

### 200 OK

Request succeeded.

```http
GET /users/123
→ 200 OK
```

### 201 Created

A resource was successfully created.

```http
POST /users
→ 201 Created
```

### 204 No Content

Request succeeded but there is no response body.

Commonly used for DELETE operations.

---

# 4xx — Client Errors

### 400 Bad Request

The request is invalid.

```json
{
  "error": "Invalid email"
}
```

### 401 Unauthorized

Authentication is required or invalid.

### 403 Forbidden

The user is authenticated but doesn't have permission.

```text
401 → "Who are you?"
403 → "I know who you are, but you can't do this."
```

### 404 Not Found

The requested resource doesn't exist.

```http
GET /users/999999
→ 404 Not Found
```

### 409 Conflict

The request conflicts with the current state.

Example:

```text
Creating a username that already exists.
```

### 429 Too Many Requests

The client has exceeded the rate limit.

---

# 5xx — Server Errors

### 500 Internal Server Error

Unexpected server-side failure.

### 502 Bad Gateway

A gateway/proxy received an invalid response from an upstream service.

### 503 Service Unavailable

The service is temporarily unavailable.

---

# Designing RESTful URLs

A good API should use **resources**, not actions.

### ❌ Bad

```text
/getUser
/createUser
/deleteUser
/getUserOrders
```

### ✅ Better

```text
GET    /users/123
POST   /users
DELETE /users/123
GET    /users/123/orders
```

The HTTP method already describes the action.

---

# Resource Naming

Use nouns rather than verbs.

```text
/users
/orders
/products
/payments
```

Prefer plural resource names:

```text
/users
/orders
/products
```

instead of:

```text
/user
/order
/product
```

Consistency matters more than the exact convention.

---

# Nested Resources

Suppose a user has orders.

You could expose:

```http
GET /users/123/orders
```

This means:

> Get the orders belonging to user 123.

For a specific order:

```http
GET /users/123/orders/456
```

However, don't create deeply nested URLs.

Avoid:

```text
/users/123/orders/456/products/789/reviews/10
```

If the relationship isn't important for the operation, a flatter endpoint is often better:

```text
/reviews/10
```

---

# Query Parameters

Query parameters are useful for filtering, sorting, searching, and pagination.

Example:

```http
GET /products?category=shoes
```

Multiple filters:

```http
GET /products?category=shoes&brand=nike
```

Sorting:

```http
GET /products?sort=price
```

Descending:

```http
GET /products?sort=-price
```

Searching:

```http
GET /products?search=running
```

---

# Pagination

Never return millions of records in one API response.

Instead:

```http
GET /products?page=2&limit=20
```

Response:

```json
{
  "data": [
    ...
  ],
  "page": 2,
  "limit": 20,
  "total": 1000
}
```

---

# Offset vs Cursor Pagination

## Offset Pagination

```http
GET /products?offset=100&limit=20
```

The database skips the first 100 records.

### Advantages

* Simple
* Easy to understand
* Good for smaller datasets

### Problems

For very large datasets, large offsets can become expensive.

---

## Cursor Pagination

Instead of specifying an offset:

```http
GET /products?cursor=eyJpZCI6MTAwfQ==&limit=20
```

The cursor represents the position in the dataset.

Conceptually:

```text
Page 1
 ↓
Cursor
 ↓
Page 2
 ↓
Cursor
 ↓
Page 3
```

Cursor-based pagination is generally better for large or frequently changing datasets.

---

# Filtering, Sorting and Searching

A good API should provide predictable query parameters.

Example:

```http
GET /products
    ?category=electronics
    &minPrice=100
    &maxPrice=1000
    &sort=-rating
    &limit=20
```

This allows the client to control the query without creating many specialized endpoints.

---

# Request Body

Use the request body for data being created or updated.

Example:

```http
POST /orders
Content-Type: application/json
```

```json
{
  "productId": 123,
  "quantity": 2,
  "addressId": 456
}
```

---

# Response Design

Responses should be predictable and consistent.

Example:

```json
{
  "data": {
    "id": 123,
    "name": "Anshu"
  }
}
```

For lists:

```json
{
  "data": [
    {
      "id": 1,
      "name": "Product A"
    },
    {
      "id": 2,
      "name": "Product B"
    }
  ],
  "pagination": {
    "nextCursor": "abc123"
  }
}
```

---

# Error Response Design

Don't return random error formats from different endpoints.

### ❌ Bad

Endpoint A:

```json
{
  "error": "Invalid input"
}
```

Endpoint B:

```json
{
  "message": "User not found"
}
```

Endpoint C:

```json
{
  "err": "Unauthorized"
}
```

### ✅ Better

Use a consistent structure:

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with ID 123 was not found"
  }
}
```

This makes API clients easier to implement.

---

# Authentication vs Authorization

These are different concepts.

## Authentication

> Who are you?

Common approaches:

* Session cookies
* JWT
* OAuth 2.0
* API keys

Example:

```http
Authorization: Bearer <token>
```

---

## Authorization

> What are you allowed to do?

For example:

```text
Admin → Can delete users
User  → Cannot delete users
```

Even if both are authenticated:

```text
User → DELETE /users/123
      → 403 Forbidden
```

---

# API Versioning

APIs evolve.

Suppose version 1 returns:

```json
{
  "name": "Anshu"
}
```

Later you change the response:

```json
{
  "firstName": "Anshu",
  "lastName": "Prakash"
}
```

Existing clients may break.

One solution is versioning:

```text
/api/v1/users
/api/v2/users
```

Another approach is header-based versioning.

The important principle is:

> Don't make breaking changes to an API without considering existing clients.

---

# Idempotency

Idempotency means performing the same request multiple times has the same effect as performing it once.

### GET

Usually idempotent:

```text
GET /users/123
```

Calling it multiple times doesn't change the resource.

### PUT

Generally idempotent:

```text
PUT /users/123
```

Setting the same state repeatedly results in the same state.

### DELETE

Generally idempotent:

```text
DELETE /users/123
```

After the first deletion, subsequent deletions don't recreate the resource.

### POST

Usually **not idempotent**.

```http
POST /payments
```

Sending the request twice could potentially charge the customer twice.

---

# Idempotency Keys

For critical operations such as payments, use an idempotency key.

```http
POST /payments
Idempotency-Key: abc123
```

The server stores:

```text
abc123 → Payment Result
```

If the client retries:

```text
POST /payments
Idempotency-Key: abc123
```

the server recognizes the request and returns the previous result instead of creating another payment.

This is especially important when network failures cause clients to retry requests.

---

# API Rate Limiting

Without rate limiting:

```text
Attacker
   ↓
1,000,000 requests/sec
   ↓
API
   ↓
Server overloaded
```

Rate limiting restricts how many requests a client can make.

Example:

```text
100 requests / minute / user
```

If the limit is exceeded:

```http
429 Too Many Requests
```

Common algorithms include:

* Fixed Window
* Sliding Window
* Token Bucket
* Leaky Bucket

---

# API Gateway

In a microservices architecture, clients shouldn't necessarily communicate directly with every service.

Instead:

```text
Client
  ↓
API Gateway
  ↓
 ┌──────────┬──────────┬──────────┐
 ↓          ↓          ↓
User       Order     Payment
Service    Service    Service
```

The API Gateway can handle:

* Authentication
* Authorization
* Rate limiting
* Routing
* Request validation
* Logging
* Load balancing
* Response aggregation

---

# API Design Example

Suppose we're designing an e-commerce API.

### Users

```http
POST   /users
GET    /users/123
PATCH  /users/123
DELETE /users/123
```

### Products

```http
GET /products
GET /products/123
POST /products
PATCH /products/123
```

### Orders

```http
POST /orders
GET /orders/123
GET /users/123/orders
PATCH /orders/123
```

### Payments

```http
POST /payments
GET /payments/123
```

---

# Designing an API Step-by-Step

When designing an API in a system-design interview, follow this process.

## Step 1 — Identify the Resources

Ask:

> What are the main entities?

For an e-commerce system:

```text
Users
Products
Orders
Payments
```

---

## Step 2 — Identify Operations

For each resource:

```text
Create
Read
Update
Delete
```

Map these to HTTP methods.

```text
POST   → Create
GET    → Read
PUT    → Replace
PATCH  → Partial Update
DELETE → Delete
```

---

## Step 3 — Define URLs

```text
/users
/users/{id}

/products
/products/{id}

/orders
/orders/{id}
```

---

## Step 4 — Define Request/Response

For each endpoint, specify:

```text
Method
URL
Headers
Query parameters
Request body
Response
Status codes
```

Example:

```http
POST /orders
```

Request:

```json
{
  "productId": 123,
  "quantity": 2
}
```

Response:

```json
{
  "id": 456,
  "status": "CREATED"
}
```

---

## Step 5 — Think About Scale

Ask:

* How many requests/sec?
* How large are responses?
* Do we need pagination?
* Do we need caching?
* Do we need rate limiting?
* Can requests be retried?

---

## Step 6 — Think About Failures

Ask:

* What happens if the request times out?
* Can the client safely retry?
* Is the operation idempotent?
* What happens if a downstream service fails?

For payments, for example:

```text
Client
 ↓
POST /payments
 ↓
Timeout ❌
```

The client may retry.

Without idempotency:

```text
Payment 1
Payment 2 ❌
```

With an idempotency key:

```text
Payment 1
Retry
 ↓
Same idempotency key
 ↓
Return existing payment
```

---

# API Design Checklist

When designing an API, think about:

```text
✓ Resources
✓ HTTP methods
✓ URL structure
✓ Request body
✓ Response format
✓ Status codes
✓ Error format
✓ Authentication
✓ Authorization
✓ Pagination
✓ Filtering
✓ Sorting
✓ Rate limiting
✓ Idempotency
✓ Versioning
✓ Caching
✓ Timeouts
✓ Retries
✓ Observability
```

---

# REST API Mental Model

Think:

```text
               API
                |
       ┌────────┼────────┐
       ↓        ↓        ↓
   Resources  Methods  Responses
       ↓        ↓        ↓
    /users     GET      200
    /orders    POST     201
    /products  PATCH    400
                         404
                         500
```

The key principle is:

> **Design APIs as stable contracts between clients and services, with predictable resources, operations, responses, errors, and behavior under failure.**
