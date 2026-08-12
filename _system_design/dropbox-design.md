---
layout: default
title: "Dropbox Design"
permalink: /system-design/dropbox/
---

## System Design Review: Dropbox

**Outcome:** A cloud file storage system that allows users to upload, sync, and download files/photos from any device.

**Why it matters:** The system must have anywhere/anytime data availability with 100% data reliability and durability so users never have to worry about accessing their files or running out of storage space.

### 1) Problem & Requirements

**Functional requirements**
- Users should be able to upload and download their files/photos from any device.
- Users should be able to share files or folders with other users.
- Our service should support automatic synchronization between devices, i.e., after updating a file on one device, it should get synchronized on all devices.
- Our system should support offline editing. Users should be able to add/delete/modify files while offline, and as soon as they come online, all their changes should be synced to the remote servers and other online devices.

**Non-functional requirements**
- The system should support storing large files up to a GB.
- ACID-ity is required. Atomicity, Consistency, Isolation and Durability of all file operations should be guaranteed.

**Out of scope**
- Multiple contributors in real-time

### 2) Approach Overview

**High-level architecture**

![Dropbox Architecture Diagram]({{ '/assets/img/dropbox-diagram.png' | relative_url }})

**Core components**
- Upload service
- Data segmentation worker
- Processing queue + workers
- Metadata SQL database
- Block/Cloud storage
- Block retrieval worker + assembly service
- Listening service + sync worker
- Notification service
- File query service
- Cache + CDN

**Design choice**
- Use block storage with data segmentation and assembly for management of diff-based blobs while utilizing a SQL-based metadata solution for mapping blocks to files along with versioning

### 3) Data Model & Storage

**API**

```text
uploadFile(fileData, fileId, userId)
```

```text
getFileMetadata(fileId, userId)
```

```text
getChanges(userId, sinceToken)
```

```text
downloadFile(fileId, userId)
```

```text
updateFile(fileData, fileId, userId)
```

**Entities and relationships**
- User
- Devices
- Folder
- Files
- Blobs/blocks


**Behavioral relationships**
- Users access file storage from many devices
- Users upload files to cloud folder
- Files are segmented into blobs for block storage


**Storage choices**
- Block/cloud storage for file data
- Relational database for file metadata 
- In memory caching for reading assembled file data
- CDNs for quick retrieval

**Consistency strategy**
- Eventual consistency using semi-synchronous replication

### 4) Request Flow

**Sequence**


**Where latency is controlled**


**Failure handling**


### 5) Scalability & Reliability

**Scaling plan**


**SLO / monitoring**


### 6) Key Tradeoffs



### 7) Validation

**Back-of-the-envelope math**
- Total users: 500 million
- Daily active users (DAU): 100 million
- Daily uploads per user: 3
- File storage size: 100 KB
- Annual storage with replication: 30 PB
- Five-year storage with replication: 150 PB
- Read-to-write ratio: 5:1
- File uploads per second: 3,000
- Inbound (Upload) bandwidth: 300 MB/second
- Outbound (View) bandwidth: 1,500 MB/second
- Downloads per second: 1,000
- Download bandwidth: 100 MB/second
- Connections per minute: 1 million


**Assumptions**
- Average file upload speed of 20 MB/second
- Average file download speed of 100 MB/second

### 8) What I’d Improve Next

**Next iteration**


**Risks to revisit**

