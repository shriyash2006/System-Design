# Bottlenecks & Tradeoffs: Distributed Cloud Storage

Analyzing tradeoffs is critical when coordinating file systems across millions of client devices. This document evaluates the core choices made in the Dropbox / Google Drive system design.

---

## 1. Database Choice: Relational (MySQL/PostgreSQL) sharded vs. NoSQL

### Option A: NoSQL (e.g. Cassandra / DynamoDB)
* **Pros**:
  * Linear write scalability. Operates without database sharding complexity.
* **Cons**:
  * **Lack of ACID Transactions**: Eventual consistency is highly risky for file sync. Concurrent edits from multiple devices can cause file directory corruption, duplicate folders, and file version conflicts.

### Option B: Sharded Relational Database (MySQL/PostgreSQL) (Selected)
* **Pros**:
  * **Strict ACID Compliance**: Essential for resolving file metadata transactions, locks, and parent-child hierarchy movements correctly.
* **Cons**:
  * **Scale Ingestion Bottleneck**: Requires sharding MySQL using consistent hashing based on `file_id` or `workspace_id`. Adds operational complexity to handle database growth.

---

## 2. Notification Gateway: Long Polling vs. WebSockets

```
               [ Notification Transport Comparison ]

    ┌───────────────────────────────────┐     ┌───────────────────────────────────┐
    │           Long Polling            │     │            WebSockets             │
    ├───────────────────────────────────┤     ├───────────────────────────────────┤
    │  - Standard HTTP/S requests       │     │  - Full-duplex persistent TCP     │
    │  - Easy load balancing & proxy    │     │  - Complex load-balancer routing  │
    │  - High connection resource load  │     │  - Fast real-time updates         │
    └───────────────────────────────────┘     └───────────────────────────────────┘
```

### Option A: WebSockets
* **Pros**:
  * Low latency, full-duplex TCP connections. Perfect for active, bidirectional data exchange.
* **Cons**:
  * **High Connection Overhead**: Maintaining millions of idle websocket sockets is resource-heavy. WebSockets do not play well with standard corporate firewall proxies.

### Option B: Long Polling (Selected)
* **Pros**:
  * Standard HTTP/S. Load balancing and reverse proxies (NGINX/HAProxy) route requests easily without socket configuration.
  * Fits the storage model: Clients only receive updates occasionally (when a change occurs), not continuously.
* **Cons**:
  * Slightly higher latency to establish a connection compared to an active WebSocket.

---

## 3. Block Size Threshold: 4 MB vs. 1 MB Blocks

### Option A: Small Blocks (1 MB)
* **Pros**:
  * Highly granular diffs. Minimal network bandwidth wasted on minor text file edits.
* **Cons**:
  * **Massive Metadata Database**: Splitting a 1 GB video file creates 1,000 block rows in the database. Multiplying this by 500 million active users will cause database metadata indexing tables to swell.

### Option B: Large Blocks (4 MB) (Selected)
* **Pros**:
  * Balances block count metadata against networking overhead. Optimal size for chunking large media/data objects.
* **Cons**:
  * Modifying a 1-byte character in a 4 MB block forces the client to re-upload the entire 4 MB block.

---

## 4. Deduplication: Global vs. User-Level Deduplication

### Option A: User-Level Deduplication Only
* **Pros**:
  * Highly secure. Prevents data leaks between users (avoiding side-channel attacks where an attacker guesses if another user has uploaded a file).
* **Cons**:
  * Substantial storage waste. Millions of users uploading identical files (e.g. popular movies, software binaries) require redundant storage.

### Option B: Global Deduplication (Selected)
* **Pros**:
  * **Massive Cost Savings**: Blocks are hashed globally. Matches are referenced rather than re-uploaded, saving petabytes of storage.
* **Cons**:
  * Requires client block hashing verification mechanisms to prevent unauthorized users from gaining access to a block by simply guessing its hash (solved using Proof of Ownership protocols).
