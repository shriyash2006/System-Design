# Instagram System Design Specification

This document details the system design, capacity planning, and architectural topology for Instagram, a social network optimized to handle high-throughput media ingestion and low-latency timeline feed delivery.

---

## 1. Core Requirements & Scale Estimations

### Functional Requirements
* **Media Upload**: Users can upload high-quality media payloads (both images and video files).
* **Social Graph**: Users can follow and unfollow other accounts.
* **Timeline Generation**: The system generates a reverse-chronological home timeline/feed based on follow lists.
* **Discovery Search**: Users can search across post text captions and hashtags.

### Non-Functional Requirements
* **High Availability**: Prefer eventual consistency over strict atomic consistency across read operations.
* **Maximum Durability**: Uploaded media must be safely replicated and stored without data loss.
* **Ultra-Low Latency**: Rendering home timelines should take $< 200\text{ ms}$.

### Scale Capacity Calculations

We base our math on **500 Million Daily Active Users (DAU)**.

#### Ingestion & Write Volume
* Average upload cadence: 1 post per user every 5 days ($0.2\text{ posts/day}$).
* **Total Daily Posts**:
  $$\text{Daily Posts} = 500,000,000 \times 0.2 = 100,000,000\text{ posts/day}$$
* **Write Query QPS**:
  $$\text{Write QPS} = \frac{100,000,000}{86,400\text{ seconds}} \approx 1,157\text{ writes/second}$$

#### Read Volume
* Assuming a standard social media **100:1 read-to-write split**:
  $$\text{Read QPS} = 1,157 \times 100 = 115,700\text{ reads/second}$$

#### Storage Footprint (Raw Files)
* Average post size after optimization/compression: $1\text{ MB}$.
* **Daily Storage Ingestion**:
  $$\text{Daily Ingestion} = 100,000,000 \times 1\text{ MB} = 100\text{ TB/day} \quad (\approx 95\text{ TB/day with deduplication})$$
* **Annual Storage Requirement**:
  $$\text{Annual Storage} \approx 95\text{ TB/day} \times 365 \approx 34.6\text{ PB/year}$$

---

## 2. System Architecture Topology

The diagram below maps the complete media ingestion (Write Path) and feed compilation (Read Path) flows.

![High-Level System Architecture](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Instagram%20App:%20System%20Design/architecture-diagram.png)

### Mindmap Breakdown

For a conceptual overview of the component dependencies and strategies, see the Playbook Mindmap:

![System Design Mindmap](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Instagram%20App:%20System%20Design/mindmap.jpg)

---

## 3. Data Schema & Relationships

The system uses a relational PostgreSQL database to ensure strong metadata stability, sharded horizontally using `user_id` hashing.

![Database ER Diagram](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Instagram%20App:%20System%20Design/database-schema.png)

### Primary Entities
* **Users**: Stores account information.
* **Followers**: A join table mapping follower-followee pairs to represent the social graph.
* **Media**: Stores raw asset metadata and S3 object storage location references.
* **Posts**: Holds post captions, hashtags, and author details.
* **Post_Media (Intersection)**: Decouples posts from media items, allowing single posts to map to multiple media items (carousel posts).

---

## 4. Playbook Modules

For deep technical details on specific aspects of the design, navigate to the following documents:

1. [API Design Matrix](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Instagram%20App:%20System%20Design/api-design.md) - Specifying REST contracts, multipart chunked uploads, and cursor-based pagination.
2. [Scaling & Ingestion Strategy](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Instagram%20App:%20System%20Design/scaling-notes.md) - Covering S3 chunking, CDN cache invalidation, asynchronous Kafka consumers, and hybrid push/pull fan-out mechanics.
3. [Bottleneck & Tradeoff Analysis](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Instagram%20App:%20System%20Design/tradeoffs.md) - Deep dive on Push vs Pull latency, relational sharding vs NoSQL, and infinite comment structures (Adjacency Lists vs Nested Sets).
