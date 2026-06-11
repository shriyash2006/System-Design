# Bottlenecks & Tradeoffs: Real-Time Messaging App

This document analyzes the engineering trade-offs made in the Messaging App (WhatsApp/Discord) architecture to achieve low-latency message delivery and high availability.

---

## 1. Connection Protocol: WebSockets vs. Long Polling

### Option A: Long Polling
* **Pros**:
  * Utilizes standard stateless HTTP connections. Works out-of-the-box with any standard proxy, firewall, or load balancer.
* **Cons**:
  * **High Protocol Overhead**: Re-opening connection sockets repeatedly wastes CPU cycles and network packets.
  * **Latent Delivery**: Creating a new HTTP request on event arrival introduces packet delivery delay.

### Option B: WebSockets (Selected)
* **Pros**:
  * **True Duplex duplexing**: Allows instant, bidirectional data flow over a single TCP socket connection. Low packet headers overhead.
* **Cons**:
  * **Stateful Management**: Requires custom load balancing and session stickiness. Firewalls may block persistent WS connections, requiring fallback pipelines.

---

## 2. Message History Storage: Cassandra (NoSQL) vs. MySQL / PostgreSQL

```
              [ Messaging Storage Performance Comparison ]

    ┌───────────────────────────────────┐     ┌───────────────────────────────────┐
    │          MySQL/PostgreSQL         │     │         Apache Cassandra          │
    ├───────────────────────────────────┤     ├───────────────────────────────────┤
    │  - Strong schema constraints      │     │  - Fast write-path append logs    │
    │  - Index lock contention on write │     │  - Simple key-value access model  │
    │  - Complex sharding setup         │     │  - No joins/relational operations │
    └───────────────────────────────────┘     └───────────────────────────────────┘
```

### Option A: Relational Database (SQL)
* **Pros**:
  * Strong transactions. Easy query definitions for complex relational operations.
* **Cons**:
  * **Write Path Index Locks**: Indexes must update on every single insert. Under high write traffic, this causes disk I/O bottlenecks and write queue build-up.

### Option B: Wide-Column NoSQL (Cassandra) (Selected)
* **Pros**:
  * **Write-Optimized Storage**: Uses LSM-trees to execute writes sequentially to commit logs and memory buffers, avoiding random disk seeks.
  * **No Single Point of Failure**: Masterless P2P architecture scales throughput linearly by adding hardware nodes.
* **Cons**:
  * No support for complex JOIN operations. The application must perform data consolidation logic.

---

## 3. Group Chat Message Distribution: Fan-out on Write vs. Fan-out on Read

### Option A: Fan-out on Read
* **Pros**:
  * The write path is simple. A message is written once to a conversation inbox.
* **Cons**:
  * When a user loads their chat feed, the system must query many different message indices and sort them, causing slow load times.

### Option B: Fan-out on Write (Selected with Capped Groups)
* **Pros**:
  * Messages are cloned and written directly to each conversation participant's inbox queue. Reading a chat timeline is extremely fast ($\mathcal{O}(1)$).
* **Cons**:
  * Clones are created for every user.
  * **Mitigation**: We restrict group chat sizes to a maximum of 150 members. Cloning a message 150 times is a lightweight operations that avoids scaling bottlenecks.

---

## 4. End-to-End Encryption (E2EE): Client-Side vs. Server-Side Hashing

### Option A: E2EE (Signal Protocol) (Selected)
* **Pros**:
  * High user privacy. Cryptographic keys are generated solely on client devices, meaning the server only routes unreadable ciphertexts.
* **Cons**:
  * **No Server-Side Search**: The server cannot search message histories for keywords since it does not have the decryption keys. Search must run locally on the client device.
  * High client-side processing overhead (key rotation and decryption cycles).
