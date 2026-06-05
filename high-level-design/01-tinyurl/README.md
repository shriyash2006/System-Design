# Architectural Specification: Distributed URL Shortener (TinyURL)

This document provides a production-grade, highly available, and horizontally scalable architectural specification for a distributed URL Shortener (TinyURL) system. It outlines the requirements, mathematical scaling limits, network topologies, range allocation algorithms, storage trade-offs, and critical production safeguards.

---

## 1. Requirements Framework

### Functional Requirements
* **Short Link Generation**: Given a long URL, the system must generate a unique, short, and opaque token (e.g., `https://tiny.url/aB87zK2`).
* **HTTP Redirect (301 Routing)**: When a user accesses a short URL token, the system must perform an HTTP 301 (Permanent Redirect) lookup to redirect the client to the original long URL.
* **Custom Aliases (Optional / Extensible)**: Support for custom human-readable tokens (e.g., `https://tiny.url/my-promo`).

### Non-Functional Requirements
* **Ultra-Low Read Latency**: Read operations (redirection lookup) must resolve with a sub-10ms latency at the database/caching tier.
* **High Availability**: The system must target 99.999% availability ($5\times9$s) for read requests, prioritizing read availability over write availability (CAP theorem: AP model).
* **Fault Tolerance & No Single Point of Failure (SPOF)**: Every tier—from ingress to storage—must be clustered and highly redundant.
* **Scale**: The system must sustain massive write ingestion and read redirection volumes at global scale.

### API Design

The API tier is exposed as an idempotent RESTful gateway. All write payloads are JSON-formatted, and redirects use standard HTTP status codes.

#### POST `/api/v1/url` (Token Generation)
Creates a shortened URL from a long URL.

* **HTTP Method**: `POST`
* **Request Headers**:
  * `Content-Type: application/json`
  * `X-RateLimit-Token: <client_api_key>` (for authentication and rate limiting)
* **Request Payload**:
  ```json
  {
    "long_url": "https://sports.yahoo.com/news/live-updates-nba-finals-game-seven-001245782.html",
    "custom_alias": "nba-finals-g7",
    "ttl_seconds": 315360000
  }
  ```
  *(Note: `custom_alias` and `ttl_seconds` are optional parameters)*
* **Response Headers**:
  * `Status: 201 Created`
  * `Content-Type: application/json`
* **Response Payload**:
  ```json
  {
    "short_url": "https://tiny.url/nba-finals-g7",
    "short_url_token": "nba-finals-g7",
    "long_url": "https://sports.yahoo.com/news/live-updates-nba-finals-game-seven-001245782.html",
    "created_at": "2026-06-05T06:45:00Z",
    "expires_at": "2036-06-02T06:45:00Z"
  }
  ```

#### GET `/{short_url_token}` (Redirection Lookup)
Resolves the short token and issues a permanent redirect.

* **HTTP Method**: `GET`
* **Path Parameter**: `short_url_token` (e.g., `aB87zK2`)
* **Response Headers**:
  * `Status: 301 Moved Permanently`
  * `Location: https://sports.yahoo.com/news/live-updates-nba-finals-game-seven-001245782.html`
  * `Cache-Control: public, max-age=86400`
* **Response Payload**: None (Body is empty; redirection is handled natively by browser parsing of the `Location` header).

> [!TIP]
> A `301 Moved Permanently` redirect is preferred over `302 Found` (Temporary) because it instructs search engines and browsers to cache the destination mapping locally. This drastically reduces read traffic hitting our load balancers for highly viral links. However, for systems that require real-time redirect analytics for every click, a `302 Found` redirect is utilized.

---

## 2. Back-of-the-Envelope Scaling Math

### Traffic & Generation Volumes
To dimension the scaling demands, we assume a continuous write throughput of **1,000 writes/sec**.

$$\text{Writes per year} = 1,000\text{ writes/sec} \times 86,400\text{ sec/day} \times 365\text{ days/year} \approx 31.536\text{ Billion writes/year}$$

Assuming a standard read-to-write ratio of **10:1**:

$$\text{Read Redirection Rate} = 10,000\text{ reads/sec}$$

$$\text{Reads per year} = 31.536\text{ Billion writes/year} \times 10 \approx 315.36\text{ Billion reads/year}$$

