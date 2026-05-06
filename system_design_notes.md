# System Design

Revision areas:

1. **Delivery Framework** — interview structure, timing, and delivery.
2. **Core Concepts** — technology-agnostic fundamentals.
3. **Key Technologies** — common system design building blocks.
4. **Common Patterns** — reusable architecture patterns.

# Part 1: Delivery Framework


## 1. Why Delivery Matters

Reliable delivery sequence:

```text
Requirements
    ↓
Core Entities
    ↓
API (Application Programming Interface) / System Interface
    ↓
Optional Data Flow
    ↓
High-Level Design
    ↓
Deep Dives
```

---

## 2. Recommended Interview Timing

Typical 45-minute structure:

| Section | Approx Time |
|---|---:|
| Requirements | ~5 minutes |
| Core Entities | ~2 minutes |
| API (Application Programming Interface) / System Interface | ~5 minutes |
| Data Flow (optional) | ~5 minutes |
| High-Level Design | ~10–15 minutes |
| Deep Dives | ~10 minutes |

Timing can shift, but keep early sections short so enough time remains for architecture and deep dives.

---

## 3. Step 1 — Requirements (~5 Minutes)

Use requirements to narrow a broad prompt into a focused problem.

Break requirements into two groups:

```text
Functional Requirements
Non-Functional Requirements
```

Discuss capacity only when numbers affect the design.

---

## 4. Functional Requirements

Functional requirements describe user or client actions.

Useful phrasing:

```text
Users should be able to...
Clients should be able to...
The system should allow...
```

These define the core product behavior.

Example for a Twitter-like system:

```text
Users should be able to post tweets.
Users should be able to follow other users.
Users should be able to view a feed of tweets from people they follow.
```

Example for a cache:

```text
Clients should be able to insert items.
Clients should be able to read items.
Clients should be able to set expiration times.
```

Example for a ticket booking system:

```text
Users should be able to search events.
Users should be able to view available seats.
Users should be able to reserve and purchase seats.
```

### How Many Functional Requirements?

Keep the list focused: prioritize the top 3 core requirements.

Every requirement needs architectural support; too many requirements dilute the answer.

Good interview behavior:

```text
I will focus on the three most important flows first: posting content, following users, and generating the feed. I will treat likes, comments, and notifications as out of scope unless we have time later.
```

### Functional Requirement Pitfalls

Avoid:

- Listing every possible feature.
- Accepting vague requirements without narrowing them.
- Spending too long debating product details.
- Designing for features that were never prioritized.

---

## 5. Non-Functional Requirements

Non-functional requirements describe system qualities: **how well** it works.

Useful phrasing:

```text
The system should be...
The system should support...
The system should prioritize...
```

Example for a Twitter-like system:

```text
The system should be highly available.
The system should support 100M+ daily active users.
The system should render feeds with low latency, for example under 200 ms for cached feeds.
```

Example for a ticket booking system:

```text
The system should prevent double booking of seats.
The system should handle bursty traffic during popular ticket launches.
The system should prioritize consistency for seat reservation and payment flows.
```

### Make Non-Functional Requirements Specific

A weak requirement is:

```text
The system should be low latency.
```

A stronger requirement is:

```text
The search API should return results in under 500 ms for common queries.
```

A weak requirement is:

```text
The system should scale.
```

A stronger requirement is:

```text
The system should support bursty read traffic during event launches, where thousands of users may refresh the same event page at the same time.
```

### Useful Non-Functional Requirement Checklist

Checklist:

(SCaLE For Cloud Design SystemS)

| Area | Questions to Ask |
|---|---|
| CAP (Consistency, Availability, Partition Tolerance) / Consistency | Should the system prioritize consistency or availability during failures? |
| Scalability | Does the system need to scale reads, writes, or both? Is traffic bursty? |
| Latency | Which operations need low latency? Search? Feed loading? Booking? Chat? |
| Durability | How bad is data loss? Can the system lose events, messages, payments, or files? |
| Fault Tolerance | What happens if a server, database, queue, or region fails? |
| Security | Does the system need authentication, authorization, encryption, or abuse prevention? |
| Compliance | Are there legal or regulatory constraints, such as payments, healthcare, or privacy? |
| Environment | Are clients mobile devices, browsers, low-bandwidth devices, or internal services? |

### How Many Non-Functional Requirements?

Pick the top 3–5 that matter most for the problem.

For example:

```text
For a chat system:
1. Low message delivery latency.
2. High availability.
3. Eventual consistency is acceptable for read receipts, but message persistence should be durable.

For a banking system:
1. Strong consistency for balances and transactions.
2. Durability and auditability.
3. Security and compliance.
```

---

## 6. Capacity Estimation

Do not start every interview with DAU (Daily Active Users), QPS (Queries Per Second), storage, bandwidth, and server counts.

Rule:

```text
Do capacity math only when it changes a design decision.
```

Capacity estimation helps decide:

- Do we need sharding?
- Do we need read replicas?
- Can a single Redis instance handle this?
- How many workers do we need?
- How many Kafka partitions do we need?
- Can one region handle the traffic?
- Can this data structure fit in memory?

### When to Skip Upfront Estimation

Say:

```text
I would like to skip detailed capacity estimates upfront and do the math when it directly affects the design, such as deciding whether we need sharding, caching, or multiple worker pools.
```

Better than calculating many numbers and only concluding:

```text
The system is large.
```

That does not guide the architecture.

### Example: Useful Capacity Estimate

Suppose you are designing a trending topics system.

The number of unique topics matters because it affects whether you can keep counts in one in-memory structure or need to partition the work.

```text
If there are only thousands of active topics:
    A single in-memory min-heap may work.

If there are millions of active topics:
    Partition by topic hash and aggregate partial TopK results.
```

Here, the estimate directly affects the architecture.

---

## 7. Step 2 — Core Entities (~2 Minutes)

After requirements, identify core entities: the main nouns exchanged by APIs and persisted by databases.

Example for Twitter:

```text
User
Tweet
Follow
Feed
```

Example for Ticketmaster:

```text
User
Event
Venue
Seat
Reservation
Booking
Payment
Ticket
```

Example for Uber:

```text
Rider
Driver
Location
Ride
Match
Payment
```

Example for Dropbox:

```text
User
File
Folder
Permission
FileVersion
UploadSession
```

### How to Identify Core Entities

Ask yourself:

```text
Who are the actors in the system?
What are the main resources users interact with?
What data must be persisted?
What objects appear in the functional requirements?
```

If the requirement says “users should be able to book seats for events,” the likely entities are:

```text
User
Event
Seat
Booking
Payment
```

### Why Not Design the Full Schema Immediately?

Do not list every column yet; access patterns and state transitions are still evolving.

Start with the entity names first. Later, during the high-level design, add important fields next to the database where they matter.

For example, instead of writing a complete `User` schema, focus on fields that affect the design:

```text
Seat
- seat_id
- event_id
- status
- reservation_expires_at
```

These fields protect against double booking.

### Core Entity Pitfalls

Avoid:

- Writing a full schema too early.
- Including every minor object.
- Using vague names like `Data`, `Record`, or `Object`.
- Missing important stateful entities such as Reservation, Payment, Job, or Session.

---

## 8. Step 3 — API (Application Programming Interface) or System Interface (~5 Minutes)

Before architecture, define the client-system contract.

Default to REST (Representational State Transfer) for product-style interviews.

```text
Client → REST API (Application Programming Interface) → Backend System
```

Use GraphQL for flexible client data fetching. Use gRPC (Google Remote Procedure Call) for high-performance internal service-to-service calls.

---

## 9. Choosing an API Style

| API Style | When to Use |
|---|---|
| REST (Representational State Transfer) | Default for most interviews and product APIs |
| GraphQL | Clients need different shapes of data and want to avoid over-fetching |
| RPC (Remote Procedure Call) / gRPC (Google Remote Procedure Call) | Internal service-to-service communication where performance and strong contracts matter |
| WebSockets | Bidirectional realtime communication |
| SSE (Server-Sent Events) | Server-to-client realtime updates |

Interview line:

```text
I will use REST for the core API because it is simple and widely supported. If we need realtime updates, I will add SSE or WebSockets separately.
```

---

## 10. REST API Design Guidelines

Use plural resources:

```text
/tweets
/users
/events
/bookings
/messages
```

Use HTTP methods by intent:

| Method | Meaning |
|---|---|
| GET | Read data |
| POST | Create data or start an action |
| PUT/PATCH | Update data |
| DELETE | Delete data |

Example Twitter APIs:

```http
POST /v1/tweets
Body:
{
  "text": "hello world"
}

GET /v1/tweets/{tweet_id}

POST /v1/follows
Body:
{
  "followee_id": "user_123"
}

GET /v1/feed?cursor=abc&limit=20
```

Example ticket booking APIs:

```http
GET /v1/events/search?city=sf&date=2026-05-06
GET /v1/events/{event_id}/seats
POST /v1/events/{event_id}/reservations
POST /v1/reservations/{reservation_id}/payments
GET /v1/bookings/{booking_id}
```

### Derive Current User from Auth

Do not trust sensitive identity from request bodies.

Weak design:

```json
{
  "user_id": "user_123",
  "tweet": "hello"
}
```

Better design:

```text
Current user is derived from the authentication token.
Request body contains only the input needed for the action.
```

Example:

```http
POST /v1/tweets
Authorization: Bearer <token>
Body:
{
  "text": "hello"
}
```

The backend derives `user_id` from the token.

### API Design Pitfalls

Avoid:

- Spending too long on API details.
- Designing 20 endpoints.
- Putting sensitive identity in the request body.
- Ignoring pagination for list APIs.
- Forgetting idempotency for payment, booking, or retryable write APIs.

---

## 11. Optional Step — Data Flow (~5 Minutes)

For backend processing systems, describe data flow before components.

Use data flow for staged processing.

Examples:

- Web crawler
- Video processing pipeline
- Analytics pipeline
- Search indexing system
- Notification delivery system
- Payment workflow
- File upload and processing system

Example web crawler data flow:

```text
Fetch seed URLs
      ↓
Parse HTML
      ↓
Extract links
      ↓
Store page content
      ↓
Schedule new URLs
      ↓
Repeat
```

Example video upload data flow:

```text
User uploads video
      ↓
Store raw video in blob storage
      ↓
Create processing job
      ↓
Transcode video
      ↓
Generate thumbnails
      ↓
Update metadata
      ↓
Publish video
```

Example payment flow:

```text
Create order
      ↓
Reserve inventory
      ↓
Authorize payment
      ↓
Capture payment
      ↓
Confirm order
      ↓
Send receipt
```

### When to Skip Data Flow

Skip data flow for simple CRUD (Create, Read, Update, Delete) or request-response systems.

---

## 12. Step 4 — High-Level Design (~10–15 Minutes)

High-level design = main architecture boxes and arrows.

Goal: a working system that satisfies APIs and requirements.

A common starting point:

```text
Client
  ↓
Load Balancer / API Gateway
  ↓
Application Service
  ↓
Database
```

Add components only when requirements justify them:

```text
Cache → for hot reads and low latency
Queue → for async or long-running work
Blob Storage → for large files
Search Index → for full-text search
Stream → for event processing and multiple consumers
CDN (Content Delivery Network) → for global static content
Distributed Lock → for limited resource contention
```

---

## 13. Build the Design Endpoint by Endpoint

Build architecture endpoint by endpoint.

Example for Twitter:

### `POST /tweets`

```text
Client
  ↓
API Gateway
  ↓
Tweet Service
  ↓
Tweet Database
```

The service validates the request, derives the user from auth, stores the tweet, and returns success.

