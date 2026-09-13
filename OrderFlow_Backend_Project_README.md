# OrderFlow --- Distributed Order Processing Platform

> A backend-first, event-driven order processing platform built to
> demonstrate production-oriented backend engineering, distributed
> systems, reliability patterns, cloud deployment, and observability.

------------------------------------------------------------------------

## 1. Project Overview

**OrderFlow** is a backend-heavy distributed commerce/order-processing
platform.

The purpose of this project is **not** to build a visually impressive
e-commerce website. The purpose is to demonstrate how a backend engineer
designs and builds a system that remains reliable when requests are
duplicated, services fail, databases are slow, messages are delayed,
workers crash, and traffic increases.

The system models a simplified order lifecycle:

``` text
Client
  |
  v
API Gateway
  |
  v
Order Service
  |
  +----------------------+
  |                      |
  v                      v
PostgreSQL             Event/Message Bus
                           |
              +------------+------------+-------------+
              |            |            |             |
              v            v            v             v
           Payment      Inventory   Notification   Shipment
           Worker        Worker        Worker        Worker
              |            |            |             |
              +------------+------------+-------------+
                           |
                           v
                       Order State
```

The project should start as a well-structured modular backend and evolve
into an event-driven distributed architecture.

------------------------------------------------------------------------

# 2. Primary Goals

The project should demonstrate:

-   REST API design
-   Java and Spring Boot
-   PostgreSQL and relational database design
-   ACID transactions
-   Concurrent inventory updates
-   Authentication and authorization
-   Redis caching
-   Asynchronous processing
-   Kafka and/or AWS SQS
-   Event-driven architecture
-   Idempotency
-   Transactional Outbox Pattern
-   Retry and exponential backoff
-   Dead Letter Queue
-   Circuit breaker and timeout handling
-   Distributed locking where appropriate
-   Eventual consistency
-   Observability
-   Structured logging
-   Metrics
-   Distributed tracing
-   Docker
-   CI/CD
-   AWS deployment
-   Infrastructure/configuration management
-   Automated testing
-   Load testing
-   Failure testing

------------------------------------------------------------------------

# 3. Engineering Philosophy

Do **not** build unnecessary microservices simply to say
"microservices."

The project should evolve in stages.

### Stage 1

Build a clean modular Spring Boot application.

### Stage 2

Add authentication, authorization, validation, transactions and Redis.

### Stage 3

Introduce asynchronous events.

### Stage 4

Separate independent workloads into services/workers.

### Stage 5

Add reliability patterns.

### Stage 6

Deploy the system to AWS.

This makes the architecture understandable and gives a clear engineering
story during interviews.

------------------------------------------------------------------------

# 4. Suggested Technology Stack

## Backend

-   Java 17+
-   Spring Boot 3.x
-   Spring Web
-   Spring Data JPA
-   Spring Security
-   Bean Validation
-   Spring Actuator
-   Maven

## Database

Primary:

-   PostgreSQL

Optional:

-   DynamoDB for selected workloads

## Cache

-   Redis

## Messaging

Local development:

-   Apache Kafka

AWS deployment:

-   Amazon SQS/SNS where appropriate

The implementation should keep messaging behind an abstraction so that
local Kafka and AWS messaging can be swapped without rewriting business
logic.

## Infrastructure

-   Docker
-   Docker Compose
-   AWS
-   GitHub Actions

## API Documentation

-   OpenAPI
-   Swagger UI

## Testing

-   JUnit 5
-   Mockito
-   Spring Boot Test
-   Testcontainers
-   REST Assured
-   JMeter or k6 for load testing

## Observability

-   Spring Boot Actuator
-   Micrometer
-   CloudWatch
-   OpenTelemetry where practical

------------------------------------------------------------------------

# 5. Core Functional Requirements

## 5.1 User Management

Users should be able to:

-   Register
-   Login
-   View profile
-   Update profile
-   View order history

Roles:

``` text
CUSTOMER
ADMIN
```

Optional future roles:

``` text
SUPPORT
WAREHOUSE
```

------------------------------------------------------------------------

# 6. Product Management

Products should contain:

``` text
id
sku
name
description
price
currency
category
status
createdAt
updatedAt
```

Possible statuses:

``` text
ACTIVE
INACTIVE
OUT_OF_STOCK
DISCONTINUED
```

Admin APIs:

``` http
POST   /api/v1/products
PUT    /api/v1/products/{id}
DELETE /api/v1/products/{id}
```

Customer APIs:

``` http
GET /api/v1/products
GET /api/v1/products/{id}
```

Use pagination and filtering.

Example:

``` http
GET /api/v1/products?page=0&size=20&category=electronics
```

------------------------------------------------------------------------

# 7. Cart Management

Customers can:

``` http
POST   /api/v1/cart/items
GET    /api/v1/cart
PUT    /api/v1/cart/items/{productId}
DELETE /api/v1/cart/items/{productId}
DELETE /api/v1/cart
```

The cart may use Redis for fast access.

However, the final order must store the actual price at purchase time.

Do not depend on the cart price after order creation.

------------------------------------------------------------------------

# 8. Order Management

Core APIs:

``` http
POST /api/v1/orders
GET  /api/v1/orders/{orderId}
GET  /api/v1/users/me/orders
POST /api/v1/orders/{orderId}/cancel
```

Order states:

