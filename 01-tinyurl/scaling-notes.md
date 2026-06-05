# Scaling Strategy: Distributed URL Shortener (TinyURL)

Scaling a URL shortener requires designing for high write throughput (1,000 writes/sec) and massive read workloads (10,000 reads/sec). This document outlines the key distributed patterns used to eliminate bottlenecks.

---

## 1. Token Range Allocation (Distributed Sequence Coordination)

To generate unique shortened tokens without database write collisions or transaction overhead, we utilize a coordinated sequence allocator:

```
                  [ Apache Zookeeper Cluster ]
                               │
            ┌──────────────────┼──────────────────┐
            ▼ (CAS Lease)      ▼ (CAS Lease)      ▼ (CAS Lease)
      [ Web Node 1 ]     [ Web Node 2 ]     [ Web Node 3 ]
       Allocated:         Allocated:         Allocated:
       1M - 2M            2M - 3M            3M - 4M
            │                  │                  │
      [ AtomicLong ]     [ AtomicLong ]     [ AtomicLong ]
      (Thread-safe)      (Thread-safe)      (Thread-safe)
```

### The Allocation Algorithm
1. A shared, persistent node (`/ranges/offset`) is maintained in **Apache Zookeeper** to track the globally distributed master offset index.
2. At boot time, a stateless API Web Server connects to Zookeeper and executes an atomic compare-and-swap (CAS) operation to claim a numeric range block (e.g., $1,000,000$ integers).
3. The server stores this block limits in local memory (e.g., starts at $1,000,001$, ends at $2,000,000$).
4. As write requests arrive, the server increments a local thread-safe `AtomicLong` counter. It performs Base-62 encoding on this incremented counter to construct the URL token.
5. Zookeeper is removed entirely from the per-request transaction path. The server only communicates with Zookeeper when its local block is depleted and it needs to lease a new one.

---

## 2. Multi-Tier Caching System (Redis / CDN)

Redirection reads follow a highly read-heavy distribution (often obeying the 80/20 rule, where 20% of links drive 80% of redirect requests).

### Cache-Aside Pattern
1. **Cache Read Hook**: On receiving a request for token `aB87zK2`, the API node performs a read check in the local **Redis Cluster** using the token as the lookup key.
2. **Cache Hit**: Returns original URL immediately (resolves in <2ms).
3. **Cache Miss**: On a miss, the API node queries the **Cassandra Cluster**. It then writes the retrieved mapping back to the Redis cluster with an associated Time-To-Live (TTL) header before issuing the redirect.

### Regional Edge Caching
* Highly active links are cached at Edge CDNs (e.g., Cloudflare, Fastly) using `Cache-Control` headers. This routes lookups away from our infrastructure entirely.
* Real-time metadata analytics pipelines (processed asynchronously via Apache Kafka and Spark Streaming) scan log events to preemptively warm up caches in regional Redis clusters when a link becomes popular in a specific geographic area.

---

## 3. Format-Preserving Cryptographic Encryption (Predictability Safeguard)

Because range-allocated IDs are sequential ($1 \rightarrow 2 \rightarrow 3$), standard Base-62 encoding produces sequential, easily predictable short tokens:
$$\text{ID } 5,000,001 \rightarrow \text{`aaab1c'}$$
$$\text{ID } 5,000,002 \rightarrow \text{`aaab1d'}$$

To prevent automated malicious bots from crawling the namespace and scraping private shortened links, we implement a **Format-Preserving Encryption (FPE)** Feistel Cipher:

```
[ Sequential ID: 5,000,001 ] ──► [ Feistel Cipher (Secret Key) ] ──► [ Obfuscated ID: 2,987,144,302 ] ──► [ Base-62 Encode ] ──► `z9Xq7a`
```

* **Mathematical Security**: The Feistel cipher acts as a bijective, collision-free mapping function.
* **Obfuscation**: Consecutive IDs output wildly scrambled, pseudo-random strings. This prevents scrapers from guessing valid links while maintaining complete data uniqueness.

---

## 4. Ingestion Rate Limiting & DDoS Safeguards

To prevent Denial-of-Service attacks or token namespace depletion by rogue scripts, the API gateway tier (e.g., Kong, Nginx) implements rate-limiting policies:

* **Redis Token Bucket Rate Limiting**: Limit write requests per IP and API Client Key.
* **Write Quotas**: Enforce strict quotas (e.g. Free Tier is restricted to 10 writes/minute, 100/day).
* **Action**: Requests exceeding limits are rejected immediately at the gateway layer with an `HTTP 429 Too Many Requests` status code, preserving web server and database resources.
