# Bottlenecks & Tradeoffs: Twitter / Newsfeed

This document analyzes the design trade-offs made in the Twitter / Newsfeed architecture to balance extreme read/write performance.

---

## 1. Fan-out Strategy: Push vs. Pull vs. Hybrid

### Option A: Pure Push (Fan-out on Write)
* **Pros**:
  * Reading the timeline is fast ($\mathcal{O}(1)$). The timeline is pre-generated in Redis.
* **Cons**:
  * **The Celebrity Problem**: Writing a tweet for a celebrity with 80M followers triggers 80M database/cache writes, causing write replication queues to block.

### Option B: Pure Pull (Fan-out on Read)
* **Pros**:
  * Write path is extremely fast and simple. A tweet is written to a single location (the author's user timeline).
* **Cons**:
  * **High Read Latency**: Every time a user requests their home timeline, the system must fetch the followings list, pull tweets from all followed users, and merge/sort them in real-time ($\mathcal{O}(N \log N)$), causing major database bottlenecks.

### Option C: Hybrid Model (Selected)
* **Pros**:
  * Combines push model for regular users and pull model for celebrity/VIP users. Keeps write pipelines responsive while keeping read latency low for 99.9% of users.
* **Cons**:
  * **Increased Application Complexity**: The application layer must track user follow counts, handle dual-execution paths, and merge feeds at read-time.

---

## 2. Social Graph Storage: Relational (SQL) vs. Graph Database

```
            [ Social Graph Storage Tier Choices ]

    ┌───────────────────────────────────┐     ┌───────────────────────────────────┐
    │       Sharded PostgreSQL          │     │          Neo4j (Graph DB)         │
    ├───────────────────────────────────┤     ├───────────────────────────────────┤
    │  - Linear, predictable scale      │     │  - Fast multi-hop traversals      │
    │  - Excellent partition sharding   │     │  - Complex write scaling          │
    │  - Simple join-table lookups      │     │  - High storage overhead          │
    └───────────────────────────────────┘     └───────────────────────────────────┘
```

### Option A: Graph Database (e.g., Neo4j)
* **Pros**:
  * Optimized for multi-hop relationship traversals (e.g. "friends of friends").
* **Cons**:
  * Difficult to shard horizontally. High operational complexity for a social network where we only need 1-hop queries (followers and followees).

### Option B: Sharded Relational Database (Selected)
* **Pros**:
  * Simple, flat schemas (`user_id`, `follower_id`). Sharding on `user_id` allows rapid indexes and scales write volumes horizontally.
* **Cons**:
  * Joins require careful caching strategies at the application layer to avoid routing queries across multiple database nodes.

---

## 3. Timeline Consistency: Strong vs. Eventual Consistency

### Option A: Strong Consistency (Synchronous Write Propagation)
* **Pros**:
  * Follower timelines update immediately. If User A posts, User B (follower) sees it instantly on page reload.
* **Cons**:
  * High write latency. The write operation must block until all follower timeline caches are updated.

### Option B: Eventual Consistency (Asynchronous Fan-out) (Selected)
* **Pros**:
  * Write path returns immediately. Fan-out runs in the background via Kafka and workers.
* **Cons**:
  * **Propagation Lag**: Follower feeds may take several seconds (or minutes during peak load) to update. This is an acceptable trade-off for social networks, as absolute real-time synchronization is not a critical safety requirement.

---

## 4. Cache Eviction: Active Users vs. All Users

### Option A: Cache Timelines of All Users
* **Pros**:
  * 100% cache hits for all timeline requests.
* **Cons**:
  * Severe memory cost. Millions of inactive users waste terabytes of expensive RAM.

### Option B: Cache Active Users Only (Selected)
* **Pros**:
  * Highly memory-efficient. We only load timelines into Redis for users who logged in within the last 3 days.
* **Cons**:
  * First login after an inactive period results in a cache miss, causing write/fetch overhead as the timeline is dynamically generated from database queries.