``` text
PENDING
PAYMENT_PENDING
PAYMENT_CONFIRMED
INVENTORY_RESERVED
CONFIRMED
PROCESSING
SHIPPED
DELIVERED
CANCELLED
FAILED
```

The state transitions must be validated.

For example:

``` text
PENDING
   |
   v
PAYMENT_PENDING
   |
   v
PAYMENT_CONFIRMED
   |
   v
INVENTORY_RESERVED
   |
   v
CONFIRMED
   |
   v
PROCESSING
   |
   v
SHIPPED
   |
   v
DELIVERED
```

Invalid transitions must be rejected.

------------------------------------------------------------------------

# 9. Payment Processing

This project should use a **mock payment provider**.

Do NOT integrate real financial transactions.

Example:

``` http
POST /api/v1/payments/{orderId}
```

Payment states:

``` text
PENDING
PROCESSING
SUCCESS
FAILED
REFUNDED
```

The payment worker should process payment asynchronously.

Example event:

``` json
{
  "eventId": "uuid",
  "eventType": "PaymentRequested",
  "orderId": "uuid",
  "userId": "uuid",
  "amount": 2499.00,
  "currency": "INR",
  "timestamp": "2026-09-13T10:00:00Z"
}
```

------------------------------------------------------------------------

# 10. Inventory Management

Inventory is the most important concurrency component.

Inventory:

``` text
productId
availableQuantity
reservedQuantity
version
updatedAt
```

Example:

``` text
Product A
Available = 10
Reserved  = 0
```

If two customers simultaneously purchase:

``` text
Customer A → 7
Customer B → 6
```

the system must NOT allow the inventory to become negative.

The implementation should demonstrate either:

-   Optimistic locking
-   Pessimistic locking

Prefer optimistic locking initially.

------------------------------------------------------------------------

# 11. Idempotency

Order creation must support an idempotency key.

Example:

``` http
POST /api/v1/orders
Idempotency-Key: 7e6f4c1a-...
```

If the client sends the same request multiple times:

``` text
Request 1 → Create Order #1001
Request 2 → Return Order #1001
Request 3 → Return Order #1001
```

Never create duplicate orders.

Suggested table:

``` text
idempotency_keys

id
key
user_id
request_hash
response_status
response_body
created_at
expires_at
```

Explain why idempotency is required for unreliable networks and client
retries.

------------------------------------------------------------------------

# 12. Transactional Outbox Pattern

This is a key architecture requirement.

Problem:

``` text
Database transaction succeeds
        |
        v
Publish Kafka event
        |
        X
Kafka fails
```

Now the database says the order exists but downstream services never
receive the event.

Solution:

``` text
Database Transaction
       |
       +--> orders
       |
       +--> outbox_events
```

A separate publisher reads:

``` text
outbox_events
```

and publishes events to Kafka/SQS.

Example:

``` text
Create Order
    |
    +--> orders
    |
    +--> outbox_events
             |
             v
       Event Publisher
             |
             v
         Message Bus
```

Outbox event:

``` text
id
aggregateType
aggregateId
eventType
payload
status
createdAt
publishedAt
retryCount
```

------------------------------------------------------------------------

# 13. Event-Driven Architecture

Important events:

``` text
OrderCreated
PaymentRequested
PaymentSucceeded
PaymentFailed
InventoryReservationRequested
InventoryReserved
InventoryReservationFailed
OrderConfirmed
OrderCancelled
ShipmentRequested
ShipmentCreated
NotificationRequested
```

Example:

``` text
OrderCreated
     |
     v
Message Bus
     |
     +----> Payment Worker
     |
     +----> Inventory Worker
     |
     +----> Notification Worker
```

Events should contain:

-   event ID
-   event type
-   aggregate ID
-   timestamp
-   correlation ID
-   payload
-   schema version

Example:

``` json
{
  "eventId": "uuid",
  "eventType": "OrderCreated",
  "schemaVersion": 1,
  "aggregateId": "order-123",
  "correlationId": "request-456",
  "timestamp": "2026-09-13T10:00:00Z",
  "payload": {}
}
```

------------------------------------------------------------------------

# 14. Retry Strategy

Transient failures should be retried.

Example:

``` text
Attempt 1
   |
 failure
   |
 wait 1s
   |
Attempt 2
   |
 failure
   |
 wait 2s
   |
Attempt 3
   |
 failure
   |
 wait 4s
   |
Attempt 4
   |
 failure
   |
DLQ
```

Use exponential backoff.

Do not retry indefinitely.

------------------------------------------------------------------------

# 15. Dead Letter Queue

Messages that repeatedly fail should go to a DLQ.

``` text
Main Queue
    |
    v
Worker
    |
    +---- success
    |
    +---- failure
             |
             v
          Retry
             |
             v
          Failure
             |
             v
            DLQ
```

The system should expose an admin endpoint to inspect failed messages.

Example:

``` http
GET /api/v1/admin/dead-letters
```

------------------------------------------------------------------------

# 16. Duplicate Message Handling

At-least-once delivery means a consumer may receive the same message
more than once.

Example:

``` text
PaymentSucceeded
PaymentSucceeded
```

The consumer must not perform the payment-success side effect twice.

Create an inbox/processed-events table:

``` text
processed_events

event_id
consumer_name
processed_at
```

Before processing:

``` text
if event already exists:
    ignore
else:
    process
    save event ID
```

This demonstrates the difference between:

``` text
At-most-once
At-least-once
Exactly-once
```