### Storage Sizing (10-Year Horizon)
* Let the average payload size for a mapping record (Token + Long URL + Expiry + Created Time) be **500 Bytes**.
* **Total unique records stored over 10 years**: $31.536\text{ Billion/year} \times 10\text{ years} = 315.36\text{ Billion records}$.

$$\text{Total Storage Required} = 315.36\times10^9\text{ records} \times 500\text{ bytes} = 157.68\text{ Terabytes}$$

### Character Space & Encoding Analysis
We encode internal system-generated integer IDs (64-bit unsigned integers) into URLs using **Base-62 encoding** ($[a-z, A-Z, 0-9]$).
The maximum token permutations for token lengths ($N$) are governed by $62^N$:

| Token Length ($N$) | Mathematical Permutation Space ($62^N$) | Lifespan Capacity (at $31.536\text{ Billion}$ writes/year) |
| :---: | :--- | :--- |
| **5** | $62^5 = 916,132,832$ (~916 Million) | $\approx 10$ days |
| **6** | $62^6 = 56,800,235,584$ (~56.8 Billion) | $\approx 1.8$ years |
| **7** | $62^7 = 3,521,614,606,208$ (~3.52 Trillion) | $\approx 111.6$ years |

> [!IMPORTANT]
> A **7-character token slot** is mathematically required to support a 10-year enterprise horizon. A 6-character space is highly vulnerable to exhaustion in under 2 years, whereas a 7-character space provides a massive safety buffer ($3.52\times10^{12}$ available keys), consuming only a negligible extra byte per link.

---

## 3. Core Architecture & Request Flow

The system employs a multi-tiered, decoupled microservices architecture. Reads utilize aggressive caching, while writes use a range-allocated distributed coordination pattern to generate keys synchronously without centralized bottlenecks.

```mermaid
graph LR
    %% Define Nodes
    Client([Client Browser])
    LB[Edge Load Balancer <br> NGINX / Cloudflare]
    WebServers[Web Server Pool <br> Stateless API Nodes]
    ZK[Apache Zookeeper <br> Range Coordinator]
    Cache[Distributed Cache <br> Redis Cluster]
    Cassandra[(Data Layer <br> Cassandra Cluster)]

    %% Define Flows
    Client -->|1. GET/POST Request| LB
    LB -->|2. Route Traffic| WebServers
    WebServers -.->|3. Heartbeat & Range Acquisition| ZK
    WebServers -->|4a. Lookup Cache (GET)| Cache
    Cache -->|4b. Cache Miss: Query DB| Cassandra
    WebServers -->|5. Write Record (POST)| Cassandra
    WebServers -.->|6. Populate Cache on Miss| Cache

    %% Styling
    style Client fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    style LB fill:#efebe9,stroke:#4e342e,stroke-width:2px;
    style WebServers fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    style ZK fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    style Cache fill:#eceff1,stroke:#37474f,stroke-width:2px;
    style Cassandra fill:#f3e5f5,stroke:#4a148c,stroke-width:2px;
```

### Request Flow Topography
1. **Write Request Path (Generation)**:
   * **Ingress**: Client issues a `POST /api/v1/url` which hits the **Edge Load Balancer**.
   * **Application Processing**: The load balancer forwards the request to an instance in the **Web Server Pool**.
   * **Range Ingestion**: The web server retrieves a local sequence number from its pre-allocated range block. (This range block was assigned by **Apache Zookeeper**).
   * **Encoding**: The server performs Base-62 encoding on the sequential integer ID to produce a 7-character string token.
   * **Persistence**: The server writes the token-to-URL mapping to the **Cassandra Cluster**.
   * **Cache Warmup**: The server asynchronously writes the new mapping to the **Redis Cluster** to preemptively warm up reads.
   * **Response**: The server returns `201 Created` with the shortened link.

2. **Read Request Path (Lookup & Redirection)**:
   * **Ingress**: Client clicks `https://tiny.url/aB87zK2`. The HTTP GET request hits the **Edge Load Balancer**.
   * **Resolution Tier 1 (Cache)**: The server queries the **Redis Cluster** (key-value hash lookup for token `aB87zK2`). If it's a **Cache Hit**, the server immediately returns a `301 Moved Permanently` with the original URL.
   * **Resolution Tier 2 (Data Store)**: On a **Cache Miss**, the server queries the **Cassandra Cluster**.
     * If the mapping is found, the server writes it back to Redis (with an expiration TTL) and responds with `301`.
     * If the mapping is not found, the server returns a `404 Not Found`.

