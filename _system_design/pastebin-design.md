---
layout: default
title: "Pastebin Design"
permalink: /system-design/pastebin/
---

## System Design Review: Pastebin

**Outcome:** A simple paste-sharing service that lets users store text online and retrieve it through a unique URL.

**Why it matters:** The design must balance durability and availability while keeping read latency low for a highly access-heavy workload.

### 1) Problem & Requirements

**Functional requirements**
- Users can upload or “paste” content and receive a unique URL to access it.
- Users can upload only text.
- Data and links expire automatically, and users can specify an expiration time.
- Users may optionally choose a custom alias for their paste.

**Non-functional requirements**
- High reliability; uploaded data should not be lost.
- High availability; if the service is down, users cannot access their pastes.
- Low-latency reads for real-time access.
- Paste links should be non-guessable.

**Out of scope**
- Analytics
- Public API

### 2) Approach Overview

**High-level architecture**

![Pastebin Architecture Diagram]({{ '/assets/img/pastebin-diagram.png' | relative_url }})

**Core components**
- User account creation and tenancy
- Document storage for pastes
- Default and custom link generation
- Rate limiting

**Design choice**
- Use heterogeneous storage: a relational database for structured user account data and a NoSQL document store for pastes and low-latency reads.

### 3) Data Model & Storage

**Entities and relationships**
- User accounts stored in SQL
- Pastes stored in a NoSQL document store
- Each paste document includes a user ID for authorship attribution

**API**

```text
createPaste(userId, pasteContent, pasteName, createdAt, expiresAt = null, customUrl = null)
```

```text
getPaste(pasteId)
```

**Storage choices**
- User account data stored in a SQL database
- Paste data stored in a NoSQL document database
- NoSQL read replicas to handle a high read-to-write ratio
- In-memory key-value cache for serving pastes
- CDN caching for public content since analytics are not collected

**Consistency strategy**
- Read-your-writes consistency through synchronous replication

### 4) Request Flow

**Sequence**
1. A user authenticates to access pastes.
2. The user uploads text and metadata as paste content.
3. The system generates a unique, URL-safe string for the paste.
4. The paste and its URL are stored in the primary NoSQL database.
5. The primary database propagates writes to read replicas.
6. Reads are served efficiently through a write-through caching strategy.
7. A background worker removes expired pastes.

**Where latency is controlled**
- CDN caching
- API gateway load balancing
- CQRS for separate read and write paths
- NoSQL read replicas for paste storage
- In-memory key-value cache on the read path using an LRU strategy

**Failure handling**
- Rate limiting of paste creation
- A uniqueness constraint in the primary database triggers retries in the key generation service
- Active-passive redundancy

### 5) Scalability & Reliability

**Scaling plan**
- Horizontal scaling of NoSQL databases via consistent hashing

**SLO / monitoring**
- **Availability SLO:** 99.9% uptime for read paths
- **Latency SLO:** p99 latency < 200 ms for reads and < 500 ms for writes
- **Durability SLO:** Zero data loss; write replication to at least two replicas before acknowledgment
- **Key metrics to monitor**
  - API endpoint response times for reads and writes
  - Database replication lag
  - Cache hit ratio
  - Expired paste cleanup job success rate
  - Rate limiter reject rate
  - URL collision frequency
  - CDN cache hit ratio
  - Custom URL uniqueness constraint violations

### 6) Key Tradeoffs

- **Decision:** Polyglot persistence
  - **Why:** User data has a predictable schema, while pastes require low-latency reads at scale.
  - **Impact:** The system needs to store a user ID from the users table inside paste documents.

- **Decision:** Rate limiting paste writes
  - **Why:** This mitigates write abuse.
  - **Impact:** No single user should overwhelm the text upload service or paste write database.

- **Decision:** Expiration sweeper as a background job
  - **Why:** Since pastes require expiration values, database hygiene can be handled on a predictable schedule during quieter traffic periods.
  - **Impact:** This avoids unnecessary storage bloat and reduces long-term cost.

### 7) Validation

**Back-of-the-envelope math**
- Read-to-write ratio: 8:1
- 30,000 writes per day (0.5 writes per second)
- 240,000 reads per day (3 reads per second)
- Paste size: 512 KB (0.5 MB)
- 256 KB per second inbound
- 1.5 MB per second outbound

**Assumptions**
- Every read downloads the full 512 KB.
- The default paste expiration value is used.

### 8) What I’d Improve Next

**Next iteration**
- Public/private paste exposure
- User and paste deletion strategy, such as cascading delete

**Risks to revisit**
- Whether to build a URL generation service in-house or reuse a library
- Preloading a database of available URL codes
- Whether object storage or document storage is the better long-term fit