### `POST /follows`

```text
Client
  ↓
API Gateway
  ↓
Follow Service
  ↓
Follow Database
```

The service creates a follow relationship.

### `GET /feed`

Start simple:

```text
Client
  ↓
API Gateway
  ↓
Feed Service
  ↓
Tweet DB + Follow DB
```

Then call out the obvious bottleneck:

```text
This simple feed read path may be slow at scale. I will first complete the baseline design, then deep dive into fanout-on-read vs fanout-on-write and caching.
```

This signals bottleneck awareness without derailing the baseline design.

---

## 14. What to Include in the High-Level Design

Include components required by core requirements.

Common components:

```text
Client
API Gateway
Load Balancer
Application Services
Primary Database
Read Replicas
Cache
Queue
Worker Pool
Blob Storage
CDN
Search Index
Stream / Event Bus
Notification Service
Monitoring / Logging if relevant
```

For each important request, explain:

```text
1. Where the request enters.
2. Which service handles it.
3. What data is read or written.
4. What state changes.
5. What response is returned.
6. What happens asynchronously, if anything.
```

### Add Important Fields Near the Database

Skip full schemas; add only design-relevant fields.

Example for seat booking:

```text
Seat
- seat_id
- event_id
- status

Reservation
- reservation_id
- seat_id
- user_id
- expires_at
- status

Booking
- booking_id
- reservation_id
- payment_status
```

These fields explain seat state and reservation expiration.

### High-Level Design Pitfalls

Avoid:

- Adding cache, queue, sharding, and Kafka before a basic working system exists.
- Drawing components without explaining data flow.
- Spending too much time on icons or perfect diagrams.
- Forgetting to persist important state.
- Designing only the write path and forgetting the read path.
- Designing only happy path and ignoring state changes.

---

## 15. Step 5 — Deep Dives (~10 Minutes)

Deep dives harden the baseline design against non-functional requirements.

Deep dives cover:

```text
Scale
Latency
Consistency
Availability
Failure handling
Bottlenecks
Hot paths
Edge cases
Data correctness
Operational concerns
```

Senior answers proactively identify the highest-impact bottlenecks.

---

## 16. How to Choose Deep Dives

Use the non-functional requirements to decide.

Example: Twitter-like feed

```text
NFR: Feed should load quickly for 100M+ users.
Deep dive: fanout-on-read vs fanout-on-write, feed cache, celebrity users, ranking, cache invalidation.
```

Example: Ticket booking

```text
NFR: Prevent double booking during high-demand sales.
Deep dive: transactions, row locks, reservation TTL, distributed locks, queue-based serialization, payment failure handling.
```

Example: YouTube upload

```text
NFR: Support large video uploads and background processing.
Deep dive: presigned URLs, multipart uploads, transcoding workers, job status, retries, CDN delivery.
```

Example: Uber

```text
NFR (Non-Functional Requirement): Match riders with nearby drivers at low latency.
Deep dive: geospatial indexing, driver location updates, matching consistency, realtime ride updates.
```

---

## 17. Common Deep Dive Topics

### Scaling Reads

Discuss:

- Indexes
- Read replicas
- Cache-aside with Redis
- CDN for static/public content
- Hot key handling
- Cache invalidation

### Scaling Writes

Discuss:

- Partitioning
- Sharding
- Queue buffering
- Batching
- Load shedding
- Backpressure
- Idempotency

### Contention and Consistency

Discuss:

- Database transactions
- Conditional updates
- Optimistic concurrency control
- Pessimistic locking
- Distributed locks
- Reservation TTLs (Time To Live)
- Sagas or compensation

### Long-Running Tasks

Discuss:

- Job queue
- Worker pool
- Job status table
- Retries
- Dead letter queue
- Idempotency
- Progress updates

### Realtime Updates

Discuss:

- Polling vs SSE vs WebSockets
- Stateful connection management
- Pub/Sub fanout
- Reconnection
- Message ordering
- Load balancing for persistent connections

### Availability and Fault Tolerance

Discuss:

- Replication
- Failover
- Multi-AZ deployment
- Graceful degradation
- Retry with backoff
- Circuit breakers
- Monitoring and alerting

---

## 18. Communication During Deep Dives

Do not talk over the interviewer. Deep dives should be collaborative.

A good approach:

```text
The biggest bottleneck I see is feed generation latency. I can deep dive into fanout-on-read vs fanout-on-write, unless you would prefer I focus on database scaling or cache invalidation first.
```

This shows ownership while giving the interviewer room to guide the discussion.

If the interviewer asks about a specific bottleneck, follow their lead.

---

## 19. Full Delivery Framework Cheat Sheet

| Step | What to Do | What to Avoid |
|---|---|---|
| Requirements | Pick 3 core functional requirements and 3–5 non-functional requirements | Long feature lists and vague goals |
| Core Entities | List the main nouns/resources | Full schema too early |
| API / Interface | Define 4–6 key endpoints or system operations | Spending 10+ minutes on API details |
| Optional Data Flow | Use for processing pipelines or workflows | Forcing it into simple CRUD systems |
| High-Level Design | Draw a complete baseline architecture | Adding complexity before the system works |
| Deep Dives | Improve for scale, latency, failures, consistency | Talking nonstop without interviewer input |

---

## 20. Interview Script You Can Reuse

### Opening

```text
I’ll start by clarifying the functional and non-functional requirements, then identify the core entities, define the main APIs, draw a high-level design, and finally deep dive into bottlenecks and trade-offs.
```

### Requirements

```text
For functional requirements, I’ll focus on the top three user flows so we can keep the design scoped. For non-functional requirements, I’ll call out the most important system qualities such as latency, availability, consistency, and scale.
```

### Capacity Estimation

```text
I’ll skip detailed capacity math upfront unless it affects a design decision. If we need to decide whether to shard, cache, or add workers, I’ll do the estimate at that point.
```

### Core Entities

```text
The main entities I see are User, [Entity 1], [Entity 2], and [Entity 3]. I’ll keep this as a first draft and refine fields as we design the read and write paths.
```

### API

```text
I’ll use REST for the external API because it is simple and fits the request-response flows. The current user will come from the auth token rather than the request body.
```

### High-Level Design

```text
I’ll first build a simple design that satisfies the main APIs, then call out bottlenecks and improve them in the deep dive.
```

### Deep Dive

```text
The main bottleneck I see is [bottleneck]. I’ll address it by comparing [option A] and [option B], then choose the approach that best fits our requirements.
```

---

## 21. Final Delivery Summary

Delivery framework: produce a complete, understandable design under time pressure.

```text
Clarify what to build.
Identify the core data.
Define the system contract.
Draw a working baseline architecture.
Then improve the design where the requirements demand it.
```

Strong candidates manage the interview, make trade-offs, and evolve a simple design into a robust one.

---

# Part 2: Core Concepts

Based on the Hello Interview **System Design in a Hurry: Core Concepts** material.

## 1. Why Core Concepts Matter

Core concepts explain **why** and **when** tools such as Redis, Kafka, Postgres, S3 (Simple Storage Service), and Elasticsearch are useful.

Core concept vocabulary:

- How services communicate over the network
- How APIs expose system behavior
- How data should be modeled
- How indexes speed up queries
- How caching reduces latency
- How sharding distributes data
- How consistency and availability trade off
- How rough numbers guide scaling decisions

Interviewers check principles and trade-offs, not only tool names.

```text
Core Concepts
      ↓
Key Technologies
      ↓
Common Patterns
      ↓
Complete System Design
```

### How to Use Core Concepts in Interviews

Start simple; add concepts when requirements demand them.

```text
Understand the product requirements
        ↓
Identify the main data and API flows
        ↓
Pick simple storage and communication choices
        ↓
Use core concepts to handle scale, latency, reliability, and consistency
        ↓
Explain trade-offs clearly
```

Use only concepts relevant to the bottleneck or failure mode.

---

## 2. Networking Essentials

### What It Means

Networking covers communication between clients, services, databases, queues, caches, and other components. Focus on choices that affect latency, reliability, and scalability.

Most systems use **HTTP (Hypertext Transfer Protocol) over TCP (Transmission Control Protocol)** as the default communication model. It is reliable, widely supported, easy to debug, and works well for normal request-response APIs.

```text
Client
  ↓ HTTP request
API Server
  ↓ HTTP/gRPC/internal call
Backend Service
  ↓ query
Database / Cache / Queue
```

### Default Protocol Choice

For most product design interviews, start with:

```text
External client APIs → REST over HTTP
Internal service calls → HTTP or gRPC
Realtime communication → SSE (Server-Sent Events) or WebSockets if needed
```

Do not start with WebSockets, gRPC, or custom protocols unless the requirement justifies them.

### HTTP

HTTP is the default for most client-server communication.

Use HTTP when:

- The client sends a request and expects a response
- The operation is not continuously streaming
- The system needs broad browser/mobile/client compatibility
- You want simple routing, debugging, observability, and caching

Example:

```text
GET /users/123
POST /orders
GET /events/456/seats
```

HTTP is enough for normal CRUD (Create, Read, Update, Delete) APIs, search APIs, booking APIs, and user-facing flows.

### SSE (Server-Sent Events)

SSE is used when the server needs to push updates to the client over a long-lived connection.

```text
Client opens connection
        ↓
Server keeps connection open
        ↓
Server pushes updates to client
```

SSE is **unidirectional**:

```text
Server ───────► Client
```

Use SSE when:

- The server pushes notifications
- The client mostly listens
- Updates flow in one direction
- You want something simpler than WebSockets

Examples:

- Live score updates
- Notification feed
- Job progress updates
- Dashboard refresh events

### WebSockets

WebSockets provide bidirectional communication over a persistent connection.

```text
Client ⇄ Server
```

Use WebSockets when both sides need to send messages frequently.

Examples:

- Chat applications
- Multiplayer games
- Collaborative document editing
- Live cursors
- Realtime auctions

WebSockets add complexity because connections are stateful. Servers must track active connections, handle reconnects, and route messages to the correct connected users.

### SSE vs WebSockets

| Requirement | Better Choice |
|---|---|
| Server pushes updates only | SSE |
| Client and server both send frequent messages | WebSockets |
| Simple notification stream | SSE |
| Chat or collaboration | WebSockets |
| Easier integration with HTTP infrastructure | SSE |
| True bidirectional realtime communication | WebSockets |

### gRPC

gRPC fits high-performance internal service-to-service communication. It uses HTTP/2 and binary serialization, often faster and more compact than JSON (JavaScript Object Notation) over HTTP.

Use gRPC when:

- Services communicate internally
- Low latency matters
- Strong contracts are useful
- You control both client and server
- The system has high request volume between services

Common pattern:

```text
External API: REST / HTTP
Internal APIs: gRPC
```

Avoid gRPC as the default public browser API because browsers do not natively support full gRPC without workarounds such as gRPC-Web and proxies.

### Load Balancing Basics

When multiple servers can handle the same request, use a load balancer to distribute traffic.

```text
Client Requests
      ↓
Load Balancer
  ↓       ↓       ↓
Server  Server  Server
```

### L4 (Layer 4 / Transport Layer) vs L7 (Layer 7 / Application Layer) Load Balancing

| Type | Layer | What It Sees | Good For |
|---|---|---|---|
| L4 (Layer 4 / Transport Layer) | TCP/UDP (User Datagram Protocol) | Connection information | WebSockets, high throughput, persistent connections |
| L7 (Layer 7 / Application Layer) | HTTP/application | URL (Uniform Resource Locator), headers, cookies, request content | REST APIs, path routing, service routing |

Rule of thumb:

```text
Normal HTTP APIs → L7 load balancer
Persistent WebSocket connections → often L4 load balancer
```

### Geography and Latency

Network distance matters. Requests across continents are limited by the speed of light and network routing. Even before application processing, cross-region calls can add tens or hundreds of milliseconds.

```text
Same machine: extremely fast
Same data center: low milliseconds
Same region: few milliseconds to tens of milliseconds
Cross-continent: tens to hundreds of milliseconds
```

If global latency matters, consider:

- CDN for static assets
- Regional deployments
- Geo-partitioned data
- Replication across regions
- Routing users to the nearest region

### Common Networking Pitfalls

- Do not propose WebSockets just because the word “realtime” appears.
- Do not ignore reconnects and failure handling for stateful connections.
- Do not route all global traffic to one region if low latency is required.
- Do not overdesign protocols before the basic API and data model are clear.

### Interview Line to Use

> I would default to HTTP for external APIs because it is simple and widely supported. If the system requires server-pushed updates, I would consider SSE first. If it requires true bidirectional realtime communication, such as chat or collaboration, I would use WebSockets and account for stateful connection scaling.

---

## 3. API Design

### What It Means

API design defines how clients interact with your system. In most system design interviews, you only need to sketch the most important APIs, not every endpoint.

Show product flow and main system operations.

### Default Choice: REST

For most interviews, REST over HTTP is a safe default.

REST maps resources to URLs and uses HTTP methods to act on them.

Examples:

```text
GET    /users/{user_id}
POST   /users
GET    /events/{event_id}
POST   /events/{event_id}/bookings
GET    /orders/{order_id}
DELETE /sessions/{session_id}
```

### How Much API Detail Is Enough?

In interviews, 4–6 core endpoints are enough.

Focus on:

- Main write operation
- Main read operation
- Status or lookup operation
- Search/list operation if relevant
- Realtime or async status endpoint if relevant

Example for a ticket booking system:

```text
GET  /events/search?city=sf&date=2026-05-06
GET  /events/{event_id}/seats
POST /events/{event_id}/bookings
GET  /bookings/{booking_id}
POST /payments
```

### Pagination

If an endpoint returns many results, use pagination.

Offset pagination:

```text
GET /posts?limit=20&offset=40
```

Cursor pagination:

```text
GET /posts?limit=20&cursor=abc123
```

Cursor-based pagination fits feeds and realtime lists because new items can arrive during paging.

| Type | Good For | Weakness |
|---|---|---|
| Offset pagination | Simple admin pages, stable datasets | Can skip or duplicate items when data changes |
| Cursor pagination | Feeds, timelines, high-scale lists | Slightly more complex |

### Authentication

Common choices:

```text
User authentication → JWT (JSON Web Token) or session token
Service-to-service authentication → API key, mTLS (Mutual Transport Layer Security), service identity
```

For interviews, mention authentication and authorization at the API gateway or service layer.

### Rate Limiting

Use rate limiting when the system may face abuse, bots, scraping, or accidental overload.

Examples:

```text
100 requests/minute per user
1000 requests/minute per API key
10 login attempts/hour per IP
```

Rate limiting can be implemented using Redis counters, token bucket, or leaky bucket algorithms.

### API Design Pitfalls

- Do not spend 10 minutes designing endpoints.
- Do not include every minor API.
- Do not ignore pagination for large lists.
- Do not forget idempotency for retryable writes such as payments or booking requests.

### Interview Line to Use

> I would expose a small set of REST APIs for the main user flows, keep the endpoint design simple, and move quickly to the harder parts of the system such as data modeling, consistency, scaling, and failure handling.

---

## 4. Data Modeling

### What It Means

Data modeling is deciding what data to store, how entities relate to each other, and how the structure supports your main access patterns.

Data modeling affects:

- Query performance
- Storage cost
- Consistency
- Scalability
- Feature flexibility
- Operational complexity

### Start with the Main Entities

A good first step is identifying core entities.

Example for an e-commerce system:

```text
User
Product
Inventory
Cart
Order
Payment
Shipment
```

Example for a chat system:

```text
User
Conversation
Message
Membership
Attachment
ReadReceipt
```

### Relational vs NoSQL (Non-Relational / Not Only SQL) Thinking

Relational databases are often a safe default when data is structured and consistency matters.

Use relational modeling when:

- Entities have clear relationships
- You need transactions
- You need flexible queries
- You need constraints and integrity

NoSQL modeling is useful when:

- Access patterns are simple and known up front
- You need very high horizontal scale
- You need flexible schemas
- You want to optimize for key-based lookups

### Normalization

Normalization means splitting data into separate tables to avoid duplication.

Example:

```text
users table
- user_id
- name
- email

orders table
- order_id
- user_id
- total_amount
```

Benefits:

- Reduces duplicate data
- Keeps updates consistent
- Prevents anomalies
- Works well with transactions and joins

Downside:

- Reads may require joins
- Joins can become expensive at scale

### Denormalization

Denormalization duplicates data to make reads faster.

Example:

```text
orders table
- order_id
- user_id
- user_name
- user_email
- total_amount
```

Now an order page can display user details without joining the users table.

Benefits:

- Faster reads
- Fewer joins
- Better for read-heavy paths

Downsides:

- More complex updates
- Risk of stale duplicated data
- More storage usage

### Safe Interview Default

Start with a normalized relational model, then denormalize specific hot paths if read performance becomes a problem.

```text
Start normalized
      ↓
Identify expensive read path
      ↓
Denormalize or precompute only that path
```

### Access Pattern Design for NoSQL

In NoSQL systems, design tables around queries.

Example:

```text
Access pattern: get all posts for a user
Partition key: user_id
Sort key: created_at
```

This query becomes fast because all posts for a user are stored together or can be retrieved efficiently by key.

But a different query may become hard:

```text
Access pattern: get all posts for hashtag X
```

If the table was not designed for hashtag lookup, the system may need a secondary index, separate table, or search system.

### Data Modeling Pitfalls

- Do not pick NoSQL without knowing the access patterns.
- Do not denormalize everything upfront.
- Do not ignore update complexity when duplicating data.
- Do not design only tables without explaining the main queries.

### Interview Line to Use

> I would start by modeling the main entities and access patterns. For structured transactional data, I would start with a normalized relational model. If specific reads become hot, I would denormalize or precompute those paths while keeping the primary database as the source of truth.

---

## 5. Database Indexing

### What It Means

Indexes make database queries faster by allowing the database to find data without scanning every row.

Without an index:

```text
Scan every row in users table to find email = x
```

With an index:

```text
Use index to jump directly to matching row
```

### When to Use Indexes

Add indexes on fields used frequently in:

- WHERE clauses
- JOIN conditions
- ORDER BY clauses
- Range filters
- Unique lookups

Examples:

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_events_city_date ON events(city, event_date);
```

### B-Tree Indexes

B-tree indexes are the most common database indexes. They support:

- Exact lookups
- Range queries
- Sorting
- Prefix scans in some cases

Examples:

```text
Find user by email
Find orders between two dates
Find events in a city sorted by date
```

### Hash Indexes

Hash indexes are useful for exact lookups but not range queries.

```text
Good: email = 'a@test.com'
Bad: created_at between date1 and date2
```

### Compound Indexes

A compound index includes multiple columns.

Example:

```sql
CREATE INDEX idx_events_city_date ON events(city, event_date);
```

This helps queries like:

```sql
SELECT * FROM events
WHERE city = 'San Francisco'
AND event_date = '2026-05-06';
```

The order of columns matters. Put the fields that match your query patterns first.

### Specialized Indexes

Some systems need specialized indexes:

| Need | Index / Technology |
|---|---|
| Full-text search | Elasticsearch, OpenSearch, Postgres GIN index |
| Geo queries | PostGIS, Redis Geo, Elasticsearch geo index |
| Prefix/autocomplete search | Search index or trie-like structures |
| Vector similarity | Vector index / ANN index |

### Index Trade-Offs

Indexes are not free.

Benefits:

- Faster reads
- Faster filtering
- Faster joins
- Faster ordering

Costs:

- Slower writes
- More storage
- Index maintenance overhead
- Too many indexes can hurt performance

### External Indexes

Sometimes your primary database is not enough. You may maintain an external index such as Elasticsearch.

```text
Application writes data
        ↓
Primary Database
        ↓ CDC / queue / event
Search Index
```

This makes search fast, but the index may lag behind the primary database. Slight staleness is acceptable for many search use cases.

### Interview Pitfalls

- Do not say “add an index” without naming the field and query.
- Do not index every column.
- Do not forget write overhead.
- Do not use Elasticsearch as the source of truth for transactional data.

### Interview Line to Use

> I would add indexes based on the actual query patterns. For example, if we frequently fetch orders by user, I would index orders.user_id. If we need full-text search or geospatial search, I would use a specialized external index and accept slight async replication lag.

---

## 6. Caching

### What It Means

Caching stores frequently accessed data in faster storage, often memory, to avoid repeated database queries or computations.

```text
Client
  ↓
API Server
  ↓
Cache
  ↓ if miss
Database
```

### Why Caching Matters

Caches reduce:

- Latency
- Database load
- Repeated computation
- Cost of serving hot content

Redis cache lookups are faster than database queries, so caching helps read-heavy systems.

### When to Use Caching

Use caching when:

- Data is read frequently
- Data does not change constantly
- Query computation is expensive
- Many users request the same data
- Database reads are becoming a bottleneck

Examples:

- Popular product pages
- User sessions
- Event metadata
- Feed results
- Search suggestions
- Aggregated dashboard metrics

### Cache-Aside Pattern

Cache-aside is the common default.

```text
Read request
  ↓
Check cache
  ↓
Cache hit → return cached data
Cache miss → query DB → store in cache → return data
```

### Cache Invalidation

Invalidation is the hardest part of caching. When the source-of-truth data changes, the cached copy may become stale.

Common strategies:

- Delete cache entry after write
- Update cache entry after write
- Use TTL expiration
- Use event-driven invalidation
- Use versioned cache keys

Example:

```text
User updates profile
      ↓
Database update succeeds
      ↓
Delete user:{id} from cache
      ↓
Next read reloads fresh value
```

### TTL (Time To Live)

TTL means time-to-live. The cache entry expires automatically after a fixed period.

```text
Cache user profile for 5 minutes
Cache product catalog page for 1 hour
Cache static content for 1 day
```

Short TTL means fresher data but lower hit rate. Long TTL means better performance but more staleness.

### Cache Stampede

A cache stampede happens when many requests miss the cache at the same time and all hit the database.

```text
Hot key expires
      ↓
10,000 requests miss cache
      ↓
10,000 database queries
      ↓