Be precise in documentation: application-level exactly-once behavior is
usually achieved through idempotent processing rather than assuming the
transport itself provides global exactly-once semantics.

------------------------------------------------------------------------

# 17. Redis Caching

Use Redis for frequently accessed data.

Good candidates:

``` text
Product details
Product listings
Cart
Rate-limit counters
Short-lived idempotency data
```

Use cache-aside:

``` text
Request
  |
  v
Redis
  |
  +---- HIT ---> return
  |
  +---- MISS
          |
          v
      PostgreSQL
          |
          v
       Redis
          |
          v
       Response
```

Consider:

-   TTL
-   cache invalidation
-   stale data
-   cache stampede

Do not cache data blindly.

------------------------------------------------------------------------

# 18. Rate Limiting

Implement API rate limiting.

Example:

``` text
Public product API:
100 requests/minute

Login:
10 requests/minute

Order creation:
20 requests/minute
```

Redis can be used for distributed counters.

Possible algorithms:

-   Fixed Window
-   Sliding Window
-   Token Bucket

Start with one implementation and document the trade-offs.

------------------------------------------------------------------------

# 19. Authentication and Authorization

Use:

``` text
Spring Security
JWT
BCrypt/Argon2 password hashing
```

Example:

``` text
POST /api/v1/auth/register
POST /api/v1/auth/login
```

JWT should contain:

``` text
userId
role
issuedAt
expiration
```

Authorization examples:

``` text
CUSTOMER → create/view own orders

ADMIN → manage products/inventory

ADMIN → inspect DLQ
```

Never store plaintext passwords.

Never commit secrets to Git.

------------------------------------------------------------------------

# 20. Database Design

Recommended PostgreSQL tables:

``` text
users
roles
products
categories
inventory
carts
cart_items
orders
order_items
payments
shipments
idempotency_keys
outbox_events
processed_events
audit_logs
```

Relationships:

``` text
User
 |
 +---- Orders
 |       |
 |       +---- OrderItems ---- Product
 |
 +---- Cart
         |
         +---- CartItems ---- Product
```

Use:

-   Primary keys
-   Foreign keys
-   Unique constraints
-   Check constraints
-   Indexes
-   Proper transaction boundaries

Important indexes should be documented.

Examples:

``` sql
CREATE INDEX idx_orders_user_id
ON orders(user_id);

CREATE INDEX idx_orders_created_at
ON orders(created_at);

CREATE UNIQUE INDEX idx_idempotency_key
ON idempotency_keys(user_id, key);
```

------------------------------------------------------------------------

# 21. Transaction Boundaries

Transactions should protect business invariants.

Example:

``` text
BEGIN

Validate order
Create order
Create order items
Create outbox event

COMMIT
```

Do not keep database transactions open while making slow network calls.

Bad:

``` text
BEGIN TRANSACTION
   |
   v
Call Payment API
   |
   v
Wait 5 seconds
   |
   v
Update database
COMMIT
```

Better:

``` text
Create local state
Commit
   |
   v
Publish event
   |
   v
Async worker
```

------------------------------------------------------------------------

# 22. Concurrency

The project must explicitly address concurrent inventory updates.

Example:

``` text
Stock = 10

Request A → buy 8
Request B → buy 5
```

Expected:

``` text
Only one succeeds
or
both succeed only if enough stock exists
```

Never:

``` text
Stock = -3
```

Demonstrate the chosen locking strategy and explain why.

------------------------------------------------------------------------

# 23. API Design Principles

Use:

``` text
/api/v1/...
```

Use appropriate HTTP status codes:

``` text
200 OK
201 CREATED
202 ACCEPTED
400 BAD REQUEST
401 UNAUTHORIZED
403 FORBIDDEN
404 NOT FOUND
409 CONFLICT
422 UNPROCESSABLE ENTITY
429 TOO MANY REQUESTS
500 INTERNAL SERVER ERROR
```

For asynchronous operations, consider:

``` text
202 ACCEPTED
```

rather than pretending the entire workflow completed synchronously.

Use a standard error response:

``` json
{
  "timestamp": "2026-09-13T10:00:00Z",
  "status": 409,
  "error": "CONFLICT",
  "message": "Insufficient inventory",
  "path": "/api/v1/orders",
  "traceId": "abc-123"
}
```

------------------------------------------------------------------------

# 24. Correlation IDs

Every request should have a correlation ID.

``` text
Client Request
      |
      v
Correlation ID: abc123
      |
      +--> Order Service
      |
      +--> Payment Worker
      |
      +--> Inventory Worker
      |
      +--> Notification Worker
```

This makes debugging distributed workflows much easier.

------------------------------------------------------------------------

# 25. Observability

Expose:

``` text
/actuator/health
/actuator/metrics
```

Track:

### Application

``` text
Request count
Request latency
Error rate
HTTP status distribution
```

### Business

``` text
Orders created
Orders failed
Payments succeeded
Payments failed
Inventory reservation failures
```

### Infrastructure

``` text
Database latency
Connection pool usage
Queue depth
Consumer lag
Redis hit ratio
CPU
Memory
```

------------------------------------------------------------------------

# 26. Logging

Use structured JSON logs where practical.

Example:

``` json
{
  "timestamp": "2026-09-13T10:00:00Z",
  "level": "INFO",
  "service": "order-service",
  "correlationId": "abc123",
  "orderId": "order-1001",
  "event": "ORDER_CREATED"
}
```

Never log:

-   passwords
-   JWT secrets
-   API keys
-   payment credentials
-   sensitive personal information

------------------------------------------------------------------------

# 27. Testing Strategy

Testing should be treated as a first-class feature.

## Unit Tests

Test:

``` text
Order state transitions
Inventory calculations
Pricing
Idempotency
Validation
Retry logic
```

## Integration Tests

Use Testcontainers for:

``` text
PostgreSQL
Redis
Kafka
```

Test real infrastructure interactions.

## API Tests

Test:

``` text
Authentication
Authorization
Validation
HTTP status codes
Error responses
Pagination
```

## Concurrency Tests

Explicitly test:

``` text
Two users buying the last item
```

## Failure Tests

Simulate:

``` text
Database unavailable
Redis unavailable
Message consumer crash
Payment failure
Duplicate message
Timeout
```

------------------------------------------------------------------------

# 28. Load Testing

Use k6 or JMeter.

Test scenarios:

``` text
100 concurrent users
500 concurrent users
1000 concurrent requests
```

Measure:

``` text
Average latency
p95 latency
p99 latency
Requests/sec
Error rate
Database usage
Queue depth
```

Do not claim scalability numbers without actually measuring them.

------------------------------------------------------------------------

# 29. Docker Architecture

Local development should run using:

``` text
docker-compose.yml
```

Example:

``` text
orderflow
 |
 +-- order-service
 |
 +-- payment-service
 |
 +-- inventory-service
 |
 +-- notification-service
 |
 +-- postgres
 |
 +-- redis
 |
 +-- kafka
 |
 +-- zookeeper / kafka controller
```

Each service should have its own Dockerfile.

Use multi-stage builds.

Example concept:

``` text
Build image
     |
     v
Compile Java
     |
     v
Runtime image
     |
     v
Run Spring Boot
```

Keep runtime images small.

------------------------------------------------------------------------

# 30. AWS Deployment

The project should have a cost-conscious AWS deployment.

A practical target architecture is:

``` text
                    Internet
                       |
                       v
                Amazon API Gateway
                       |
                       v
                Spring Boot App
                       |
          +------------+-------------+
          |            |             |
          v            v             v
       RDS/DB        Redis          SQS
                                     |
                          +----------+----------+
                          |          |          |
                          v          v          v
                       Worker     Worker     Worker
```

Potential AWS services:

``` text
API Gateway
ECS/Fargate or another suitable compute option
RDS PostgreSQL
SQS
SNS
CloudWatch
ECR
IAM
VPC
Security Groups
Secrets Manager / Parameter Store
```

The exact AWS architecture should be selected based on current AWS Free
Tier eligibility, regional pricing, account type and expected usage.

**Important:** always verify current AWS pricing/free-tier eligibility
before deploying. Add billing alerts and destroy unused resources.

------------------------------------------------------------------------

# 31. AWS Security

Follow least privilege.

Use:

``` text
IAM roles
Security Groups
Private subnets where appropriate
Environment variables
Secrets Manager / Parameter Store
HTTPS
```

Never:

``` text
Hard-code AWS credentials
Commit .env files
Commit passwords
Commit JWT secrets
Expose PostgreSQL publicly
```

------------------------------------------------------------------------

# 32. CI/CD

Use GitHub Actions.

Pipeline:

``` text
Git Push
   |
   v
Build
   |
   v
Unit Tests
   |
   v
Integration Tests
   |
   v
Static Analysis
   |
   v
Build Docker Image
   |
   v
Push Image to ECR
   |
   v
Deploy
   |
   v
Health Check
```

The pipeline should fail if tests fail.

------------------------------------------------------------------------

# 33. Suggested Repository Structure

A monorepo is recommended initially:

``` text
orderflow/
│
├── services/
│   ├── order-service/
│   ├── payment-service/
│   ├── inventory-service/
│   ├── notification-service/
│   └── shipment-service/
│
├── libs/
│   ├── event-contracts/
│   ├── common-security/
│   └── common-observability/
│
├── infrastructure/
│   ├── docker/
│   ├── aws/
│   └── scripts/
│
├── docs/
│   ├── architecture/
│   ├── api/
│   ├── database/
│   └── decisions/
│
├── tests/
│   ├── integration/
│   └── load/
│
├── .github/
│   └── workflows/
│
├── docker-compose.yml
├── README.md
└── .gitignore
```

------------------------------------------------------------------------

# 34. Service Responsibilities

## Order Service

Owns:

``` text
Orders
Order Items
Order State
Idempotency
Order APIs
Outbox Events
```

## Payment Service

Owns:

``` text
Payment state
Payment processing
Payment retries
```

## Inventory Service

Owns:

``` text
Stock
Reservations
Concurrency control
```

## Notification Service

Owns:

``` text
Email/SMS simulation
Notification state
Retry handling
```

## Shipment Service

Owns:

``` text
Shipment creation
Tracking state
```

------------------------------------------------------------------------

# 35. Service Communication

Use synchronous REST only where immediate response is genuinely
required.

Use asynchronous events for:

``` text
Payment processing
Inventory processing
Notifications
Shipment creation
Analytics
```

General rule:

``` text
Query → synchronous

Command with long-running processing → asynchronous
```

------------------------------------------------------------------------

# 36. Reliability Patterns

The final project should demonstrate:

### Timeout

Never wait indefinitely for another service.

### Retry

Retry transient failures.

### Exponential Backoff

Increase delay between retries.

### Circuit Breaker