---

## 4. The Token Range Allocation Algorithm

The principal design challenge in a distributed URL shortener is generating unique keys efficiently without introducing central database locks or transaction collisions. To achieve this, the system uses a **Token Range Allocation Algorithm** coordinated by **Apache Zookeeper**.

### Distributed Range Coordination
Instead of hitting a database sequence generator or using distributed locks for every write request, we divide the 64-bit integer range space and distribute allocations in bulk.

```
Total Numeric Range: [ 1 --------------------------------------------------- 3,521,614,606,208 ]
                                             |
                                  Divided into Ranges of 1,000,000
                                             |
                      +----------------------+----------------------+
                      |                      |                      |
Node 1: [1 - 1,000,000]   Node 2: [1,000,001 - 2,000,000]   Node 3: [2,000,001 - 3,000,000]
```

1. **Zookeeper Znode Hierarchy**: A persistent node `/ranges` is maintained in Zookeeper to track the globally distributed master offset counter.
2. **Web Node Bootstrapping**:
   * When a stateless API server starts up, it connects to Zookeeper.
   * It performs an atomic *compare-and-swap* (CAS) transaction to increment the value in `/ranges` by **1,000,000**.
   * **Example**: If the current value in Zookeeper is $5,000,000$, the server claims the block $[5,000,001 \text{ to } 6,000,000]$ and updates Zookeeper to $6,000,000$.
