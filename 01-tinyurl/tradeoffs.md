# Bottlenecks & Tradeoffs: Distributed URL Shortener (TinyURL)

Analyzing trade-offs and choosing the right compromises is the hallmark of professional systems architecture. This document evaluates the core architectural decisions made in the TinyURL system design.

---

## 1. Storage Layer: Cassandra (Wide-Column) vs. Relational Sharding

### Option A: Sharded Relational Database (e.g., PostgreSQL / MySQL)
* **Pros**:
  * Strong ACID guarantees.
  * Simple schema structure and mature querying patterns.
* **Cons**:
  * **Sharding Complexity**: Must select a shard key (like `hash(short_token)`). Shard migration, rebalancing, and directory-based routing add severe operational and application-level overhead.
  * **Master Failover Bottleneck**: Relational master-slave write topologies suffer from replica lag and promotion downtime during master crashes.

### Option B: Wide-Column NoSQL (Cassandra Cluster) (Selected)
* **Pros**:
  * **High Write Throughput**: LSM-tree storage engine writes sequentially to commit logs and memory buffers (Memtables), bypassing random disk seek latency.
  * **Masterless peer-to-peer (P2P)**: Any node can accept writes. No single point of failure.
  * **Linear Scalability**: Scale throughput and storage capacities linearly by adding new nodes.
* **Cons**:
  * **Eventual Consistency**: Tuning consistency levels (e.g. read quorum vs. write quorum) requires careful configuration to avoid serving stale read data.

---

## 2. Token Allocation: Zookeeper Range Leasing vs. Distributed Locking

```
               [ Concurrency Management Comparison ]

    ┌───────────────────────────────────┐     ┌───────────────────────────────────┐
    │     Zookeeper Range Leasing       │     │     Distributed Locking (Redis)   │
    ├───────────────────────────────────┤     ├───────────────────────────────────┤
    │  - Nodes lease 1,000,000 IDs      │     │  - Database checked for every ID  │
    │  - 0% lock contention on write    │     │  - Massive read/write overhead    │
    │  - Crash causes range loss        │     │  - Perfect sequence continuity    │
    └───────────────────────────────────┘     └───────────────────────────────────┘
```

### Option A: Distributed Locking / Mutex (e.g., Redis-based locks)
* **Pros**:
  * Guarantees strict sequential continuity. Zero token sequence numbers are wasted.
* **Cons**:
  * **Severe Latency Bottleneck**: Every single write request must wait for a lock acquisition/release or query a central counter, capping system write scalability.

### Option B: Zookeeper Range Allocation (Selected)
* **Pros**:
  * **Extreme Performance**: Nodes claim blocks in bulk (e.g. $1,000,000$ IDs) and manage local allocation thread-safely in-memory. Zero network latency or lock contention during writes.
* **Cons**:
  * **Sequence Gaps on Crash**: If a node crashes, its remaining allocated range block is lost, causing permanent gaps in the sequential namespace. However, this is negligible given the $3.52\times10^{12}$ available range capacity.

---

## 3. HTTP Redirects: 301 Moved Permanently vs. 302 Found

### Option A: HTTP 301 Redirect (Selected Default)
* **Pros**:
  * **Low Latency & Scalability**: Browser caches the redirection mapping. Subsequent clicks bypass our system load balancer entirely, eliminating read traffic on hot links.
* **Cons**:
  * **Loss of Telemetry**: Inability to track clicks, user agents, referrers, or locations for cached redirect requests.

### Option B: HTTP 302 Redirect (Selected for Analytics Sites)
* **Pros**:
  * **Rich Analytics Tracking**: Force every redirect request to route through our servers, enabling 100% accurate click analytics, user profiling, and geolocation logging.
* **Cons**:
  * **High Resource Load**: Substantially increases web server traffic and cache lookup workloads.

---

## 4. Security: Plain Sequential IDs vs. format-Preserving Encryption (FPE)

### Option A: Plain Sequential Base-62 IDs
* **Pros**:
  * Fast generation; zero mathematical computation overhead.
* **Cons**:
  * **Highly Predictable**: Links are easily crawlable (e.g., guessing `aaab1c` then `aaab1d` allows attackers to scrape all mappings).

### Option B: Format-Preserving Encryption / Feistel Cipher (Selected)
* **Pros**:
  * **Obfuscation**: Scrambles sequential counter values into pseudo-random numbers before Base-62 encoding. Retains 1-to-1 collision-free mapping while preventing guessing/scraping.
* **Cons**:
  * Minor CPU cycle cost during token generation.
