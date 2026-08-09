---
layout: default
title: "URL Shortener Design"
permalink: /system-design/url-shortener/
---

## System Design Review: URL Shortener

**Outcome:** A URL-shortening service that creates compact aliases for long links and redirects users to the original destination.

**Why it matters:** The system must stay highly available and extremely fast for redirect traffic while preserving data integrity for newly created links.

### 1) Problem & Requirements

**Functional requirements**
- Given a long URL, the service returns a short link.
- Following a short link redirects the user to the original URL.

**Non-functional requirements**
- High availability
- Low latency on redirects
- Short links must not be guessable

**Out of scope**
- Link previews
- User accounts
- Malware scanning of destinations

### 2) Approach Overview

**High-level architecture**

![URL Shortener Architecture Diagram]({{ '/assets/img/url-shortener-diagram.png' | relative_url }})

**Core components**
- Short code generation service
- Relational database with replication for persistent storage
- URL redirection service
- In-memory key-value store for caching redirects

**Design choice**
- Use Command Query Responsibility Segregation (CQRS) to lower latency in a read-heavy system.

### 3) Data Model & Storage

**Entities and relationships**
- Long URLs mapped to shorter aliases

**SQL schema**
```sql
CREATE TABLE mapped_links (
    short_code VARCHAR(10) PRIMARY KEY UNIQUE,
    original_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_created_at ON mapped_links(created_at);
```

**Storage choices**
- In-memory queue of candidate codes per key generation service instance
- Primary write database with multiple read replicas
- Key-value in-memory cache for serving redirects
- CDN redirect caching since analytics are not collected

**Consistency strategy**
- Read-your-writes consistency through synchronous replication

### 4) Request Flow

**Sequence**
1. A user submits a long URL.
2. The short code generation service pops a pre-generated short code from the in-memory queue.
3. A database transaction containing the original URL and short code is validated for uniqueness.
4. The transaction is committed or retried with the next queued short code if a uniqueness constraint is violated.
5. The leader database propagates the transaction to read replicas before returning success.
6. Redirect requests pass through the redirection service and the in-memory cache using a read-through strategy.
7. Shortened URLs are returned to the client with a 301 status since analytics are not collected.

**Where latency is controlled**
- CQRS separation of writes from reads
- Short code assignment served from a preloaded in-memory queue
- Key-value store used for read-through caching
- Returning 301 instead of 302 allows for CDN caching

**Failure handling**
- A uniqueness constraint in the primary database triggers retries in the key generation service.

### 5) Scalability & Reliability

**Scaling plan**
- Horizontal scaling of the database via sharding with consistent hashing of the short code

**SLO / monitoring**
- Create availability: 99.9%
- Create p99 latency: ≤ 200 ms
- Redirect availability: 99.95%
- Redirect p99 latency: ≤ 50 ms

### 6) Key Tradeoffs

- **Decision:** Command Query Responsibility Segregation pattern
  - **Why:** The system is expected to have a 100:1 read-to-write ratio.
  - **Impact:** The design adds architectural complexity to optimize write and read performance.

- **Decision:** Synchronous leader-follower RDBMS replication
  - **Why:** This improves data persistence and consistency.
  - **Impact:** It reduces the chance of data loss at the cost of some latency.

- **Decision:** Read-through caching strategy
  - **Why:** This alleviates database load.
  - **Impact:** Cache-miss resolution is handled by the in-memory key-value store rather than the redirection service.

### 7) Validation

**Back-of-the-envelope math**
- Read-to-write ratio: 100:1
- 20 million writes per day (230 writes per second)
- 200 million reads per day (2,300 reads per second)
- Link size: 500 bytes
- 11.5 KB per second inbound
- 1.15 MB per second outbound

**Assumptions**
- Usage patterns remain steady with no major surge periods
- The product is medium-sized compared with industry peers

### 8) What I’d Improve Next

**Next iteration**
- Rate limiting
- Database-level load balancing

**Risks to revisit**
- Storage requirements if short links do not expire
- Rate limiting write requests