Stop repeatedly calling an unhealthy dependency.

### Bulkhead

Prevent one workload from exhausting all resources.

### Idempotency

Prevent duplicate side effects.

### Outbox

Prevent DB/event publication inconsistency.

### Inbox / Processed Events

Prevent duplicate message side effects.

### DLQ

Capture messages that cannot be processed.

------------------------------------------------------------------------

# 37. Architecture Decision Records

Create ADR documents for important decisions.

Examples:

``` text
ADR-001 Why PostgreSQL?
ADR-002 Why Kafka locally?
ADR-003 Why SQS in AWS?
ADR-004 Why Outbox Pattern?
ADR-005 Optimistic vs Pessimistic Locking
ADR-006 Redis Cache Strategy
ADR-007 Monolith-to-Microservices Evolution
```

Each ADR should explain:

``` text
Context
Decision
Alternatives
Trade-offs
Consequences
```

------------------------------------------------------------------------

# 38. Important System Design Questions

The README should eventually answer:

1.  What happens when payment succeeds but inventory reservation fails?
2.  What happens when the same order request is sent twice?
3.  What happens when Kafka/SQS is temporarily unavailable?
4.  What happens when a consumer crashes after processing but before
    acknowledging a message?
5.  How are duplicate events handled?
6.  How is inventory protected against race conditions?
7.  Why use PostgreSQL instead of MongoDB?
8.  Where should Redis be used?
9.  Why use asynchronous processing?
10. How would the system scale to 10x traffic?
11. What is the database bottleneck?
12. What happens if Redis goes down?
13. What happens if the payment service is unavailable?
14. How do you monitor queue lag?
15. How do you debug a request across multiple services?
16. How would you partition Kafka topics?
17. What delivery guarantee does the system provide?
18. How would you deploy a new service version without downtime?
19. How would you handle schema evolution?
20. How would you handle a database migration in production?

These questions should be answered in `/docs/system-design.md`.

------------------------------------------------------------------------

# 39. Failure Scenarios

The project should intentionally test:

``` text
Scenario 1:
Payment service crashes.

Scenario 2:
Inventory service crashes.

Scenario 3:
Kafka/SQS unavailable.

Scenario 4:
Duplicate OrderCreated event.

Scenario 5:
Database connection lost.

Scenario 6:
Redis unavailable.

Scenario 7:
Worker crashes during processing.

Scenario 8:
Network timeout.

Scenario 9:
Two customers purchase the last product.

Scenario 10:
Client retries the same order request.
```

For every failure, document:

``` text
Failure
Detection
Recovery
User impact
Data consistency
Retry behavior
```

------------------------------------------------------------------------

# 40. Development Roadmap

## Phase 1 --- Foundation

-   Create Spring Boot project
-   PostgreSQL
-   Flyway/Liquibase migrations
-   User model
-   Product model
-   Order model
-   REST APIs
-   Validation
-   Exception handling
-   Swagger

## Phase 2 --- Security

-   Spring Security
-   JWT
-   Roles
-   Password hashing
-   Authorization

## Phase 3 --- Order Workflow

-   Cart
-   Order creation
-   Order state machine
-   Transactions
-   Inventory

## Phase 4 --- Concurrency

-   Optimistic locking
-   Race-condition tests
-   Inventory reservation

## Phase 5 --- Redis

-   Product cache
-   Cart cache
-   Rate limiting

## Phase 6 --- Messaging

-   Kafka locally
-   Event contracts
-   Consumers
-   Producer/consumer error handling

## Phase 7 --- Reliability

-   Idempotency
-   Outbox
-   Inbox/processed events
-   Retry
-   Backoff
-   DLQ
-   Circuit breaker
-   Timeouts

## Phase 8 --- Observability

-   Actuator
-   Metrics
-   Structured logs
-   Correlation IDs
-   Tracing

## Phase 9 --- Docker

-   Dockerfiles
-   Docker Compose
-   Local production-like environment

## Phase 10 --- AWS

-   ECR
-   Compute
-   RDS
-   SQS/SNS
-   API Gateway
-   IAM
-   CloudWatch
-   Secrets
-   Networking

## Phase 11 --- CI/CD

-   GitHub Actions
-   Test pipeline
-   Docker build
-   Image publishing
-   Deployment
-   Health checks

## Phase 12 --- Performance

-   Load testing
-   Bottleneck analysis
-   Database indexes
-   Redis tuning
-   Queue tuning

------------------------------------------------------------------------

# 41. Definition of Done

The project is considered complete only when:

-   [ ] Authentication works
-   [ ] Authorization works
-   [ ] Product APIs work
-   [ ] Cart works
-   [ ] Orders work
-   [ ] Inventory concurrency is safe
-   [ ] Payment workflow works
-   [ ] Events are published
-   [ ] Consumers process events
-   [ ] Duplicate messages are safe
-   [ ] Idempotency works
-   [ ] Outbox works
-   [ ] Retries work
-   [ ] DLQ works
-   [ ] Redis caching works
-   [ ] Rate limiting works
-   [ ] Structured logs exist
-   [ ] Metrics exist
-   [ ] Correlation IDs exist
-   [ ] Integration tests exist
-   [ ] Concurrency tests exist
-   [ ] Docker setup works
-   [ ] CI/CD works
-   [ ] AWS deployment works
-   [ ] Cloud monitoring works
-   [ ] Architecture documentation exists
-   [ ] Failure scenarios are documented
-   [ ] Load testing has been performed
-   [ ] No secrets are committed
-   [ ] AWS resources are monitored for cost