Database overload
```

Mitigations:

- Request coalescing
- Lock per hot key
- Staggered TTLs with jitter
- Background refresh
- Serve stale data temporarily
- Circuit breakers

### Cache Failure

If Redis goes down, traffic may fall back to the database. You should think about whether the database can handle that load.

Options:

- Graceful degradation
- Local in-process cache for small critical data
- Circuit breaker
- Rate limiting
- Read replicas

### What to Cache

Do not cache everything. Cache hot, read-heavy, relatively stable data.

Good cache candidates:

```text
Popular event metadata
Product details
User sessions
Feature flags
Precomputed feed pages
Aggregated dashboard metrics
```

Bad cache candidates:

```text
Data that changes every request
Critical payment state without source-of-truth verification
Rarely accessed data
Highly user-specific sensitive data without proper controls
```

### CDN vs Distributed Cache vs In-Process Cache

| Cache Type | Best For |
|---|---|
| CDN | Static/global content close to users |
| Distributed cache | Shared application data like sessions or hot objects |
| In-process cache | Small local config or feature flags |

### Interview Pitfalls

- Do not say “add Redis” without explaining what data goes into Redis.
- Do not ignore invalidation.
- Do not cache critical state without verifying against source of truth.
- Do not forget the cache failure scenario.

### Interview Line to Use

> I would use cache-aside with Redis for hot read-heavy data. On a cache miss, the service reads from the database and stores the value with a TTL. On writes, I would invalidate or update the cache entry. For critical flows, I would still verify against the database as the source of truth.

---

## 7. Sharding

### What It Means

Sharding splits data across multiple independent database servers. Each shard stores only part of the dataset.

```text
Database Shard 1 → users 1, 4, 7
Database Shard 2 → users 2, 5, 8
Database Shard 3 → users 3, 6, 9
```

Use sharding when a single database cannot handle the storage size, write throughput, or read throughput even after simpler optimizations.

### When to Consider Sharding

Consider sharding when:

- Data size is too large for one database
- Write throughput exceeds one database
- Read replicas are not enough
- A single database becomes a hard scaling limit
- Data is naturally partitionable

Do not shard too early. A well-tuned single database with indexes, caching, and read replicas can handle a lot.

### Shard Key

The shard key determines where each record lives.

Example:

```text
shard_id = hash(user_id) % number_of_shards
```

A good shard key:

- Distributes data evenly
- Distributes traffic evenly
- Keeps related data together
- Supports common queries
- Avoids hot shards

### Hash-Based Sharding

Hash-based sharding distributes data evenly by hashing the shard key.

```text
hash(user_id) % N
```

Good for:

- Even distribution
- Avoiding obvious hot ranges
- User-scoped data

Downside:

- Range queries become harder
- Adding/removing shards can require data movement unless using consistent hashing

### Range-Based Sharding

Range-based sharding assigns ranges to shards.

Example:

```text
Shard 1: user_id 1 - 1,000,000
Shard 2: user_id 1,000,001 - 2,000,000
Shard 3: user_id 2,000,001 - 3,000,000
```

Good for range queries, but can create hot spots if one range receives more traffic.

### Directory-Based Sharding

Directory-based sharding uses a lookup service to find the shard.

```text
Lookup user_id → shard_id
```

This is flexible but adds another dependency and lookup latency.

### Sharding Trade-Offs

Sharding creates new problems:

- Cross-shard queries are expensive
- Cross-shard transactions are hard
- Resharding is operationally painful
- Hot shards can still happen
- Joins across shards are difficult
- Application logic becomes more complex

### Example: Social Media App

If you shard by `user_id`:

```text
Fast: get posts for user X
Fast: get likes by user X
Hard: get global trending posts across all users
```

That may be acceptable if most queries are user-scoped.

### Interview Pitfalls

- Do not propose sharding before doing rough capacity reasoning.
- Do not forget cross-shard transaction problems.
- Do not choose timestamp as shard key for high-write systems unless you handle hot partitions.
- Do not ignore resharding.

### Interview Line to Use

> I would avoid sharding until the capacity numbers justify it. If we outgrow a single database, I would shard by a stable key such as user_id or tenant_id, depending on the access pattern. This makes user-scoped queries fast, but global queries may need aggregation across shards.

---

## 8. Consistent Hashing

### What It Means

Consistent hashing is a technique for distributing keys across servers while minimizing data movement when servers are added or removed.

With simple modulo hashing:

```text
server = hash(key) % number_of_servers
```

If the number of servers changes, many keys map to different servers, causing massive data movement.

### Consistent Hashing Ring

Consistent hashing places both servers and keys on a virtual ring.

```text
          [Server A]
        /            \
   key1                key2
      \              /
       [Server B] -- [Server C]
```

A key belongs to the first server encountered while moving clockwise around the ring.

### Why It Helps

When a new server is added, only nearby keys move.

```text
Before: A owns a range of keys
After adding D: D takes only part of A's range
```

This is much better than moving most keys in the system.

### Where It Is Used

Consistent hashing appears in:

- Distributed caches
- Redis Cluster-like systems
- Memcached clients
- Cassandra-style databases
- DynamoDB-style partitioning
- Sharded storage systems
- Some load balancing strategies

### Virtual Nodes

Virtual nodes improve balance by placing each physical server at multiple positions on the ring.

```text
Server A → A1, A2, A3
Server B → B1, B2, B3
Server C → C1, C2, C3
```

This reduces the chance that one physical server owns a disproportionately large range.

### When to Mention It

Mention consistent hashing when:

- You distribute cache keys across multiple cache nodes
- You shard data across dynamically changing nodes
- You need elastic scaling
- You want to minimize rebalancing when nodes change

### Interview Pitfalls

Do not over-explain the algorithm unless asked. Usually, it is enough to say that consistent hashing helps distribute keys while limiting data movement during scaling events.

### Interview Line to Use

> I would use consistent hashing to distribute keys across cache or storage nodes so that adding or removing a node only moves a small portion of the keys instead of remapping the entire dataset.

---

## 9. CAP (Consistency, Availability, Partition Tolerance) Theorem

### What It Means

CAP theorem describes trade-offs in distributed systems during network partitions.

The three properties are:

| Property | Meaning |
|---|---|
| Consistency | All nodes see the same latest data |
| Availability | Every request receives a response |
| Partition tolerance | The system continues despite network splits |

During a network partition, a distributed system chooses between consistency and availability.

```text
Network partition occurs
        ↓
Choose consistency → reject/stop some requests to avoid stale data
Choose availability → keep serving requests, possibly with stale data
```

### Consistency Preference

Choose stronger consistency when stale data causes serious correctness issues.

Examples:

- Bank account balances
- Payments
- Inventory counts
- Ticket booking
- Seat reservation
- Order state transitions

In these systems, returning stale data can cause fraud, overselling, or double booking.

### Availability Preference

Choose availability or eventual consistency when stale data is acceptable.

Examples:

- Social media likes
- View counts
- News feeds
- Recommendations
- Analytics dashboards
- Product reviews

A like count stale for a few seconds often has low business impact.

### Eventual Consistency

Eventual consistency means replicas may temporarily disagree, but they converge when new writes stop.

```text
Write happens on Node A
        ↓
Node B may be stale briefly
        ↓
Replication catches up
        ↓
Nodes converge
```

### Strong Consistency

Strong consistency means reads reflect the latest committed write, but this often requires coordination and higher latency.

```text
Write must be confirmed by required nodes
        ↓
Read returns latest value
```

### Different Parts Can Have Different Consistency

Different flows can use different consistency models.

Example e-commerce system:

```text
Product description → eventual consistency is okay
Product reviews → eventual consistency is okay
Inventory count → strong consistency needed
Payment state → strong consistency needed
Order creation → strong consistency needed
```

### PACELC Reminder

CAP focuses on what happens during partitions. PACELC adds that even when there is no partition, systems still trade off latency and consistency.

```text
If Partition: choose Availability or Consistency
Else: choose Latency or Consistency
```

### Interview Pitfalls

- Do not say “CAP means you can only have two forever” without explaining partitions.
- Do not choose strong consistency for everything.
- Do not choose eventual consistency for payments, inventory, or limited resources.
- Do not ignore latency impact from coordination.

### Interview Line to Use

> For most non-critical reads, I would accept eventual consistency to improve availability and latency. For flows involving money, inventory, or booking limited resources, I would prefer strong consistency and accept the extra coordination cost.

---

## 10. Numbers to Know

### Why Numbers Matter

Back-of-the-envelope estimates guide scaling decisions. Use numbers when they change architecture.

Examples:

- Do we need sharding?
- Can one Redis cluster handle this?
- How many app servers do we need?
- Is the bottleneck storage, reads, writes, CPU, or network?

### Latency Numbers

Rough latency intuition:

| Operation | Rough Latency |
|---|---|
| Memory access | Nanoseconds |
| SSD (Solid-State Drive) read | Microseconds to low milliseconds |
| Redis/cache lookup | Around 1 ms |
| Database query | Few ms to tens of ms depending on query |
| Same data center network call | 1–10 ms |
| Cross-region call | Tens to hundreds of ms |

Use these numbers to reason about caching, regional deployments, and avoiding unnecessary network hops.

### Capacity Numbers

Approximate scale intuition:

| Component | Rough Capability | Scale Trigger |
|---|---|---|
| Redis/cache | Very high ops/sec, often 100k+ operations/sec per instance depending on workload | Memory pressure, low hit rate, hot keys, latency increase |
| Relational database | Can handle significant read/write volume when indexed and tuned | Write throughput, storage size, slow queries, lock contention |
| App server | Can handle many concurrent requests depending on CPU (Central Processing Unit), memory, and workload | CPU > 70%, latency over SLA (Service-Level Agreement), memory pressure |
| Message broker | Can handle very high throughput with partitions | Consumer lag, partition limits, broker saturation |
| Blob storage | Managed systems are highly scalable | Usually not the first bottleneck; watch bandwidth, cost, and lifecycle |

### Example: App Server Estimate

```text
Expected traffic: 50,000 requests/sec
One app server handles: 5,000 requests/sec
Needed servers: 10
Add headroom: 12–15 servers
```

### Example: Storage Estimate

```text
10 million users
Each user stores 1 MB (Megabyte) of metadata
Total metadata = 10 TB (Terabyte)
```

At this point, you might start thinking about storage growth, archiving, partitioning, or sharding depending on query and write patterns.

### Example: Queue Lag Estimate

```text
Incoming jobs: 1,000 jobs/sec
Worker capacity: 800 jobs/sec
Backlog growth: 200 jobs/sec
```

This means the queue will grow forever unless you add workers, reduce work, apply backpressure, or drop non-critical jobs.

### When to Use Calculations

Use calculations when they affect a design decision:

```text
Should we shard?
Should we cache?
How many workers do we need?
How many partitions do we need?
Can one database handle this?
Can one region serve global traffic?
```

### Interview Pitfalls

- Do not use outdated assumptions that every system needs sharding early.
- Do not calculate for 10 minutes without connecting it to a decision.
- Do not ignore headroom.
- Do not forget that modern managed systems can handle large scale before custom complexity is needed.

### Interview Line to Use

> I would do capacity math when it changes the architecture. For example, if estimated writes are far below what a tuned single database can handle, I would avoid sharding initially and rely on indexes, caching, and read replicas before adding distributed complexity.

---

## 11. Core Concepts Cheat Sheet

| Concept | Use It When | Main Trade-Off |
|---|---|---|
| Networking | Services need to communicate | Simplicity vs latency vs connection state |
| API Design | Defining client-system interaction | Enough detail vs wasting interview time |
| Data Modeling | Deciding how data is stored | Normalized consistency vs denormalized read speed |
| Indexing | Queries are slow or frequent | Faster reads vs slower writes and more storage |
| Caching | Reads are hot or expensive | Lower latency vs stale data and invalidation complexity |
| Sharding | One database cannot handle scale | Horizontal scale vs cross-shard complexity |
| Consistent Hashing | Nodes are added/removed dynamically | Less data movement vs implementation complexity |
| CAP Theorem | Data is replicated across distributed nodes | Consistency vs availability during partitions |
| Numbers to Know | Architecture decision needs scale validation | Rough estimates vs over-analysis |

---

## 12. How Core Concepts Connect to Technologies and Patterns

Core concepts explain the reason behind technologies and patterns.

```text
Caching concept
      ↓
Redis / Memcached / CDN
      ↓
Scaling Reads pattern
```

```text
Sharding concept
      ↓
DynamoDB / Cassandra / sharded Postgres
      ↓
