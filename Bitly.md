# Bitly / URL Shortener

## Problem

Design a URL shortening service like Bitly.

The system converts long URLs into short URLs and redirects users from the short URL to the original URL.

Example:

```text
Long URL:  https://www.example.com/some/very/long/url
Short URL: https://short.ly/abc123
```

---

## Functional Requirements

Core requirements:

1. Users can submit a long URL and receive a short URL.
2. Users can optionally specify a custom alias.
3. Users can optionally specify an expiration date.
4. Users can access the original URL using the short URL.

Out of scope:

- User authentication and account management.
- Click analytics.
- Geographic analytics.
- Spam detection and malicious URL filtering.

---

## Non-Functional Requirements

Core requirements:

1. Short codes must be unique.
2. Redirect latency should be low, for example under 100 ms.
3. System should be highly available, around 99.99%.
4. System should scale to around 1B shortened URLs and 100M DAU (Daily Active Users).
5. Reads are much heavier than writes.

Read/write ratio:

```text
1000 redirects : 1 short URL creation
```

Design implication:

- Optimize heavily for fast reads and redirects.
- Use caching for hot short codes.
- Keep writes simple and reliable.

Availability vs consistency:

- Redirect availability matters more than perfect consistency.
- For URL creation, uniqueness must still be guaranteed.

---

## Core Entities

```text
User
OriginalURL
ShortURL
```

More useful data model:

```text
URLMapping
- short_code
- long_url
- custom_alias
- created_at
- expires_at
- creator_user_id
```

Important constraints:

- `short_code` must be unique.
- `custom_alias` must be unique if present.
- Expired URLs should not redirect.

---

## APIs

Use REST (Representational State Transfer) APIs.

### Create Short URL

```http
POST /urls
```

Request:

```json
{
  "long_url": "https://www.example.com/some/very/long/url",
  "custom_alias": "optional_custom_alias",
  "expiration_date": "optional_expiration_date"
}
```

Response:

```json
{
  "short_url": "https://short.ly/abc123"
}
```

Why `POST`:

- Creates a new URL mapping.
- Stores short code to long URL mapping in the database.

### Redirect

```http
GET /{short_code}
```

Response:

```http
HTTP/1.1 302 Found
Location: https://www.original-long-url.com
```

Why `GET`:

- Reads the existing mapping.
- Redirects the browser to the original URL.

---

## Redirect Status Code

Use `302 Found` for most URL shorteners.

Why `302`:

- Temporary redirect.
- Browser is less likely to permanently cache it.
- Server keeps control over expiration, deletion, updates, and analytics.

Avoid `301 Moved Permanently` unless the mapping should never change.

Problem with `301`:

- Browsers and intermediaries may cache the redirect.
- Future requests may bypass the shortener service.
- Expiration and analytics become harder.

Expired URL:

```http
HTTP/1.1 410 Gone
```

---

## High-Level Design

### Create Short URL Flow

```text
Client
  ↓ POST /urls
Load Balancer
  ↓
Write Service
  ↓
Short Code Generator
  ↓
Database
  ↓
Return short URL
```

Steps:

1. Client submits long URL, optional custom alias, and optional expiration date.
2. Write Service validates the long URL.
3. If custom alias exists, validate uniqueness.
4. If no custom alias exists, generate a unique short code.
5. Store mapping in database.
6. Return short URL to the client.

### Redirect Flow

```text
Browser
  ↓ GET /abc123
Load Balancer
  ↓
Read Service
  ↓
Cache
  ↓ cache miss
Database
  ↓
302 Redirect
```

Steps:

1. Browser sends request to `short.ly/{short_code}`.
2. Read Service checks cache.
3. On cache hit, return redirect.
4. On cache miss, query database by `short_code`.
5. If code does not exist, return `404 Not Found`.
6. If code is expired, return `410 Gone`.
7. If valid, cache the mapping and return `302 Found`.

---

## Database Choice

PostgreSQL is a good default.

Why PostgreSQL works:

- URL creation write volume is relatively low.
- 1B rows is large but manageable with indexing, storage planning, and partitioning if needed.
- Strong uniqueness constraints are useful for `short_code`.
- Simple lookup by primary key is efficient.

Table:

```sql
CREATE TABLE url_mappings (
  short_code TEXT PRIMARY KEY,
  long_url TEXT NOT NULL,
  custom_alias TEXT UNIQUE,
  created_at TIMESTAMP NOT NULL,
  expires_at TIMESTAMP,
  creator_user_id TEXT
);
```

Important index:

```sql
PRIMARY KEY (short_code)
```

This avoids a full table scan during redirects.

---

## Capacity Estimate

Assume:

```text
1B URL mappings
~500 bytes per row after metadata overhead
```

Storage:

```text
1B * 500 bytes = 500 GB
```

This is reasonable for modern databases.

Write volume example:

```text
100K new URLs/day
≈ 1 write/second
```

Reads dominate the system, so the harder scaling problem is redirect traffic, not URL creation.

---

## Deep Dive 1: Short Code Generation

Requirements:

- Unique.
- Short.
- Fast to generate.
- Low collision risk.

### Option 1: Hash Long URL

```text
hash(long_url) → short_code
```

Pros:

- Simple.
- Deterministic for the same long URL.

Cons:

- Collisions are possible.
- Same long URL maps to same short code unless salted.
- Harder to support multiple short links for the same long URL.

Collision handling:

- Check database before insert.
- Add random salt if collision occurs.
- Retry with a new code.

### Option 2: Random Code

```text
random_base62(7 or 8 chars)
```

Pros:

- Simple.
- Non-predictable.
- No central counter required.

Cons:

- Collision possible.
- Must check database and retry.

### Option 3: Counter + Base62

```text
counter = 125
base62(counter) = cb
```

Base62 alphabet:

```text
a-z, A-Z, 0-9
```

Pros:

- Guaranteed uniqueness if counter is unique.
- Short codes are compact.
- No collision retry needed.

Cons:

- Counter becomes shared state.
- Predictable URLs can be a security concern.
- Multi-region writes need coordination.

Recommended interview choice:

```text
Use a centralized atomic counter and encode it with Base62.
Use a database UNIQUE constraint as the final safety net.
```

---

## Deep Dive 2: Fast Redirects

Problem:

```text
Redirects are read-heavy and latency-sensitive.
```

Use cache-aside:

```text
Read Service
  ↓
Redis Cache
  ↓ cache miss
Database
```

Cache key:

```text
short_code -> long_url, expires_at
```

Cache behavior:

1. Check Redis first.
2. If hit and not expired, return `302`.
3. If miss, query database.
4. Store result in Redis.
5. Set TTL (Time To Live) to match or be shorter than URL expiration.

Why TTL matters:

- Prevents expired links from staying cached.
- Reduces manual cache invalidation.

Hot links:

- Popular short URLs may receive massive traffic.
- Cache absorbs most reads.
- CDN (Content Delivery Network) or edge caching can help only if redirect semantics and analytics requirements allow it.

---

## Deep Dive 3: Scaling

### Separate Reads and Writes

Reads and writes have different traffic patterns.

```text
Write Service → create short URLs
Read Service  → handle redirects
```

Benefits:

- Scale read service independently.
- Keep redirect path optimized.
- Reduce impact of write traffic on redirects.

### Horizontally Scale Services

```text
Load Balancer
  ↓
Read Service Instances
  ↓
Cache / Database
```

Read Service can be scaled by adding more instances.

Write Service can also be scaled, but short code generation needs global uniqueness.

---

## Scaling the Counter

Problem:

```text
Multiple Write Service instances need unique counter values.
```

Use Redis atomic increment:

```text
INCR url_counter
```

Flow:

1. Write Service asks Redis for next counter value.
2. Redis atomically increments counter.
3. Write Service encodes value using Base62.
4. Write Service stores mapping in database.

### Counter Batching

Instead of calling Redis for every URL:

```text
Instance A gets 1-1000
Instance B gets 1001-2000
Instance C gets 2001-3000
```

Benefits:

- Fewer Redis calls.
- Higher write throughput.
- Uniqueness still preserved.

Trade-off:

- Some unused values may be lost if a service crashes.
- This is acceptable because uniqueness matters more than continuity.

### Redis Availability

Use:

- Redis Sentinel.
- Redis Cluster.
- Automatic failover.

If Redis loses a few latest counter values before replication, database uniqueness constraints still protect against duplicate short codes.