------------------------------------------------------------------------

# 42. Resume Description

Do not use numbers unless they are actually measured.

Possible final resume bullet style:

``` text
Built an event-driven distributed order-processing platform using Java,
Spring Boot, PostgreSQL, Redis and Kafka/SQS, supporting asynchronous
payment, inventory, notification and shipment workflows.

Implemented idempotent APIs, transactional outbox/inbox patterns,
optimistic locking, retry with exponential backoff, DLQs and circuit
breaking to improve reliability under duplicate requests and service
failures.

Containerized services with Docker and automated CI/CD using GitHub
Actions, deploying the platform on AWS with centralized monitoring,
logging and metrics.
```

After benchmarking, replace generic claims with real measurements.

------------------------------------------------------------------------

# 43. GitHub README Expectations

The public GitHub repository should contain:

``` text
Architecture Diagram
System Design
Database ER Diagram
API Documentation
Sequence Diagrams
ADR Documents
Local Setup
AWS Deployment Guide
Testing Guide
Load Testing Results
Failure Testing Results
Screenshots of Observability
Cost/Safety Notes
```

The README should prioritize engineering decisions over UI screenshots.

------------------------------------------------------------------------

# 44. Recommended Sequence Diagrams

Create diagrams for:

### Successful Order

``` text
Client
 |
 | POST /orders
 v
Order Service
 |
 | create order + outbox
 v
PostgreSQL
 |
 | publish OrderCreated
 v
Message Bus
 |
 +--> Payment
 |
 +--> Inventory
 |
 +--> Notification
```

### Duplicate Request

``` text
Client
 |
 | request + idempotency key
 v
Order Service
 |
 | lookup key
 v
Existing Order
 |
 v
Return existing response
```

### Payment Failure

``` text
Payment Worker
      |
      v
Payment Provider
      |
      X
Failure
      |
      v
Retry
      |
      v
Retry
      |
      v
DLQ
```

### Inventory Race Condition

Document two simultaneous customers attempting to reserve the same final
stock unit.

------------------------------------------------------------------------

# 45. What Makes This Project Senior-Level

The project is not senior-level because it has many services.

It becomes senior-level when it demonstrates that you understand:

``` text
Correctness
    +
Consistency
    +
Concurrency
    +
Reliability
    +
Scalability
    +
Observability
    +
Security
    +
Operational trade-offs
```

The interviewer should be able to ask:

> "What happens if this component fails?"

and you should have a documented answer.

------------------------------------------------------------------------

# 46. AI-Assisted Development Rules

AI may be used during development, but it should not blindly generate
the whole application.

For every generated component:

1.  Understand the design.
2.  Review the code.
3.  Write/verify tests.
4.  Check transaction boundaries.
5.  Check failure handling.
6.  Check security.
7.  Check database indexes.
8.  Check concurrency.
9.  Check resource usage.
10. Document the architectural decision.

Never accept:

``` text
"Looks correct"
```

without testing it.

------------------------------------------------------------------------

# 47. MASTER AI PROMPT

Copy the following prompt into an AI coding assistant.

