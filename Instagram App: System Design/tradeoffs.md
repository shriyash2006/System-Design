# Tradeoff Matrix: Key Design Decisions

Building a globally available, highly durable social media system like Instagram requires evaluating storage layouts and networking protocols. Below are the key engineering trade-offs evaluated in this design.

---

## 1. Timeline Feed Generation: Push vs. Pull

| Dimension | Push Model (Fan-out on Write) | Pull Model (Fan-out on Read) |
| :--- | :--- | :--- |
| **Write Amplification** | **High**: A single post by a user with $N$ followers requires $N$ database updates to cache timelines. | **Low**: A single post requires exactly 1 database insert into the global post table. |
| **Read Latency** | **Ultra-Low**: Feeds are pre-computed and fetched directly from high-performance Redis nodes ($O(1)$). | **High**: Requires querying followers, fetching their posts, merging them, and sorting on the fly. |
| **Storage Footprint** | **High**: Redundant lists of post IDs exist across hundreds of millions of user feed caches. | **Low**: Only the single post record and follow mapping metadata are persisted. |
| **Edge Cases** | **Hotkey / Celebrity Problem**: A celebrity posting causes a write storm that degrades message queue performance. | **Inactive User Overhead**: Wastes query calculations generating timelines for users who haven't logged in. |

* **Architectural Decision**: Instagram employs a **Hybrid model**. Standard users utilize the Push model to maintain fast read speeds. Celebrities ($>25,000$ followers) are excluded from the write-time fan-out queue. Instead, their posts are pulled dynamically on-demand during timeline generation.

---

## 2. Core Metadata Storage: Relational (PostgreSQL) vs. NoSQL (Cassandra)

We compared relational storage against eventually consistent NoSQL stores for transactional metadata:

* **PostgreSQL (Selected with Sharding)**:
  * **Pros**: ACID-compliant transactions ensure relational metadata consistency (e.g., ensuring a post media record doesn't point to a missing file). Easy indexing for multi-key queries (user posts, follow pairs).
  * **Cons**: Scaling requires manual horizontal partitioning (sharding) on `user_id` and setting up master-replica replication pools.
* **NoSQL (Cassandra/DynamoDB)**:
  * **Pros**: Highly scalable out-of-the-box. Easy horizontal scaling for high QPS writes.
  * **Cons**: Lacks ACID transaction support across tables, leading to eventual consistency anomalies (e.g., deleted follows still seeing feed posts for a short window).

* **Architectural Decision**: We select **Sharded PostgreSQL** for transactional user, follow, post, and media records to maintain database schema integrity. We use **Redis** as an in-memory caching layer to handle high-frequency feed reads.

---

## 3. Nested Comments: Adjacency List vs. Nested Set Model

Supporting infinite nested comment threads beneath posts requires careful database schema mapping:

### Option A: Adjacency List (Parent ID Mapping)
```sql
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INT REFERENCES posts(id),
    parent_id INT REFERENCES comments(id), -- Points to parent comment
    user_id INT REFERENCES users(id),
    content TEXT,
    created_at TIMESTAMP
);
```
* **Pros**: Extremely simple write operations ($O(1)$ inserts).
* **Cons**: Retrieving a deep comment tree requires recursive SQL queries (`WITH RECURSIVE`) or multiple database round-trips, which degrades system read performance.

### Option B: Nested Set Model
We map comments with left (`lft`) and right (`rgt`) boundary numbers representing hierarchy:
```sql
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INT REFERENCES posts(id),
    lft INT,
    rgt INT,
    content TEXT
);
```
* **Pros**: Finding all children of a parent comment requires a single range query: `SELECT * FROM comments WHERE lft BETWEEN parent.lft AND parent.rgt`, eliminating deep joins.
* **Cons**: Inserting or moving a comment requires updating the `lft` and `rgt` values of all subsequent nodes in the tree, which triggers intensive write operations.

* **Architectural Decision**: We select the **Adjacency List model with cached subtrees**. For high-concurrency platforms, write performance must not be blocked by recalculating nested sets. We cache the serialized subtrees in Redis to optimize read latency.