Scaling Writes pattern
```

```text
Networking concept
      ↓
HTTP / SSE / WebSockets / gRPC
      ↓
Realtime Updates pattern
```

```text
Consistency concept
      ↓
Transactions / locks / distributed coordination
      ↓
Contention and Multi-Step Process patterns
```

A strong answer connects all three levels:

```text
Concept → Technology → Pattern → Business requirement
```

Example:

> Because this is a read-heavy system, I would use caching as the core concept, Redis as the technology, and the Scaling Reads pattern to reduce database load for hot objects. The trade-off is stale data, so I would use TTLs and invalidate cache entries on writes.

---


# Part 3: Key Technologies

Based on the Hello Interview **System Design in a Hurry: Key Technologies** material.

## 1. Why Key Technologies Matter

System design assembles building blocks for a specific problem. Know at least one strong option in each major technology category.

Interviewers care less about choosing Kafka vs SQS (Simple Queue Service) and more about whether you understand queues, trade-offs, background processing, burst handling, and asynchronous workflows.

A good interview approach is:

```text
Understand requirements
        ↓
Pick simple building blocks first
        ↓
Explain why each technology is needed
        ↓
Discuss trade-offs and failure modes
        ↓
Scale only when the requirements demand it
```

Do not name-drop. Explain what each technology does, when to use it, and what problems it creates.

---

## 2. Overall Technology Categories

Most system design problems can be solved using a combination of the following categories:

| Category | Main Use |
|---|---|
| Core Database | Store structured application data |
| Relational Database | Store transactional data with SQL (Structured Query Language), joins, indexes, and ACID (Atomicity, Consistency, Isolation, Durability) transactions |
| NoSQL (Non-Relational / Not Only SQL) Database | Store flexible or high-scale data using key-value, document, column-family, or graph models |
| Blob Storage | Store large files such as images, videos, PDFs, and backups |
| Search Optimized Database | Support full-text search, ranking, tokenization, and fuzzy matching |
| API Gateway | Route client requests and handle cross-cutting concerns |
| Load Balancer | Distribute traffic across multiple machines |
| Queue | Buffer bursty traffic and distribute async work to workers |
| Streams / Event Sourcing | Process event logs, replay events, and support multiple consumers |
| Distributed Lock | Coordinate access to shared resources across services |
| Distributed Cache | Reduce latency and database load by storing hot data in memory |
| CDN (Content Delivery Network) | Cache content close to users globally |

---

## 3. Core Database

### What It Is

A core database is the main place where your application stores persistent data. Almost every system design problem needs some form of durable storage.

Common choices:

- **Relational database**: PostgreSQL, MySQL
- **NoSQL (Non-Relational / Not Only SQL) database**: DynamoDB, Cassandra, MongoDB
- **Blob storage**: S3, Google Cloud Storage, Azure Blob Storage

### How to Think About It in Interviews

Avoid generic SQL (Structured Query Language) vs NoSQL (Non-Relational / Not Only SQL) comparisons unless asked. Choose one database and explain why it fits the problem.

Good example:

> I would use Postgres here because the system needs transactional consistency for orders, payments, and inventory updates. Postgres gives us ACID (Atomicity, Consistency, Isolation, Durability) transactions, indexes for common queries, and strong data integrity.

Weak example:

> I need SQL because the data has relationships.

That answer is weak because NoSQL systems can also model relationships. Similarly, saying “I need NoSQL because I need scale” is too broad because relational databases can scale well with the right architecture.

### Interview Rule

Pick the database you understand best, explain the features you will use, and connect those features directly to the problem requirements.

---

## 4. Relational Databases

### What They Are

Relational databases store data in tables made of rows and columns. They are commonly used for transactional product systems such as users, orders, payments, bookings, inventory, and account records.

Example tables:

```text
users
- user_id
- name
- email
- created_at

orders
- order_id
- user_id
- status
- total_amount
- created_at
```

Relational databases use SQL.

### When to Use

Use a relational database when the system needs:

- Structured data
- Transactions
- Strong consistency
- Flexible querying
- Joins between related entities
- Data integrity constraints
- Indexes for different query patterns

Common interview examples:

- E-commerce order system
- Ticket booking system
- Payment system
- User account system
- Inventory management system
- SaaS application backend

### Important Features to Know

#### SQL Joins

Joins combine data from multiple tables.

Example:

```sql
SELECT users.name, orders.total_amount
FROM users
JOIN orders ON users.user_id = orders.user_id
WHERE users.user_id = 123;
```

Joins are powerful, but they can become expensive at scale. In interviews, mention that you may denormalize or precompute data if joins become a bottleneck.

#### Indexes

Indexes make reads faster by avoiding full table scans.

Example:

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

Indexes are useful for common queries like:

- Find orders by user ID
- Find products by category
- Find messages by conversation ID
- Find events by city and date

Trade-off: indexes speed up reads but add overhead to writes because the database must update the index whenever data changes.

#### Transactions

Transactions group multiple operations into one atomic unit.

Example:

```sql
BEGIN;
  UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 10;
  INSERT INTO orders (...) VALUES (...);
COMMIT;
```

If one step fails, the whole transaction can roll back. This prevents invalid states such as creating an order without reducing inventory.

### Common Technologies

- PostgreSQL
- MySQL

If you do not have a favorite, PostgreSQL is a safe interview choice because it supports strong transactional behavior, rich indexing, JSON (JavaScript Object Notation) fields, full-text search, and extensions like PostGIS.

### Interview Pitfalls

Do not overuse joins in a high-scale read path without discussing performance. Do not split data into multiple databases too early because you lose simple transactions and make consistency harder.

---

## 5. NoSQL (Non-Relational / Not Only SQL) Databases

### What They Are

NoSQL databases are a broad category of databases that support different data models beyond traditional relational tables.

Common NoSQL models:

| Model | Example Use |
|---|---|
| Key-value | Sessions, feature flags, simple lookups |
| Document | User profiles, product documents, flexible schemas |
| Column-family | High-write event data, time-series-like workloads |
| Graph | Social graphs, recommendations, relationship-heavy data |

NoSQL databases are often schema-flexible and designed for horizontal scaling.

### When to Use

Use a NoSQL database when the system benefits from:

- Flexible or evolving data models
- High horizontal scalability
- High write throughput
- Simple access patterns by key
- Large volumes of semi-structured or unstructured data
- Event or activity storage

Common examples:

- User activity feed storage
- Product catalog with flexible attributes
- High-volume event logging
- Chat message storage
- IoT event ingestion
- Session storage

### Important Features to Know

#### Data Models

NoSQL is not one single thing. DynamoDB, Cassandra, MongoDB, and graph databases behave differently. In interviews, be specific about which database style you are using and why.

#### Consistency Models

Some NoSQL databases support strong consistency, while others are eventually consistent by default.

```text
Strong consistency:
A read immediately reflects the latest successful write.

Eventual consistency:
Replicas may temporarily return stale data, but they converge over time.
```

Use strong consistency for payments, inventory, or critical state. Eventual consistency may be acceptable for likes, view counts, analytics, or feeds.

#### Indexing

NoSQL databases also support indexes, but they may be more limited than relational databases. You should design access patterns up front.

Example DynamoDB-style thinking:

```text
Access pattern: get all orders for a user
Partition key: user_id
Sort key: created_at
```

#### Scalability

NoSQL systems commonly scale using partitioning, sharding, and consistent hashing.

### Common Technologies

- DynamoDB
- Cassandra
- MongoDB

DynamoDB is a strong interview choice for managed key-value/document workloads. Cassandra is often used for write-heavy workloads. MongoDB is common for document-style flexible data.

### Interview Pitfalls

Do not say “NoSQL is always better for scale.” That is too broad. Instead, explain the specific access pattern, partition key, consistency needs, and scaling model.

---

## 6. Blob Storage

### What It Is

Blob storage is used for large unstructured files such as images, videos, audio files, PDFs, exports, and backups.

Examples:

- Amazon S3
- Google Cloud Storage
- Azure Blob Storage

### When to Use

Use blob storage when you need to store large binary objects that would be inefficient or expensive to store directly in a database.

Common examples:

| System | Blob Storage Use | Database Use |
|---|---|---|
| YouTube | Store video files | Store metadata, title, owner, status |
| Instagram | Store images and videos | Store captions, likes, comments, user metadata |
| Dropbox | Store uploaded files | Store folders, permissions, file metadata |

### Core Design Idea

Do not route large file bytes through your application servers.

Bad design:

```text
Client → App Server → Blob Storage
```

Better design:

```text
Client → Blob Storage
Client → CDN → Blob Storage
```

The app server should issue temporary permissions, not handle file bytes.

### Upload Flow

```text
Client
  ↓ asks for upload permission
API Server
  ↓ validates user and creates metadata
Database
  ↓ stores file record with status = pending
API Server
  ↓ returns presigned upload URL
Client
  ↓ uploads directly
Blob Storage
  ↓ upload-complete event
Worker / Backend
  ↓ marks file as uploaded
Database
```

### Download Flow

```text
Client
  ↓ requests file
API Server
  ↓ checks permissions
API Server
  ↓ returns signed CDN/blob URL
Client
  ↓ downloads from CDN or blob storage
```

### Important Features to Know

#### Durability

Blob storage replicates data for high durability. In interviews, treat managed blob storage as durable and scalable.

#### Cost

Blob storage is much cheaper than storing large files in a database. Store only metadata and pointers in the core database.

#### Security

Use encryption, access policies, presigned URLs, signed download URLs, and short expiration times.

#### Chunking / Multipart Upload

Large uploads should be split into chunks so failed uploads can resume instead of restarting from zero.

### Interview Pitfalls

Use S3 or blob storage for large objects, not as the primary low-latency query database.

---

## 7. Search Optimized Database

### What It Is

A search optimized database is designed for full-text search, fuzzy matching, ranking, and fast lookup across large text datasets.

Common technology:

- Elasticsearch

Other options:

- OpenSearch
- Apache Solr
- PostgreSQL full-text search with GIN indexes

### Why Normal Database Search Is Not Enough

This query is slow at scale:

```sql
SELECT * FROM documents
WHERE document_text LIKE '%search_term%';
```

This can require scanning many rows, especially when the search term appears anywhere inside a large text field.

### How Search Databases Work

Search systems commonly use an **inverted index**.

Instead of storing only document → words, they store word → documents.

```json
{
  "pizza": ["doc1", "doc5", "doc9"],
  "concert": ["doc2", "doc4"],
  "running": ["doc3", "doc7"]
}
```

Now the system can quickly find documents containing a word without scanning every document.

### When to Use

Use a search database when the system needs:

- Full-text search
- Ranking by relevance
- Fuzzy search
- Autocomplete
- Filtering and faceting
- Searching large text fields
- Searching user-generated content

Common examples:

- Search events in Ticketmaster
- Search tweets or posts
- Search products in an e-commerce system
- Search documents in Google Drive or Dropbox
- Search restaurants by name, cuisine, or description

### Important Features to Know

#### Tokenization

Tokenization splits text into searchable terms.

```text
"The quick brown fox"
        ↓
["the", "quick", "brown", "fox"]
```

#### Stemming

Stemming reduces words to their root form.

```text
running, runs, runner → run
```

#### Fuzzy Search

Fuzzy search handles misspellings or near matches.

```text
"iphnoe" can still match "iphone"
```

#### Scaling

Search databases scale by sharding indexes across nodes and replicating shards for availability and read throughput.

### Interview Pitfalls

Do not use a search database as the source of truth for transactional data. Usually, the core database is the source of truth, and Elasticsearch is updated asynchronously for search.

Common architecture:

```text
Application write
      ↓