---

## Multi-Region Design

For multi-region writes, avoid a single global counter.

Use disjoint counter ranges:

```text
Region A: 0 to 1B
Region B: 1B to 2B
Region C: 2B to 3B
```

Benefits:

- Local writes do not need cross-region coordination.
- Each region can generate unique short codes independently.

Reads:

- Serve redirects from nearest region.
- Use distributed caches.
- Replicate database data across regions.

Trade-off:

- Replication lag may cause a newly created short URL to be unavailable briefly in another region.
- Usually acceptable if reads in the creation region are immediately available.

---

## Expiration Handling

On every redirect:

```text
if expires_at < now:
    return 410 Gone
```

Cleanup options:

1. Keep expired rows and check `expires_at`.
2. Periodically delete expired rows with a background job.

Cache rule:

```text
cache_ttl <= expires_at - now
```

This prevents stale redirects after expiration.

---

## Custom Alias Handling

For custom aliases:

1. Validate allowed characters.
2. Check uniqueness.
3. Reserve alias with database constraint.
4. Return error if alias already exists.

Collision prevention:

- Generated codes and custom aliases can share one namespace with a unique constraint.
- Or use separate namespaces.
- Or reserve a prefix for generated codes that custom aliases cannot use.

Example:

```text
Generated: /g/abc123
Custom:    /my-brand-link
```

---

## Failure Handling

Database failure:

- Redirects may still work for cached hot links.
- Cache misses may fail.
- Use read replicas and failover.

Cache failure:

- Fall back to database.
- Latency increases.
- Database load increases.

Redis counter failure:

- URL creation may be unavailable.
- Redirects should still work.
- Use Redis failover or counter batching.

Write failure after counter increment:

- Counter value may be skipped.
- Acceptable because uniqueness matters more than gap-free codes.

---

## Security Notes

Predictable counter-based URLs can be enumerated.

Mitigations:

- Use random codes instead of counters.
- Add salting or obfuscation.
- Use longer codes.
- Rate-limit suspicious scanning.
- Add malicious URL detection if in scope.

For the basic interview problem, mention this as a trade-off, not a core requirement.

---

## Final Architecture

```text
Clients / Browsers
        ↓
Load Balancer
        ↓
 ┌───────────────┬───────────────┐
 │ Write Service │ Read Service  │
 └───────↓───────┴───────↓───────┘
   Redis Counter     Redis Cache
        ↓                ↓
        └────── Database ┘
```

Write path:

```text
POST /urls
→ validate URL
→ generate or validate short code
→ store mapping
→ return short URL
```

Read path:

```text
GET /{short_code}
→ cache lookup
→ database lookup on miss
→ expiration check
→ 302 redirect
```

---

## Interview Deep Dive Choices

Best deep dives:

1. Unique short code generation.
2. Fast redirects with caching and indexes.
3. Scaling read-heavy traffic.
4. Counter coordination across write services.
5. Expiration and custom alias edge cases.

---

## Level Expectations

### Mid-Level

Expected:

- Working URL creation and redirect design.
- Basic short code generation approach.
- Database mapping from short code to long URL.
- Understand why redirects use `302`.
- Mention index on `short_code`.
- Add cache when prompted.

### Senior

Expected:

- Proactively identify uniqueness, latency, and scaling as key challenges.
- Compare hashing, random codes, and counter-based codes.
- Explain cache-aside and TTL for expiration.
- Justify database choice.
- Separate read and write services.
- Use Redis or similar coordination for global counter.

### Staff+

Expected:

- Start from read-heavy workload.
- Discuss multi-region reads and writes.
- Allocate disjoint counter ranges per region.
- Consider Redis failover and counter batching.
- Discuss predictable URL security risk.
- Explain custom alias collision handling.
- Address cleanup, operations, and future analytics.

---

## Final Summary

Bitly is a read-heavy URL mapping system.

Core idea:

```text
short_code -> long_url
```

Important design choices:

- Use `302 Found` for redirects.
- Store mappings in a database with unique `short_code`.
- Cache hot mappings for fast redirects.
- Generate unique codes using Base62 over a counter or random code with collision checks.
- Separate read and write services as traffic grows.
- Use TTL and expiration checks to avoid stale redirects.
