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

**Key idea / tradeoff:**

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
-

### 4) Request Flow

**Sequence:**  
1. 

**Where latency is controlled:**

**Failure handling:**

### 5) Scalability & Reliability

**Scaling plan:**

**SLO/monitoring:**


### 6) Key Tradeoffs

- **Decision:** 
    **Why:** 
    **Impact:**

- **Decision:**
    **Why:** 
    **Impact:** 

- **Decision:**
    **Why:**  
    **Impact:** 

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
- 

### 8) What I’d Improve Next

**Next iteration:**  
- 

**Risks to revisit:**  
- 