``` text
You are acting as a Principal Backend Engineer, Distributed Systems
Engineer, Cloud Architect and strict code reviewer.

I am building a backend-heavy project called "OrderFlow".

OrderFlow is a distributed order-processing platform designed to
demonstrate production-oriented backend engineering rather than frontend
development.

PRIMARY STACK

- Java 17+
- Spring Boot 3.x
- Spring Web
- Spring Security
- Spring Data JPA
- PostgreSQL
- Redis
- Apache Kafka for local development
- AWS SQS/SNS for AWS-oriented deployment where appropriate
- Docker
- GitHub Actions
- AWS
- OpenAPI/Swagger
- JUnit 5
- Mockito
- Testcontainers
- REST Assured
- k6 or JMeter
- Spring Actuator
- Micrometer
- OpenTelemetry where practical

CORE SERVICES

1. Order Service
2. Payment Service
3. Inventory Service
4. Notification Service
5. Shipment Service

The system should evolve gradually. Do NOT immediately create an
over-engineered microservice architecture.

Start with a clean modular implementation and introduce distributed
components only when there is a clear architectural reason.

CORE REQUIREMENTS

The system must support:

- User registration/login
- JWT authentication
- Role-based authorization
- Product management
- Product search/filtering/pagination
- Cart management
- Order creation
- Order cancellation
- Order state management
- Mock payment processing
- Inventory reservation
- Shipment creation
- Notification processing

BACKEND ENGINEERING REQUIREMENTS

The project must demonstrate:

- REST API design
- ACID transactions
- Proper transaction boundaries
- Optimistic locking for inventory
- Concurrency handling
- Idempotent APIs
- Transactional Outbox Pattern
- Inbox/processed-event pattern
- Event-driven architecture
- Kafka/SQS messaging
- Retry with exponential backoff
- Dead Letter Queues
- Circuit breakers
- Timeouts
- Redis caching
- Rate limiting
- Structured logging
- Correlation IDs
- Metrics
- Distributed tracing
- Health checks
- Automated tests
- Docker
- CI/CD
- AWS deployment

IMPORTANT ENGINEERING PRINCIPLES

Do not add technologies just because they are popular.

For every architectural decision, explain:

1. What problem does this solve?
2. Why is this technology/pattern appropriate?
3. What alternatives exist?
4. What are the trade-offs?
5. What failure scenarios must be handled?
6. How would this behave under concurrency?
7. How would this scale?

DATABASE

Use PostgreSQL as the source of truth.

Suggested tables:

users
roles
products
categories
inventory
carts
cart_items
orders
order_items
payments
shipments
idempotency_keys
outbox_events
processed_events
audit_logs

Use:

- Primary keys
- Foreign keys
- Unique constraints
- Check constraints
- Proper indexes
- Database migrations

Never store sensitive data unnecessarily.

Never store passwords in plaintext.

ORDER STATE MACHINE

Use states such as:

PENDING
PAYMENT_PENDING
PAYMENT_CONFIRMED
INVENTORY_RESERVED
CONFIRMED
PROCESSING
SHIPPED
DELIVERED
CANCELLED
FAILED

Do not allow arbitrary state transitions.

Define valid transitions explicitly.

IDEMPOTENCY

POST /api/v1/orders must support:

Idempotency-Key: <unique-key>

If the client retries the same request, the server must return the
previous result rather than creating a duplicate order.

Implement this safely using PostgreSQL.

OUTBOX

When creating an order:

- Update order state
- Create the required outbox event

These operations must happen in the same database transaction.

A background publisher should publish outbox events to the message bus.

Do not perform remote network calls inside the main database transaction.

MESSAGING

Events should contain:

- eventId
- eventType
- schemaVersion
- aggregateId
- correlationId
- timestamp
- payload

Consumers must be idempotent.

Assume at-least-once message delivery.

Do not assume that duplicate messages cannot happen.

INVENTORY CONCURRENCY

Example:

Stock = 1

Customer A buys 1.
Customer B buys 1 at almost exactly the same time.

The system must never allow:

availableQuantity < 0

Write concurrency tests proving the behavior.

RETRY

Transient failures should use:

- limited retry count
- exponential backoff
- jitter where appropriate
- DLQ after maximum attempts

Do not retry permanent failures indefinitely.

REDIS

Use Redis for appropriate high-read/short-lived data such as:

- Product cache
- Cart
- Rate limiting
- Short-lived idempotency information where appropriate

Use cache-aside where suitable.

Document invalidation strategy.

SECURITY

Use:

- Spring Security
- JWT
- secure password hashing
- role-based authorization
- validation
- secure headers where appropriate

Never hard-code secrets.

Never commit .env files.

Never expose database credentials.

AWS credentials must never be committed to Git.

OBSERVABILITY

Every request should have a correlation ID.

Logs should contain useful fields such as:

timestamp
level
service
correlationId
requestId
orderId
eventType

Never log passwords, tokens, credentials or secrets.

Expose health and metrics endpoints.

Track:

- request latency
- error rate
- order throughput
- payment failures
- inventory failures
- queue depth
- consumer lag
- database latency
- Redis hit rate

TESTING

For every feature, provide:

1. Unit tests
2. Integration tests where infrastructure is involved
3. API tests
4. Failure tests where appropriate
5. Concurrency tests where appropriate

Use Testcontainers for PostgreSQL, Redis and Kafka when useful.

Do not mock everything.

Use real infrastructure in integration tests where behavior depends
on infrastructure semantics.

DOCKER

Provide:

- Dockerfile for each service
- docker-compose.yml for local development
- PostgreSQL
- Redis
- Kafka
- application services

Use multi-stage builds where appropriate.

CI/CD

Create a GitHub Actions pipeline:

Git Push
→ Build
→ Unit Tests
→ Integration Tests
→ Static Analysis
→ Docker Build
→ Publish Image
→ Deploy
→ Health Check

The pipeline must fail if tests fail.

AWS

Design a cost-conscious deployment using appropriate AWS services.

Potential services include:

- API Gateway
- ECS/Fargate or another appropriate compute service
- RDS PostgreSQL
- SQS
- SNS
- ECR
- CloudWatch
- IAM
- VPC
- Security Groups
- Secrets Manager or Parameter Store

Before recommending a specific AWS architecture, verify current AWS
Free Tier eligibility and pricing assumptions if web access is available.

Do not promise that all services are permanently free.

Recommend billing alerts and cleanup procedures.

DOCUMENTATION

For every major architectural decision, create an ADR.

Examples:

ADR-001 PostgreSQL
ADR-002 Kafka/SQS
ADR-003 Transactional Outbox
ADR-004 Optimistic Locking
ADR-005 Redis Cache
ADR-006 Microservice Boundaries

Each ADR should include:

Context
Decision
Alternatives
Trade-offs
Consequences

CODING STYLE

Use:

- Clean Architecture principles where appropriate
- SOLID
- Dependency Injection
- Small cohesive classes
- Clear domain naming
- DTOs for API boundaries
- Global exception handling
- Validation
- Proper logging
- Meaningful comments only
- No unnecessary abstraction

Avoid:

- giant service classes
- generic "Utils" dumping grounds
- unnecessary interfaces
- premature microservices
- duplicated business logic
- business logic inside controllers
- database calls from controllers
- swallowing exceptions
- magic numbers
- hard-coded configuration

PROJECT STRUCTURE

Prefer a structure similar to:

orderflow/
services/
  order-service/
  payment-service/
  inventory-service/
  notification-service/
  shipment-service/

libs/
  event-contracts/
  common-security/
  common-observability/

infrastructure/
  docker/
  aws/
  scripts/

docs/
  architecture/
  api/
  database/
  decisions/

tests/
  integration/
  load/

.github/
  workflows/

Do not create files without explaining their purpose.

IMPORTANT WORKFLOW

We will build the system incrementally.

For each phase:

1. Explain the architecture briefly.
2. Show the folder structure.
3. Identify the files to create/change.
4. Provide complete code.
5. Explain the important code decisions.
6. Provide database migrations.
7. Provide tests.
8. Explain how to run it.
9. Explain how to verify it.
10. Identify common failure modes.
11. Review the implementation as a senior engineer.
12. Only then move to the next phase.

Do not dump the entire project in one response.

Start with PHASE 1 only.

PHASE 1 should include:

- Repository setup
- Spring Boot setup
- Maven configuration
- PostgreSQL
- Docker Compose
- Database migration system
- Basic project structure
- User entity
- Product entity
- Order entity
- OrderItem entity
- Basic repository layer
- Basic service layer
- Basic REST API
- Global exception handling
- Validation
- OpenAPI/Swagger
- Basic unit tests
- Basic integration test

Do NOT implement Kafka, Redis, AWS, microservices, payment workers,
outbox, DLQ or advanced distributed-system features in Phase 1.

We will add those in later phases.

After completing Phase 1, STOP and wait for my instruction.

IMPORTANT:
If you believe one of my requirements is technically incorrect,
challenge it instead of blindly implementing it.

Act like a senior engineer reviewing my design, not like a code
autocomplete tool.
```