Core Database
      ↓ change event / CDC / queue
Search Index
```

---

## 8. API Gateway

### What It Is

An API gateway is the first backend entry point for client requests. It routes incoming requests to the correct backend service and handles cross-cutting concerns.

```text
Client
  ↓
API Gateway
  ↓
Backend Services
```

### When to Use

In most product-style system design interviews, it is reasonable to include an API gateway in front of your backend services.

Use it for:

- Request routing
- Authentication
- Authorization
- Rate limiting
- Logging
- Request validation
- TLS termination
- API versioning
- Response aggregation in some cases

### Example

```text
GET /users/123       → User Service
GET /orders/abc      → Order Service
POST /payments       → Payment Service
GET /search?q=phone  → Search Service
```

### Common Technologies

- AWS API Gateway
- Kong
- Apigee
- NGINX
- Apache

### Interview Pitfalls

Keep gateway discussion brief unless the problem focuses on routing, authentication, rate limiting, or API management.

---

## 9. Load Balancer

### What It Is

A load balancer distributes traffic across multiple machines that can handle the same type of request.

```text
Client Requests
      ↓
Load Balancer
  ↓       ↓       ↓
Server  Server  Server
```

### When to Use

Use a load balancer whenever a service is horizontally scaled across multiple instances.

Common use cases:

- Distribute API traffic
- Avoid overloading one server
- Improve availability
- Remove unhealthy servers from rotation
- Support zero-downtime deployments

### Interview Guidance

Draw one front-door load balancer and mention internal horizontal scaling when needed.

### L4 (Layer 4 / Transport Layer) vs L7 (Layer 7 / Application Layer) Load Balancing

| Type | Works At | Good For |
|---|---|---|
| L4 (Layer 4 / Transport Layer) Load Balancer | TCP/UDP (User Datagram Protocol) connection level | WebSockets, persistent connections, high throughput |
| L7 (Layer 7 / Application Layer) Load Balancer | HTTP/application level | Path-based routing, header-based routing, flexible API routing |

Simple rule:

```text
Persistent connections like WebSockets → often L4
Normal HTTP APIs → L7
```

### Common Technologies

- AWS Elastic Load Balancer
- NGINX
- HAProxy
- Cloud load balancers

### Interview Pitfalls

Do not overfocus on the load balancer unless it matters to the problem. Mention health checks, horizontal scaling, and whether sticky sessions or persistent connections are required.

---

## 10. Queue

### What It Is

A queue stores messages between producers and consumers. Producers put work into the queue, and workers consume messages at their own pace.

```text
Producer / API Server
        ↓
      Queue
        ↓
   Worker Pool
```

### When to Use

Use queues for:

- Long-running background tasks
- Bursty traffic
- Asynchronous processing
- Work distribution
- Retryable jobs
- Decoupling producers from consumers

Common examples:

- Image processing
- Video transcoding
- Email sending
- Report generation
- Payment webhook processing
- Bulk imports

### Queue as a Buffer

If the system receives a sudden spike, the queue can absorb work temporarily.

```text
Traffic spike: 1000 requests/sec
Worker capacity: 200 jobs/sec
Extra jobs wait in queue instead of being dropped immediately
```

But this only helps if the spike is temporary. If the system constantly receives more work than it can process, the queue will grow forever.

### Important Features to Know

#### Message Ordering

Many queues provide FIFO (First In, First Out) ordering, but ordering may only be guaranteed within a partition or message group.

#### Retries

Failed messages can be retried automatically. Use exponential backoff to avoid retry storms.

#### Dead Letter Queue

Messages that fail repeatedly should go to a dead letter queue for debugging or manual handling.

```text
Queue → Worker → Failure → Retry
                    ↓ after max retries
              Dead Letter Queue
```

#### Partitioning

Queues can scale by partitioning messages. Choose a partition key that keeps related messages ordered.

Example:

```text
Chat messages partitioned by conversation_id
Order events partitioned by order_id
```

#### Backpressure

Backpressure prevents the system from accepting unlimited work when downstream systems are overloaded.

Examples:

- Reject new requests
- Return 429 Too Many Requests
- Slow producers down
- Drop non-critical messages
- Increase worker capacity if possible

### Common Technologies

- Kafka
- AWS SQS (Amazon Web Services Simple Queue Service)
- RabbitMQ
- Google Pub/Sub

### Interview Pitfalls

Do not introduce a queue into a strongly synchronous path with strict latency requirements. If the user expects a response in under 500 ms, a queue may break that requirement.

---

## 11. Streams and Event Sourcing

### What Streams Are

A stream is an append-only event log. Unlike simple queues, streams retain data for replay.

```text
Event 1 → Event 2 → Event 3 → Event 4 → Event 5
                 ↑
          Consumer can resume or replay
```

### When to Use Streams

Use streams when you need:

- Real-time event processing
- Multiple consumers reading the same events
- Event replay
- Analytics pipelines
- Event sourcing
- Audit trails
- High-volume data ingestion

Common examples:

- Real-time analytics dashboard
- Banking transaction event log
- Chat message fanout
- Activity feed generation
- Fraud detection pipeline
- Metrics aggregation

### Event Sourcing

Event sourcing stores changes as events instead of only storing the latest state.

Example account events:

```text
AccountCreated
MoneyDeposited +100
MoneyWithdrawn -20
MoneyTransferred -50
```

The current balance can be reconstructed by replaying events.

```text
Initial balance: 0
+100
-20
-50
Current balance: 30
```

### Queue vs Stream

| Queue | Stream |
|---|---|
| Message is consumed and removed or hidden | Events are retained for replay |
| Good for background jobs | Good for event logs and real-time pipelines |
| Usually one consumer group handles a job | Multiple consumer groups can read independently |
| Focuses on work distribution | Focuses on event history and processing |

### Important Features to Know

#### Partitioning

Streams scale through partitions. Events with the same key go to the same partition to preserve ordering.

```text
partition_key = user_id
partition_key = order_id
partition_key = conversation_id
```

#### Consumer Groups

Multiple consumer groups can process the same stream independently.

```text
Order Events Stream
    ↓
Consumer Group A: update search index
Consumer Group B: update analytics
Consumer Group C: send notifications
```

#### Replication

Streams replicate partitions across brokers for fault tolerance.

#### Windowing

Windowing groups events by time or count.

Example:

```text
Calculate average delivery time per region every 5 minutes
```

### Common Technologies

- Kafka
- Kinesis
- Flink
- Spark Streaming

### Interview Pitfalls

Do not use event sourcing unless the system benefits from auditability, replay, or reconstructing state. It adds complexity around schema evolution, ordering, idempotency, and replay correctness.

---

## 12. Distributed Lock

### What It Is

A distributed lock coordinates access to a shared resource across multiple machines, services, or processes.

Use it when a normal database transaction is not enough because the lock must be visible across distributed workers or services.

```text
Service A tries to lock ticket-123 → success
Service B tries to lock ticket-123 → blocked or fails
```

### When to Use

Use distributed locks for:

- Temporarily holding a concert ticket
- Reserving limited inventory during checkout
- Preventing a driver from being matched to two riders
- Ensuring only one server runs a scheduled job
- Coordinating critical updates across distributed workers

### Common Technologies

- Redis
- ZooKeeper
- etcd
- DynamoDB conditional writes

### Lock Expiry

Distributed locks need a TTL so crashed processes do not hold locks forever.

Example:

```text
Lock ticket-123 for 10 minutes
If checkout completes → release lock
If process crashes → lock expires automatically
```

### Lock Granularity

Granularity means how much data one lock protects.

```text
Fine-grained lock: one ticket
Coarse-grained lock: entire event section
Very coarse lock: entire event
```

Fine-grained locks allow more concurrency but are harder to manage. Coarse locks are simpler but reduce throughput.

### Deadlocks

A deadlock can happen when two processes wait on each other’s locks.

```text
Process A holds Lock 1 and waits for Lock 2
Process B holds Lock 2 and waits for Lock 1
```

Ways to reduce deadlock risk:

- Acquire locks in a consistent order
- Use lock timeouts
- Keep lock duration short
- Avoid locking far-apart parts of the system
- Prefer database atomic operations when possible

### Interview Pitfalls

Do not choose distributed locks as the first solution if a database transaction or atomic update can solve the problem. Distributed locks are useful, but they add operational and correctness complexity.

---

## 13. Distributed Cache

### What It Is

A distributed cache stores frequently accessed or expensive-to-compute data in memory across a cluster of machines.

```text
Client
  ↓
API Server
  ↓
Cache
  ↓ if miss
Database
```

### When to Use

Use a distributed cache to:

- Reduce database load
- Lower latency
- Store session data
- Store expensive query results
- Store precomputed dashboards or metrics
- Cache hot objects such as popular events or posts

Common examples:

- User session cache
- Product catalog cache
- Popular events cache
- Feed cache
- Rate limiting counters
- Aggregated analytics dashboard data

### Common Cache Strategies

#### Cache-Aside

The application checks the cache first. On a miss, it queries the database and stores the result in the cache.

```text
Read request
  ↓
Check cache
  ↓
Hit → return cached value
Miss → query DB → store in cache → return value
```

#### Write-Through

Writes go to both the cache and the database before returning success.

Good for stronger consistency, but writes are slower.

#### Write-Around

Writes go directly to the database and bypass the cache. This avoids polluting the cache with data that may not be read again.

#### Write-Back

Writes go to the cache first and are later flushed to the database asynchronously.

This can improve write performance but risks data loss if the cache fails before persistence.

### Eviction Policies

When the cache is full, it needs to evict data.

| Policy | Meaning |
|---|---|
| LRU (Least Recently Used) | Remove least recently used items |
| LFU (Least Frequently Used) | Remove least frequently used items |
| FIFO (First In, First Out) | Remove oldest inserted items first |

### Cache Invalidation

Cache invalidation decides how stale data is removed or updated.

Common approaches:

- TTL expiration
- Invalidate on write
- Refresh asynchronously
- Versioned cache keys
- Event-driven invalidation

### Data Structures Matter

Specify the Redis data structure, not only “store it in Redis.”

Examples:

```text
String: user session token
Hash: user profile fields
Sorted set: leaderboard or popular events
Set: followers or unique user IDs
Counter: rate limiting or metrics
```

### Common Technologies

- Redis
- Memcached

### Interview Pitfalls

Caching improves performance but creates consistency problems. Always explain what is cached, how long it is cached, how it is invalidated, and what happens on cache miss.

---

## 14. CDN (Content Delivery Network)

### What It Is

A CDN (Content Delivery Network) is a globally distributed cache that serves content from locations close to users.

```text
User
  ↓
Nearest CDN Edge
  ↓ if cache miss
