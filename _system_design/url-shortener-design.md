---
layout: default
title: "URL Shortener Design"
permalink: /system-design/url-shortener/
---
## System Design Scenario: URL Shortener

**One-line outcome:** A URL shortening service like TinyURL that creates shorter aliases for long URLs. 

### 1) Problem & Requirements

**Functional:**  
- Given a long URL, the service returns a short link.
- Following a short link redirects the user to the original URL.

**Non-functional:**  
- High availability
- Low latency on redirects
- Short links must not be guessable

**Out of Scope:**  
- Link previews
- User accounts
- Malware scanning of destinations

### 2) Approach Overview

**High-level architecture:**

![URL Shortener Architecture Diagram]({{ '/assets/img/url-shortener-diagram.png' | relative_url }})

**Core components:**  
- Short code generation service
- Relational database with replication for persistent storage
- URL redirection service
- In-memory key-value store for caching redirects

**Key idea / tradeoff:** Opted for the increased complexity of Command Query Responsibility Segregation (CQRS) for lowering latency in a read-heavy system.

### 3) Data Model & Storage

**Entities & relationships:** Long URLs mapped to shorter aliases


**SQL Schema:**
```sql
CREATE TABLE mapped_links (
    short_code VARCHAR(10) PRIMARY KEY UNIQUE,
    original_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_created_at ON mapped_links(created_at);
```


**Storage choices:**
- In-memory queue of candidate codes per Key Generation Service instance
- Primary write database with multiple read replicas
- Key-value in-memory cache for serving redirects
- CDN redirect caching since we are not collecting analytics

**Consistency strategy:** Read-Your-Writes consistency via synchronous replication

### 4) Request Flow

**Sequence:**  
1. User-submits long url
2. Short Code Key Generation Service pops a pre-generated short code from the in-memory queue 
3. DB transaction containing original url and short code validated for short code uniqueness 
4. Transaction is committed or retried with next queued short code if uniquess constraint is violated
5. Leader DB propogates transaction to read replicas before returning success
6. Redirect requests pass through Redirection Service and in-memory cache using Read-Through caching strategy
7. Shortened URLs are returned to client with 301 status since we are not collecting analytics

**Where latency is controlled:**
- CQRS division of writes from reads
- Short code assignment served from preloaded in-memory queue
- Key-value store used for read-through caching
- Returning 301 instead 0f 302 HTTP status allows for CDN caching

**Failure handling:** Uniqueness constraint in primary DB triggers key generation service retries

### 5) Scalability & Reliability

**Scaling plan:** Horizontal scaling of DB via sharding with a consistent hashing of short code   

**SLO/monitoring:**
Over a rolling 28-day window, aim for:
- Create availability: 99.9%
- Create p99 latency: ≤ 200 ms
- Redirect availability: 99.95%
- Redirect p99 latency: ≤ 50 ms

### 6) Key Tradeoffs

- **Decision:** Command Query Responsibility Segregation pattern  
    **Why:** 100:1 read-write ratio  
    **Impact:** Added architectural complexity to optimize write and read performance

- **Decision:** Synchronous leader-follower RDBMS replication  
    **Why:** Data persistence + strong consistency  
    **Impact:** Mitigates data loss at the expense of some latency

- **Decision:** Read-Through caching strategy  
    **Why:** Alleviate DB server load  
    **Impact:** Places responsibility for cache-miss resolution on the in-memory key-value store rather than the Redirection Service

### 7) Validation

**Back-of-the-envelope math:**
- Read/Write Ratio: 100:1
- 20M writes per day (230 writes per second)
- 200M reads per day (2,300 reads per second)
- Link size: 500 bytes
- 11.5 KB per second in
- 1.15 MB per second out

**Assumptions:**  
- Steady usage patterns (no peak/surge times)
- Medium-sized product compared to industry

### 8) What I’d Improve Next

**Next iteration:**  
- Rate limiting
- DB level load balancing

**Risks to revisit:**  
- Data storage requirements if short links don't expire
- Rate limiting write requests