3. **Local In-Memory Counter**:
   * The API server stores this range limits in memory.
   * As write requests flow into this specific server, it increments a local `AtomicLong` thread-safe counter from $5,000,001$ upwards.
   * For each write, the counter value is Base-62 encoded. For example:
     $$\text{ID } 5,000,001 \xrightarrow{\text{Base-62}} \text{`aaab1c'}$$
   * The server never communicates with Zookeeper again until its local counter reaches $6,000,000$.
4. **Out-of-Path Zookeeper Execution**: Since Zookeeper is only contacted once every $1,000,000$ write requests, it is entirely removed from the hot execution path. A single Zookeeper cluster can support hundreds of thousands of concurrent writes across the system.

### Edge Case: Node Crashes and Block Waste
If an API web server crashes, restarts, or is terminated by an auto-scaler midway through its range (e.g., at $5,050,000$):
* The remaining $950,000$ tokens in its allocated block $[5,050,001 \text{ to } 6,000,000]$ are lost forever.
* Upon reboot, the server requests a brand-new range (e.g., $[12,000,001 \text{ to } 13,000,000]$) from Zookeeper.

#### Architectural Trade-off Evaluation
This loss of range values is an **acceptable architectural trade-off**.
* At $3.52\times10^{12}$ capacity (7 characters), if a server crashes 100 times a day and wastes 1 million keys per crash, it wastes $36.5\text{ Billion keys/year}$.
* Even at this highly elevated failure rate, the overall space exhaustion window is only reduced by ~1% ($36.5\text{ Billion} \div 3.52\text{ Trillion} = 1\%$). The capacity is more than sufficient to absorb block waste.

---

## 5. Storage Tier Selection Trade-offs

A highly scaling URL Shortener has a very clean data structure:
* `id` (64-bit integer, internal key)
* `short_token` (7-character string, primary key/lookup index)
* `long_url` (varchar/text, mapping destination)
* `created_at` (timestamp)
* `expires_at` (timestamp, nullable)

```
+-------------+-------------+----------------------+------------+------------+
|     id      | short_token |       long_url       | created_at | expires_at |
+-------------+-------------+----------------------+------------+------------+
| 5000001     | aaab1c      | https://sports.ya... | 2026-06-05 | 2036-06-02 |
+-------------+-------------+----------------------+------------+------------+
```

### Relational Database Sharding vs. Wide-Column NoSQL

| Evaluation Metric | Relational Database (e.g., PostgreSQL Sharded) | Wide-Column NoSQL (Cassandra Cluster) |
| :--- | :--- | :--- |
| **Write Scalability** | Requires application-level or middleware sharding based on hash of `short_token`. Writing to multiple masters is highly complex and error-prone. | Masterless architecture. Any node can accept a write. High-throughput writes are natively supported by append-only LSM trees. |
| **Operational Complexity** | High. Adding database shards (resharding) requires data migration, re-hashing ranges, and managing complex master-slave failover. | Low. Node expansion is linear. The cluster re-balances token rings automatically when nodes are added or removed. |
| **Query Model Fit** | Relational features (joins, complex transactions, ACID) are unused. We only need simple point-lookup queries. | Perfect fit. Cassandra acts as a massive distributed hash table. Point-queries via Partition Key (`short_token`) are optimized. |
| **Failover & High Availability** | Master node represents a single point of failure (SPOF) before failover scripts promote a standby node. | Fully decentralized (peer-to-peer). No master node exists. Active node failures cause zero write/read downtime. |

### Cassandra Data Modeling
The table is modeled with the `short_token` acting as the partition key to guarantee all lookup reads route to the exact storage node holding that mapping.

```sql
CREATE KEYSPACE tinyurl_keyspace 
WITH replication = {'class': 'NetworkTopologyStrategy', 'us-east': 3, 'us-west': 3};

CREATE TABLE tinyurl_keyspace.url_mappings (
    short_token text,
    id bigint,
    long_url text,
    created_at timestamp,
    expires_at timestamp,
    PRIMARY KEY (short_token)
);
```

---

## 6. Production Edge Cases & System Safeguards

### Security & Crawlability (Predictability)
Because our token generation relies on sequential counters ($5,000,001 \rightarrow 5,000,002$), the output short tokens are highly predictable (`aaab1c` $\rightarrow$ `aaab1d`). A simple crawler script can easily guess and scrape all shortened URLs in our system, posing security, privacy, and scraping threats.

#### Solution: Format-Preserving Cryptographic Encryption
Instead of feeding the sequential range ID directly into the Base-62 encoder, we pass it through a **Format-Preserving Encryption (FPE)** algorithm or a mini-Feistel Cipher using a secure key:

$$\text{Sequential counter ID: } 5,000,001 \xrightarrow{\text{Feistel Cipher (with secret key)}} \text{Obfuscated ID: } 2,987,144,302 \xrightarrow{\text{Base-62}} \text{`z9Xq7a'}$$

* **Reversibility**: The Feistel cipher is a bijective mapping; every inputs map to exactly one output, meaning zero collisions are introduced.
* **Obfuscation**: Consecutive counter IDs output wildly different, randomized tokens, rendering scraping and enumeration attacks impossible.

---

### Ingestion Rate Limiting & DDoS Prevention
Due to the low cost of writing short URLs, malicious users or bots can script high-rate requests to exhaust Zookeeper token blocks or saturate database storage.

```
       [ Malicious Client Request ]
                    │
                    ▼
    [ API Gateway (e.g., Kong / APISIX) ]
      └─ Rate Limiter (Redis-backed Token Bucket)
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
[ Exceeded: 429 Limit ]   [ Under Limit: Proceed ]
```

* **Token Bucket Algorithm**: Implemented at the API gateway layer using Redis to enforce rate limits per Client IP / User API key.
* **Ingestion Limits**:
  * Free Tier: Max 10 writes/minute, 100 writes/day.
  * Enterprise Tier: Custom quotas.
* **Action on Violation**: Returns standard `HTTP 429 Too Many Requests` status codes.

---

### Analytics & Edge Caching
To maintain sub-10ms redirection times globally, we must optimize cache placement.

1. **Edge Cache (CDN)**: Highly static URL redirects (like system-wide marketing links) are cached directly at the CDN layer (Cloudflare/Fastly) using HTTP headers:
   `Cache-Control: public, max-age=86400, stale-while-revalidate=3600`
2. **Metadata-Driven Local Caching**:
   * API servers log metadata for redirections (e.g., Client IP, User-Agent, Geolocation, Timestamp) asynchronously to **Apache Kafka**.
   * An analytics consumer (e.g., Spark / Flink) processes these events to identify regional surges in specific link lookups (e.g., a link trending in Europe).
   * The analytics pipeline triggers a cache warm-up, copying the high-demand keys to regional Redis replica zones (e.g., from `us-east` Redis to `eu-west` Redis) before local nodes experience cache misses.