Origin Server / Blob Storage
```

### When to Use

Use a CDN when the system serves users across regions and needs low-latency content delivery.

Common use cases:

- Images
- Videos
- Static HTML/CSS/JavaScript
- Profile pictures
- Public documents
- Frequently accessed API responses
- Downloadable files

### How It Works

1. User requests content.
2. CDN routes the request to a nearby edge location.
3. If the edge has the content, it returns it immediately.
4. If not, it fetches the content from origin storage or the origin server.
5. CDN caches the response for future users.

### Static vs Dynamic Content

CDNs are commonly used for static assets, but they can also cache dynamic content if it changes infrequently.

Examples:

```text
Static: profile images, videos, CSS, JS
Dynamic but cacheable: public event pages, blog posts, product detail pages
```

### Important Features to Know

- TTL-based caching
- Cache invalidation
- Edge locations
- Origin fetch
- Signed URLs
- DDoS protection
- Web application firewall support

### Common Technologies

- Cloudflare
- Akamai
- Amazon CloudFront
- Google Cloud CDN

### Interview Pitfalls

When using a CDN, specify cacheable content. Avoid public caching for user-specific or sensitive data unless using private caching, signed URLs, or strict cache controls.

---

## 15. Key Technologies Cheat Sheet

| Requirement | Technology to Consider |
|---|---|
| Store transactional data | Relational database such as Postgres or MySQL |
| Store flexible high-scale data | NoSQL database such as DynamoDB, Cassandra, or MongoDB |
| Store large files | Blob storage such as S3 or Google Cloud Storage |
| Search text by relevance | Elasticsearch or another search index |
| Route external requests | API gateway |
| Distribute traffic across servers | Load balancer |
| Process work asynchronously | Queue |
| Process and replay events | Stream such as Kafka or Kinesis |
| Coordinate access to shared resources | Distributed lock |
| Reduce read latency and DB load | Distributed cache such as Redis |
| Serve global static content | CDN |

---

## 16. How to Talk About Technologies in Interviews

Strong answer structure:

```text
I am choosing [technology]
because the system needs [requirement].
This helps by [benefit].
The main trade-offs are [trade-offs].
If scale increases, I would handle it by [scaling strategy].
```

Example:

> I would use Redis as a distributed cache for popular event metadata because event pages may receive very high read traffic during ticket launches. This reduces database load and improves latency. I would use TTL-based expiration plus event-driven invalidation when event details change. The main risk is stale data, so the booking path would still verify seat availability against the source-of-truth database.

---

# Part 4: Common Patterns

## System Design Common Patterns — Revision Notes

Based on the Hello Interview **System Design in a Hurry: Common Patterns** material.

---

## 1. Why Patterns Matter in System Design

Patterns help identify the problem type and apply a known architecture quickly.

Most real systems use multiple patterns together. For example, a video platform may use:

```text
Large Blob Uploads
      ↓
Long-Running Video Processing
      ↓
Realtime Progress Updates
      ↓
Multi-Step Workflow Coordination
```

Do not memorize fixed architectures. Identify the problem type, explain trade-offs, start simple, and add complexity only when required.

---

## 2. Pushing Realtime Updates

### When to Use

Use this pattern when the system needs to send updates to users as events happen, instead of waiting for the user to refresh or make another request.

Common examples:

- Chat applications
- Live notifications
- Live dashboards
- Collaborative editing
- Order tracking
- Ride-sharing driver location updates

### Core Problem

A normal HTTP request works well when the client asks for something and the server immediately returns a response.

But in realtime systems, the server may need to notify the client later.

Example:

```text
User A sends message
        ↓
Server receives message
        ↓
User B should see message instantly
```

### Common Approaches

#### HTTP Polling

The client repeatedly asks the server for updates.

```text
Client → Server: Any new messages?
Client → Server: Any new messages?
Client → Server: Any new messages?
```

This is simple and reliable, but inefficient at scale because many requests may return no new data.

Use polling when:

- Realtime requirement is weak
- Updates are infrequent
- System is simple
- You want the easiest starting point

#### SSE (Server-Sent Events)

SSE allows the server to push one-way updates to the client over a persistent connection.

```text
Server ───────► Client
```

Good for:

- Notifications
- Live feeds
- Status updates
- Progress updates

Not ideal when the client also needs to frequently send realtime messages back over the same connection.

#### WebSockets

WebSockets provide a two-way persistent connection.

```text
Client ⇄ Server
```

Good for:

- Chat
- Gaming
- Collaborative editing
- Realtime bidding
- Live cursors

The downside is operational complexity. WebSocket servers are stateful because each server holds active connections.

### Server-Side Architecture

Realtime systems need routing from events to connected users.

```text
User Action
   ↓
API Server
   ↓
Pub/Sub or Message Broker
   ↓
Realtime Gateway / WebSocket Server
   ↓
Connected Client
```

### Common Technologies

- WebSockets
- Server-Sent Events
- HTTP polling
- Redis Pub/Sub
- Kafka
- Google Pub/Sub
- AWS SNS (Amazon Web Services Simple Notification Service) / SQS (Simple Queue Service)
- Load balancers with sticky sessions
- Consistent hashing

### Key Trade-Offs

- Polling is simple but inefficient.
- WebSockets are powerful but harder to scale.
- SSE is simpler than WebSockets but only supports server-to-client streaming.
- Pub/Sub helps decouple event producers from realtime delivery servers.

### Interview Pitfalls

Do not start with WebSockets automatically. Start with polling if realtime needs are loose, then upgrade to SSE or WebSockets when latency or interactivity requires it.

Mention connection management, reconnection, message ordering, retries, and how to scale stateful realtime servers.

---

## 3. Managing Long-Running Tasks

### When to Use

Use this pattern when a task takes too long to complete during a normal user request.

Common examples:

- Video transcoding
- PDF generation
- Report generation
- Large file processing
- Bulk imports
- Image resizing
- ML (Machine Learning) inference jobs
- Email campaigns

### Core Problem

Synchronous APIs should return quickly. Long requests risk timeouts and poor UX.

Instead of doing the work inside the request, the system accepts the task and processes it later.

### Standard Architecture

```text
Client
  ↓
API Server
  ↓
Job Queue
  ↓
Worker Pool
  ↓
Database / Blob Storage
  ↓
Status API or Realtime Update
```

### Typical Flow

1. Client submits job.
2. API validates the request.
3. API stores job metadata.
4. API pushes job into a queue.
5. API immediately returns a `job_id`.
6. Worker picks up the job.
7. Worker processes the task.
8. Worker updates job status.
9. Client checks status or receives notification.

Example response:

```json
{
  "job_id": "abc123",
  "status": "queued"
}
```

### Job Statuses

A typical job lifecycle may look like:

```text
QUEUED → RUNNING → SUCCEEDED
              ↓
            FAILED
              ↓
           RETRYING
```

### Key Components

- Message queue: coordinates work
- Worker pool: executes jobs
- Job table: stores status and metadata
- Retry logic: handles transient failures
- Dead letter queue: stores failed poison jobs
- Status endpoint: lets users track progress

### Important Design Choice

Do not push everything into a queue automatically.

If the task is short and can complete quickly, synchronous processing may be better because it gives clearer backpressure and a simpler user experience.

Use async processing when:

- Task takes more than a few seconds
- Task may fail and need retries
- Task requires independent scaling
- Task should not block user requests
- Task has unpredictable runtime

### Failure Handling

You should discuss:

- Retries with exponential backoff
- Idempotency keys
- Dead letter queues
- Job timeout handling
- Partial failure recovery
- Worker crashes
- Duplicate job execution

### Interview Pitfalls

A common mistake is adding a queue too early. Queues improve scalability and reliability for heavy work, but they add complexity around retries, ordering, duplicate processing, and user-visible status.

---

## 4. Dealing with Contention

### When to Use

Use this pattern when multiple users or services may try to update the same resource at the same time.

Common examples:

- Booking the last concert ticket
- Buying the last item in inventory
- Auction bidding
- Bank transfers
- Seat reservations
- Coupon redemption
- Distributed counters

### Core Problem

Without coordination, two users may both believe they successfully claimed the same resource.

Example race condition:

```text
User A checks ticket availability → 1 ticket left
User B checks ticket availability → 1 ticket left

User A books ticket
User B books ticket

Result: oversold ticket
```

### Main Solutions

#### Database Transactions

Use database transactions when the resource is stored in one database.

```sql
BEGIN;
  -- check availability
  -- update availability
  -- create booking
COMMIT;
```

This is the simplest starting point.

#### Pessimistic Locking

Lock the row before updating it.

```sql
SELECT * FROM tickets WHERE id = 123 FOR UPDATE;
```

This prevents other transactions from modifying the same row until the lock is released.

Good when conflicts are frequent. Bad when locks are held for too long.

#### Optimistic Concurrency Control

Allow reads, but detect conflicts when writing.

```sql
UPDATE inventory
SET quantity = quantity - 1,
    version = version + 1
WHERE item_id = 123
AND version = 7;
```

If no rows are updated, another request modified the data first.

Good when conflicts are rare.

#### Atomic Operations

Use atomic database operations when possible.

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE item_id = 123
AND quantity > 0;
```

This prevents overselling without requiring application-level locks.

#### Distributed Locks

Use distributed locks only when coordination must happen across multiple servers or services.

Examples:

- Redis lock
- ZooKeeper
- etcd
- DynamoDB conditional lock

Distributed locks are more complex and can fail in subtle ways, so they should not be the first choice unless truly needed.

#### Queue-Based Serialization

Route all updates for a specific resource through a single queue or partition.

```text
All booking requests for Event 123
        ↓
Same queue partition
        ↓
Processed one at a time
```

This can simplify consistency but may increase latency.

### Interview Pitfalls

Do not split data across multiple databases too early. Databases already provide transactions, atomicity, and locking. Once you distribute the data, you inherit many hard consistency problems yourself.

### What to Say in Interview

> I would first try to solve contention inside the database using transactions, row-level locking, conditional updates, or optimistic concurrency. If the resource becomes distributed or extremely high-throughput, then I would consider queue-based serialization or distributed locks.

---

## 5. Scaling Reads

### When to Use

Use this pattern when the system has much more read traffic than write traffic.

Common examples:

- Social media feed
- Product catalog
- User profiles
- News feed
- Video metadata
- Search results
- Public dashboards

### Core Problem

Read traffic often grows faster than write traffic. A single database may not handle millions of repeated read requests efficiently.

Example:

```text
One user posts a photo       → 1 write
Thousands of users view it   → thousands of reads
```

### Natural Progression

#### Step 1: Add Indexes

Indexes make queries faster by avoiding full table scans.

```text
Query: get all posts by user_id
Index: user_id
```

Use indexes for common query patterns.

Be careful because indexes improve reads but add overhead to writes.

#### Step 2: Denormalize Data

Store duplicated or precomputed data to avoid expensive joins.

Example:

```text
Instead of joining posts + users every time,
store author_name and author_profile_image inside post metadata.
```

This improves reads but makes updates more complex.

#### Step 3: Add Read Replicas

Read replicas copy data from the primary database and serve read traffic.

```text
Writes → Primary DB
Reads  → Read Replica 1
Reads  → Read Replica 2
Reads  → Read Replica 3
```

This scales reads horizontally.

Main issue: replication lag. A user may write something and not immediately see it on a replica.

#### Step 4: Add Caching

Use cache for frequently requested data.

```text
Client
  ↓
API Server
  ↓
Redis Cache
  ↓
Database
```

If data is in cache, return it quickly. If not, fetch from database and populate the cache.

#### Step 5: Use CDN for Static or Public Content

For images, videos, scripts, and public content, use a CDN.

```text
User → CDN Edge → Origin Storage
```

CDNs reduce latency and protect the origin service.

### Common Cache Strategies

#### Cache-Aside

Application checks cache first.

```text
Read request
  ↓
Check cache
  ↓
Cache hit  → return data
Cache miss → query DB → store in cache → return data
```

#### Write-Through

Application writes to cache and database together.

#### TTL-Based Expiration

Cache entries expire after a fixed time.

#### Invalidation

When data changes, remove or update the cache entry.

### Key Challenges

- Cache invalidation
- Hot keys
- Stale reads
- Replication lag
- Cache stampede
- Choosing what to cache

### Hot Key Problem

A hot key occurs when one item receives massive traffic.

Examples:

- Celebrity post
- Viral video
- Flash sale product
- Breaking news article

Solutions include:

- Cache replication
- Request coalescing
- CDN caching
- Sharding hot keys
- Pre-warming cache

### Interview Pitfalls

For Redis, explain cached data, cache keys, TTL, invalidation, and cache-miss behavior.

---

## 6. Scaling Writes

### When to Use

Use this pattern when a single database or storage system cannot handle the write volume.

Common examples:

- High-volume logging
- Metrics ingestion
- Chat messages
- Payment events
- IoT events
- Clickstream tracking
- Social media posts

### Core Problem

Writes are harder to scale than reads because they change system state. You must think about consistency, ordering, durability, and partitioning.

### Main Strategies

#### Vertical Partitioning

Separate different types of data into different databases or tables.

```text
User DB
Order DB
Inventory DB
Analytics DB
```

This helps when different data types have different access patterns.

#### Horizontal Sharding

Split the same type of data across multiple database nodes.

```text
Shard 1: users 1 - 1M
Shard 2: users 1M - 2M
Shard 3: users 2M - 3M
```

Better approach: hash-based partition key:

```text
shard_id = hash(user_id) % number_of_shards
```

### Choosing a Partition Key

A good partition key should:

- Distribute traffic evenly
- Keep related data together
- Avoid hot partitions
- Support common query patterns
- Be stable over time

Examples:

```text
Good: user_id for user-owned data
Good: conversation_id for chat messages
Good: order_id for independent order records

Risky: timestamp only, because recent writes may all hit the same shard
Risky: country, because one country may dominate traffic
Risky: celebrity_id, because popular users create hot shards
```

### Write Burst Handling

Sometimes write traffic spikes suddenly.

Examples:

- Flash sale
- Breaking news
- Viral event
- System retry storm

Solutions:

- Queue writes temporarily
- Batch writes
- Apply rate limits
- Use load shedding
- Prioritize critical writes

### Batching

Batching groups many writes together to reduce overhead.

```text
Single writes:
write 1
write 2
write 3

Batched write:
write [1, 2, 3]
```

Useful for:

- Analytics events
- Logs
- Metrics
- Bulk imports
- Notification delivery

### Load Shedding

During overload, the system intentionally drops or delays lower-priority work.

Example:

```text
Keep payment writes
Delay analytics writes
Drop non-critical tracking events
```

### Interview Pitfalls

Sharding adds complexity. You need to explain:

- How data is routed to shards
- How resharding works
- How cross-shard queries work
- How hot partitions are handled
- How transactions work across shards

Start with a single database. Add sharding only when the write volume justifies it.

---

## 7. Handling Large Blobs

### When to Use

Use this pattern when the system stores or serves large files.

Common examples:

- Images
- Videos
- PDFs
- Documents
- Audio files
- Backups
- Large exports

### Core Problem

Application servers should not handle large file uploads and downloads directly because they can become bandwidth bottlenecks.

Bad design:

```text
Client → App Server → Blob Storage
```

The app server wastes CPU, memory, and network bandwidth passing large files through itself.

Better design:

```text
Client → Blob Storage
Client → CDN → Blob Storage
```

### Standard Upload Flow

1. Client asks API server for upload permission.
2. API server validates user permissions.
3. API server creates metadata record.
4. API server returns a presigned upload URL.
5. Client uploads directly to blob storage.
6. Blob storage emits event after upload completes.
7. Backend updates file status.

Diagram:

```text
Client
  ↓ request upload
API Server
  ↓ returns presigned URL
Client
  ↓ direct upload
Blob Storage
  ↓ event
Worker / Backend
  ↓ update metadata
Database
```

### Standard Download Flow

1. Client requests file.
2. API checks authorization.
3. API returns signed download URL or CDN URL.
4. Client downloads from CDN/blob storage.

### Common Technologies

- Amazon S3
- Google Cloud Storage
- Azure Blob Storage
- CloudFront
- Cloud CDN
- Signed URLs
- Presigned URLs
- Multipart upload
- Resumable upload

### Key Challenges

- Upload failure
- Large file retries
- Resumable uploads
- File metadata consistency
- Access control
- Virus scanning
- Lifecycle cleanup
- CDN caching
- Deleting orphaned files

### Metadata vs Blob Storage

Usually, the database stores metadata, not the file itself.

```text
Database:
file_id
owner_id
file_name
file_size
content_type
storage_url
status

Blob Storage:
actual file bytes
```

### Interview Pitfalls

Upload large videos with presigned URLs and direct-to-storage uploads.

Also mention how you keep database metadata and blob storage synchronized.

---

## 8. Multi-Step Processes

### When to Use

Use this pattern when a business workflow has many steps, external dependencies, retries, and failure scenarios.

Common examples:

- Order checkout
- Payment processing
- User onboarding
- Ride completion
- Food delivery workflow
- Travel booking
- Loan approval
- Subscription cancellation

### Core Problem

Complex workflows may span multiple services and take time to complete. Failures can happen at any step.

Example order workflow:

```text
Create order
  ↓
Reserve inventory
  ↓
Charge payment
  ↓
Create shipment
  ↓
Send confirmation
```

If payment succeeds but shipment creation fails, the system must know what to do next.

### Simple Orchestration

One service controls the workflow.

```text
Order Service
  ↓
Inventory Service
  ↓
Payment Service
  ↓
Shipping Service
  ↓
Notification Service
```

This is easier to understand but can become a large coordinator.

### Event-Driven Choreography

Each service emits events, and other services react.

```text
OrderCreated
  ↓
InventoryReserved
  ↓
PaymentCharged
  ↓
ShipmentCreated
  ↓
NotificationSent
```

This decouples services but makes the workflow harder to trace.

### Workflow Engine

Use a durable workflow system when the process is complex and must survive failures.

Examples:

- Temporal
- AWS Step Functions
- Cadence
- Airflow for data workflows

A workflow engine manages:

- State
- Retries
- Timeouts
- Compensation
- Audit history
- Failure recovery

### Compensation

In distributed systems, you may not be able to roll back everything with a database transaction. Instead, you perform a compensating action.

Example:

```text
Payment charged
Shipment failed
        ↓
Refund payment
Release inventory
Mark order failed
```

### Saga Pattern

A saga breaks a large distributed transaction into smaller local transactions with compensating actions.

```text
Step 1: Reserve inventory
Step 2: Charge payment
Step 3: Create shipment

If Step 3 fails:
Compensate Step 2: refund payment
Compensate Step 1: release inventory
```

### Interview Pitfalls

Do not pretend distributed transactions are easy. Explain how you handle retries, idempotency, partial failure, compensation, and auditability.

Strong answer: mention durable state, not only “call services one by one.”

---

## 9. Proximity-Based Services

### When to Use

Use this pattern when users search for nearby entities.

Common examples:

- Uber drivers near rider
- Restaurants near user
- Nearby stores
- Delivery couriers
- Nearby friends
- Local events
- Available parking spots

### Core Problem

Naively scanning every entity is too slow when there are millions of locations.

Bad approach:

```text
For every driver in the world:
    calculate distance from user
```

Better approach:

```text
Search only nearby geographic regions
```

### Common Solutions

#### Geospatial Indexes

Use a database or search engine that supports geo queries.

Examples:

- PostgreSQL + PostGIS
- Redis geospatial indexes
- Elasticsearch geo queries
- MongoDB geospatial indexes

#### Divide the World into Cells

Break geography into regions and search nearby cells.

```text
+-----+-----+-----+
|     |     |     |
+-----+--X--+-----+
|     |User |     |
+-----+-----+-----+
|     |     |     |
+-----+-----+-----+
```

The system first checks the user’s current cell, then expands to neighboring cells if needed.

#### Geohash

A geohash converts latitude and longitude into a string. Nearby locations often share a common prefix.

Example:

```text
User geohash: 9q9hv
Nearby places may share prefix: 9q9
```

### Important Insight

A geospatial index is not always necessary.

If the dataset is small, such as 1,000 stores, scanning all items may be simpler and fast enough.

Use geo indexing when:

- There are hundreds of thousands or millions of locations
- Low latency is required
- Users frequently query nearby entities
- Entities move frequently

### Key Challenges

- Updating moving entities
- Handling dense areas
- Handling sparse areas
- Expanding search radius
- Ranking by distance and availability
- Filtering by status
- Avoiding hot regions

### Interview Pitfalls

Do not over-engineer. For a small number of static locations, a simple database query or scan may be enough. For Uber-scale moving drivers, use geospatial indexing and frequent location updates.

---

## 10. Pattern Selection Cheat Sheet

| Requirement | Likely Pattern |
|---|---|
| Users need instant updates | Pushing Realtime Updates |
| Task takes too long for request-response | Managing Long-Running Tasks |
| Multiple users update same resource | Dealing with Contention |
| Too many read requests | Scaling Reads |
| Too many write requests | Scaling Writes |
| System stores videos, images, or files | Handling Large Blobs |
| Workflow has many steps and failures | Multi-Step Processes |
| Search nearby drivers, stores, or restaurants | Proximity-Based Services |

---

## 11. How Patterns Combine in Real Designs

### Example: Design YouTube Upload

```text
Large Blobs:
User uploads video directly to blob storage.

Long-Running Tasks:
Workers transcode video in the background.

Multi-Step Processes:
Workflow coordinates upload, transcoding, thumbnail generation, metadata update, and publishing.

Realtime Updates:
User receives progress updates.

Scaling Reads:
CDN serves videos globally.
```

### Example: Design Uber

```text
Proximity-Based Services:
Find nearby drivers.

Realtime Updates:
Show driver location updates.

Contention:
Prevent the same driver from accepting two rides.

Multi-Step Processes:
Coordinate ride request, driver match, pickup, trip, payment, and receipt.
```

### Example: Design Ticketmaster

```text
Contention:
Prevent overselling seats.

Scaling Reads:
Cache event pages and seat maps.

Scaling Writes:
Handle massive booking spikes.

Long-Running Tasks:
Process payment or ticket generation asynchronously if needed.
```

---

## 12. Interview Answer Templates

### Realtime Updates

> This problem has a realtime update requirement, so I would start with polling for simplicity. If the product requires low-latency updates, I would move to SSE or WebSockets. For scaling, I would decouple event generation from delivery using Pub/Sub and use realtime gateway servers to manage client connections.

### Long-Running Tasks

> This operation should not block the request thread. I would accept the request, validate it, store a job record, enqueue the task, and return a job ID. Workers would process the job asynchronously, update status, retry transient failures, and send failed poison jobs to a dead letter queue.

### Contention

> Because multiple users may update the same resource, I would first rely on database transactions or conditional updates. If conflicts become distributed or very high volume, I would consider queue-based serialization or distributed locks.

### Scaling Reads

> I would first optimize queries with indexes and denormalization, then add read replicas, then introduce caching and CDN depending on access patterns. I would also define cache invalidation and handle hot keys.

### Scaling Writes

> If write volume exceeds a single database, I would consider partitioning or sharding using a stable key that distributes load evenly. For bursts, I would use queues, batching, rate limits, or load shedding depending on durability requirements.

---

## 13. Final Revision Summary

The most important idea is to start simple and scale only when needed.

```text
Start with simple synchronous APIs.
Add queues only for long-running work.
Add locks or transactions only when contention exists.
Add caching and replicas when reads bottleneck.
Add sharding when writes bottleneck.
Use blob storage for large files.
Use workflow engines for complex multi-step processes.
Use geospatial indexes only when location search is large-scale.
Use realtime protocols only when polling is insufficient.
```

A strong answer explains both pattern and trade-off. The interviewer checks whether each tool is justified, not whether you can list Redis, Kafka, WebSockets, or S3.
