---
layout: default
title: "Instagram Design"
permalink: /system-design/instagram/
---

## System Design Review: Instagram

**Outcome:** A photo-sharing platform that allows users to upload images and discover recent content from the people they follow.

**Why it matters:** This design must support large-scale media delivery, rapid feed generation, and durable storage while maintaining a strong user experience under heavy load.

### 1) Problem & Requirements

**Functional requirements**
- Users can upload and view photos.
- Users can search photos by title.
- Users can follow other users.
- The system generates a personalized News Feed from recent content shared by followed accounts.

**Non-functional requirements**
- High availability.
- Low-latency feed generation, targeting roughly 200 ms p99.
- Eventual consistency is acceptable if a user occasionally sees content slightly later than expected.
- Strong durability for uploaded media; no photo should be lost.

**Out of scope**
- Photo tagging
- Tag-based search
- Comments
- User tagging in images
- Recommendations for who to follow

### 2) Approach Overview

**High-level architecture**

![Instagram Architecture Diagram]({{ '/assets/img/instagram-diagram.png' | relative_url }})

**Core components**
- User account service
- Image upload service
- Object storage for image bytes
- Relational storage for image metadata
- Search service for images and metadata
- Graph database for user relationships
- News Feed service
- Caching and CDN

**Design choice**
- Use object storage for raw image bytes and a relational database for metadata to improve scalability and latency.

### 3) Data Model & Storage

**API**

```text
createPost(userId, image, title, createdAt)
```

```text
followUser(followerId, followeeId)
```

```text
searchPostByTitle(userId, queryString)
```

```text
generateFeed(userId, created_at)
```

**Entities and relationships**
- Users
- Images
- Image metadata
- Relationships
- News Feeds

**Behavioral relationships**
- Users upload and view images in their feeds.
- Users follow other users.
- Users see recent images from followed accounts in their own feeds.

**Storage choices**
- User account data in a relational database
- Image bytes in object storage
- Image metadata in a relational database
- User relationships in a graph database

**Consistency strategy**
- Eventual consistency using semi-synchronous replication

### 4) Request Flow

**Sequence**
1. An authenticated user uploads an image and title.
2. The image upload service processes the image.
3. The image and metadata are published to a message bus.
4. Consumers store image bytes in object storage and metadata in a transactional database.
5. Users follow other users to see their images.
6. User feeds are generated and stored with timestamps.
7. Long-polling is used for feed updates.

**Where latency is controlled**
- Geographically distributed CDNs
- Multiple API gateways
- CQRS for separate read and write paths
- Service-based caching for reads
- Message queuing for image uploads
- Search indexing for images
- Database sharding

**Failure handling**
- Retries with exponential backoff
- Circuit breakers
- Dead-letter queues for poison inputs
- Replication and backups for services and databases

### 5) Scalability & Reliability

**Scaling plan**
- Cloud-based object storage for images scales efficiently.
- Consistent hashing sharding of image metadata databases (shard key: `imageId`)
- Message bus for image upload job processing

**SLO / monitoring**
1. Feed generation latency (p99): ≤ 200 ms for News Feed generation to end users, measured over a 28-day rolling window.
2. Upload availability: 99.95% availability for the image upload service over a rolling 28-day window.
3. Photo view availability: 99.99% availability for retrieving and displaying photos over a rolling 28-day window, including CDN and object storage.
4. Data durability: 99.9999% (six nines) durability of uploaded photos over one year; no user-uploaded photo should be lost due to system failure.

### 6) Key Tradeoffs

- **Decision:** Polyglot persistence for storing image data
  - **Why:** Object storage is better suited for image bytes, while relational storage is appropriate for structured metadata.
  - **Impact:** The flat architecture of object storage offers expansive and cost-effective scalability.

- **Decision:** Hybrid feed generation
  - **Why:** Some users have a much larger follower base than average users.
  - **Impact:** Using push and pull strategies for high-follower and regular users reduces read-path overhead.

- **Decision:** Geographically distributed content delivery networks
  - **Why:** Users want to share and view photos in near real time.
  - **Impact:** Caching content closer to users reduces request distance and latency.

### 7) Validation

**Back-of-the-envelope math**
- Total users: 500 million
- Daily active users (DAU): 50 million
- Daily uploads per user: 4
- Photo storage size: 200 KB
- Read-to-write ratio: 100:1
- Photo uploads per second: 2,000
- Photo views per second: 200,000
- Inbound bandwidth: 400 MB/second
- Peak inbound bandwidth: 1.2 GB/second
- Outbound bandwidth: 40 GB/second
- Peak outbound bandwidth: 160 GB/second
- Annual storage with replication: 50 PB
- Five-year storage with replication: 250 PB

**Assumptions**
- Users must authenticate to use the system.
- Some accounts may become hot spots and generate significantly more follows than average users.

### 8) What I’d Improve Next

**Next iteration**
- Image deletion
- Feed generation algorithm for engagement optimization
- Public/private photo viewing

**Risks to revisit**
- Cache placement
- Message consumer ordering (a URL from object storage is required for each image metadata record)
