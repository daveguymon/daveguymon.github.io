---
layout: default
title: "Pastebin Design"
permalink: /system-design/pastebin/
---
## System Design Scenario: Pastebin

**One-line outcome:** Enable users to store plain text over the Internet and generate unique URLs to access the uploaded data.

### 1) Problem & Requirements

**Functional:**  
1. Users should be able to upload or “paste” their data and get a unique URL to access it.
2. Users will only be able to upload text.
3. Data and links will expire after a specific timespan automatically; users should also be able to specify expiration time.
4. Users should optionally be able to pick a custom alias for their paste.

**Non-functional:**  
1. The system should be highly reliable, any data uploaded should not be lost.
2. The system should be highly available. This is required because if our service is down, users will not be able to access their Pastes.
3. Users should be able to access their Pastes in real-time with minimum latency.
4. Paste links should not be guessable (not predictable).

**Out of Scope:**  
1. Analytics
2. Public API

### 2) Approach Overview

**High-level architecture:**

![Pastebin Architecture Diagram]({{ '/assets/img/pastebin-diagram.png' | relative_url }})

**Core components:**
- User account creation/tenancy
- Document storage for pastes
- Default & custom link generation
- Rate limiting

**Key idea / tradeoff:** Heterogeneous storage with a SQL database for structured user account data and a NoSQL document storage database for pastes and low-latency reads. 

### 3) Data Model & Storage

**Entities & relationships:** User accounts (SQL) and Pastes (NoSQL/Document Storage). Paste documents will include a user id for authorship attribution.

**API:**

```
createPaste(userId, paste_content, paste_name, created_at, expires_at=None, custom_Url=None)
```

```
getPaste(pasteId)
```

**Storage choices:**
- User account data stored in a SQL database
- Paste data stored in a NoSQL document storage database
- NoSQL paste database has read-replicas to handle higher read:write ratio
- Key-value in-memory cache for serving pastes
- CDN caching of pastes since we are not collecting analytics

**Consistency strategy:** Read-Your-Writes consistency via synchronous replication

### 4) Request Flow

**Sequence:**  
0. User authenticates to access pastes
1. User uploads text blobs and metadata as paste content
2. Unique CSPRNG URL-safe strings are generated for each upload
3. Paste content + unique URL-safe strings stored in NoSQL primary db
4. Primary DB propogates writes to read-replicas
5. Read path serves reads efficiently using a write-through caching strategy
6. Worker routinely purges expired pastes

**Where latency is controlled:**
- CDN caching
- API gateway load balancing
- CQRS for optimizing separate read/write paths
- Use of NoSQL read-replicas for paste storage
- In-memory key-value cache on read path with Least Recently Used strategy

**Failure handling:** 
- Rate limiting of paste creation
- Uniqueness constraint in primary DB triggers key generation service retries
- Active-passive redundancy

### 5) Scalability & Reliability

**Scaling plan:**
- Horizontal scaling of NoSQL dbs via consistent hashing

**SLO/monitoring:**
- **Availability SLO:** 99.9% uptime (four 9s target for read paths)
- **Latency SLO:** P99 latency < 200ms for reads, P99 < 500ms for writes
- **Durability SLO:** Zero data loss; write replication to >= 2 replicas before acknowledgment
- **Key metrics to monitor:**
    - API endpoint response times (read/write separately)
    - Database replication lag
    - Cache hit ratio
    - Expired paste cleanup job success rate
    - Rate limiter reject rate
    - URL collision frequency
    - CDN cache hit ratio
    - Custom URL uniqueness constraint violations

### 6) Key Tradeoffs

- **Decision:** Polyglot persistence
    **Why:** User data has a predictable schema. So I opted for a relational database there. Pastes are subject to high read traffic. NoSQL enables lower latency for that.
    **Impact:** Need to store userId from users table in pastes documents.

- **Decision:** Rate limiting paste writes
    **Why:** Mitigating write abuse
    **Impact:** No single user should be able to overwhelm the Text Upload Service/Paste Write DB

- **Decision:** Expiration sweeper as a background job
    **Why:** Since pastes require expiration values, DB hygiene can be achieved by running the sweeper on a predictable schedule at lower traffic times
    **Impact:** Avoids unnecessary storage bloat and associated costs

### 7) Validation

**Back-of-the-envelope math:**
- Read/Write Ratio: 8:1
- 30,000 writes per day (0.5 writes per second)
- 240,000 reads per day (3 reads per second)
- Paste size: 512 KB (0.5 MB)
- 256 KB per second in
- 1.5 MB per second out

**Assumptions:**
- Every read downloads the entire 512 KB
- Default paste expiration value

### 8) What I’d Improve Next

**Next iteration:**
- Public/private paste exposure
- User/paste deletion strategy (e.g. cascading delete)

**Risks to revisit:**
- URL generation service: DIY vs Library
- Preloading a DB of available URL codes
- Object storage vs Document storage