------------------------------------------------------------------------

# 48. Prompts for Later Phases

## Architecture Review Prompt

``` text
Review the current OrderFlow architecture as a Principal Backend
Engineer.

Do not rewrite the project.

Identify:

1. Scalability bottlenecks
2. Race conditions
3. Transaction boundary problems
4. Data consistency problems
5. Security issues
6. Failure scenarios
7. Retry problems
8. Duplicate-message problems
9. Database bottlenecks
10. Redis failure scenarios
11. Messaging failure scenarios
12. Observability gaps
13. AWS deployment risks

For every issue provide:

- Severity
- Why it is a problem
- Concrete example
- Recommended fix
- Trade-off

Do not recommend adding technology unless it solves a demonstrated
problem.
```

## Concurrency Testing Prompt

``` text
Act as a senior distributed-systems engineer.

Review the OrderFlow inventory reservation implementation.

Focus exclusively on concurrency.

Analyze:

- race conditions
- lost updates
- optimistic locking
- pessimistic locking
- transaction isolation
- deadlocks
- overselling
- duplicate requests

Create automated tests that simulate concurrent customers attempting
to purchase limited inventory.

Explain exactly why the implementation is safe or unsafe.

Do not assume that sequential unit tests prove concurrency correctness.
```

## Failure Testing Prompt

``` text
Act as a Site Reliability Engineer.

Perform a failure analysis of OrderFlow.

Consider:

- PostgreSQL failure
- Redis failure
- Kafka/SQS failure
- Payment service failure
- Inventory service failure
- Worker crash
- Network timeout
- Duplicate message
- Duplicate HTTP request
- Partial deployment
- Database connection pool exhaustion

For each scenario explain:

Detection
Impact
Recovery
Retry
Consistency
User experience
Observability

Then identify the highest-priority reliability improvements.
```

## AWS Review Prompt

``` text
Act as an AWS Solutions Architect and backend production engineer.

Review the OrderFlow AWS deployment.

My goals are:

1. Keep the architecture inexpensive.
2. Stay within applicable AWS Free Tier allowances where possible.
3. Avoid surprise charges.
4. Maintain reasonable production-like architecture.
5. Use least-privilege IAM.
6. Keep databases private where practical.
7. Add monitoring and billing alerts.

Review:

API Gateway
Compute
RDS/PostgreSQL
SQS/SNS
ECR
CloudWatch
IAM
VPC
Security Groups
Secrets
Networking

Identify unnecessary resources and cost risks.

Where AWS pricing/free-tier information may have changed, verify the
current AWS documentation before making a claim.

Provide an implementation plan rather than blindly creating resources.
```

## Interview Simulation Prompt

``` text
Act as a senior backend interviewer.

Interview me about my OrderFlow project.

Ask one question at a time.

Focus on:

- why PostgreSQL
- transactions
- isolation levels
- optimistic locking
- idempotency
- outbox pattern
- Kafka/SQS
- at-least-once delivery
- duplicate events
- retries
- DLQ
- Redis
- rate limiting
- circuit breakers
- scalability
- AWS
- observability
- security
- CI/CD

Do not accept memorized definitions.

Ask follow-up questions that force me to explain what happens during
failures and concurrency problems.

After each answer:

1. Score me from 1-10.
2. Identify what was correct.
3. Identify what was missing.
4. Give the ideal senior-level answer.
5. Ask the next question.
```

------------------------------------------------------------------------

# 49. Final Project Principle

The most important rule for OrderFlow is:

> **Do not optimize for the number of technologies. Optimize for the
> number of engineering problems you can explain and solve.**

A strong final project should allow you to confidently explain:

``` text
"I designed it this way because..."

"It fails in this situation because..."

"We prevent that failure using..."

"The trade-off is..."

"Under concurrency..."

"At higher scale..."

"During deployment..."

"During a database failure..."

"During duplicate delivery..."

"From an observability perspective..."
```

That is what turns OrderFlow from a portfolio project into a serious
backend engineering project.
