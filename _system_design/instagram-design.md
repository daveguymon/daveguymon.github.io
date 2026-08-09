---
layout: default
title: "Instagram Design"
permalink: /system-design/instagram/
---
## System Design Scenario: Instagram

**One-line outcome:** A photo-sharing service allowing users to upload and share photos with other users who follow them.

### 1) Problem & Requirements

**Functional:**  
1. Users should be able to upload/view photos.
2. Users can perform searches based on photo titles.
3. Users can follow other users.
4. The system should generate and display a user's News Feed consisting of top photos from all the people the user follows.

**Non-functional:**  
1. Our service needs to be highly available.
2. The acceptable latency of the system is 200ms for News Feed generation.
3. Consistency can take a hit (in the interest of availability) if a user doesn’t see a photo for a while; it should be fine.
4. The system should be highly reliable; any uploaded photo or video should never be lost.

**Out of Scope:**  
1. Adding tags to photos
2. Searching photos on tags
3. Commenting on photos
4. Tagging users in photos
5. Who to follow recommendations

### 2) Approach Overview

**High-level architecture:**

![Instagram Architecture Diagram]({{ '/assets/img/instagram-diagram.png' | relative_url }})

**Core components:**  
- User account service
- Image upload service
- Object storage for image bytes
- Relational storage for image metadata
- Image + metadata search service
- Graph DB for user relationships
- Newsfeed service
- Caching + CDN

**Key idea / tradeoff:** Combining relational storage for image metadata with object storage for image byte data for improved latency.

### 3) Data Model & Storage

**Entities & relationships:**
- Users
- Images
- Image metadata
- Relationships
- Newsfeeds

- Users upload and view images on their newsfeeds
- Users follow other users
- Users see other users' top images on their own newsfeeds

**Storage choices:**
- User account data in a relational DB
- Image byte data in object storage
- Image metadata in a relational DB
- User relationships in a graph DB

**Consistency strategy:**
- Eventual consistency using semi-synchronous replication

### 4) Request Flow

**Sequence:**  
1. Authenticated user uploads an image and title
2. The image upload service processes the image
3. The image and metadata are published to a message bus
4. Consumers store image byte data to an object store followed by storing image metadata to a transactional db
5. Users follow other users in order to see their images
6. User feeds are generated and stored with timestamps
7. Long-polling is used for feed updates

**Where latency is controlled:**
- Geographically distributed CDNs
- Multiple API gateways
- CQRS for read and write paths
- Service-based caching for reads
- Message queing for image uploads
- Search indexing for images
- Database sharding

**Failure handling:**
- Retries with exponential backoff
- Circuit breakers
- Dead letter queues for poison inputs
- Backup/replication of all services and databases

### 5) Scalability & Reliability

**Scaling plan:**
- Cloud-based object storage for images scales efficiently
- Consistent hashing sharding of image metadata DBs (shard key: `imageId`)
- Message bus for image upload job processing

**SLO/monitoring:**
1. Feed Generation Latency (p99): ≤ 200 ms for News Feed generation to end users, measured over a 28-day rolling window.
2. Upload Availability: 99.95% availability for the image upload service over a rolling 28-day window.
3. Photo View Availability: 99.99% availability for retrieving and displaying photos (reads) over a rolling 28-day window, including CDN and object storage.
4. Data Durability: 99.9999% (six nines) durability of uploaded photos over 1 year; no user-uploaded photo should be lost due to system failure.

### 6) Key Tradeoffs

- **Decision:** Polyglot persistence for storing image data
    **Why:** Object storage better suited for image byte data, while relational storage appropriate for image metadata.
    **Impact:** Object storage's flat architecture offers expansive and cost-effective scalability

- **Decision:** Dual approach to feed generation
    **Why:** Some users may generate a massive amount of followers compared to regular users
    **Impact:** Utilizing push (fanout-on-write) and pull (fanout-on-read) methods for high-follower and regular users avoids excessive computational overhead on the read-path

- **Decision:** Reliance on geographically-distributed Content Delivery Networks
    **Why:** Users want to share and view photos in near-real time
    **Impact:** Caching content closer to users decreases the distance and time required by requests.

### 7) Validation

**Back-of-the-envelope math:**
- Total Users: 500 Million
- Daily Active Users (DAU): 50 Million
- Daily Uploads per User: 4
- Photo storage size: 200 KB
- Read-to-Write Ratio: 100:1
- Photo uploads per second: 2000
- Photo views per second: 200,000
- Inbound bandwidth: 400 MB/second
- Peak inbound bandwith: 1.2 GB/second
- Outbound bandwidth: 40 GB/second
- Peak outbound bandwidth: 160 GB/second
- Annual Storage (with Replication): 50 PB
- 5-year Storage (with Replication): 250 PB

**Assumptions:**  
- User must authenticate to use the system
- There may be hot spot accounts that will generate significant follows compared to regular users

### 8) What I’d Improve Next

**Next iteration:**  
- Image deletion
- Feed generation algorithm for engagement optimization
- Public/private photo viewing

**Risks to revisit:**  
- Location of caches
- Message consumer ordering (URL from object store required for image metadata record)
