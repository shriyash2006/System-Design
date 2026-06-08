# Scaling Strategy: URL Frontier, Deduplication, and SSR

Scaling a web crawler to process 1 billion pages per month ($\approx 386\text{ pages/sec}$) requires optimizing the fetching pathway, managing server politeness, and handling dynamic JavaScript applications. This document outlines the scaling strategies.

---

## 1. Deep Dive into the URL Frontier

The URL Frontier is the core scheduler of the crawler. It determines which URLs to crawl next while enforcing two critical, opposing constraints: **Prioritization** (crawling valuable pages first) and **Politeness** (not overloading target hosts).

```
                      [ Extracted URLs Ingest ]
                                 │
                                 ▼
                     [ Prioritizer Component ]
                                 │
             ┌───────────────────┼───────────────────┐
             ▼ (Q1: High)        ▼ (Q2: Med)         ▼ (Qn: Low)
       [ Front Queue 1 ]   [ Front Queue 2 ]   [ Front Queue n ]
             └───────────────────┬───────────────────┘
                                 │
                     [ Front Queue Selector ]
                      (Biased Random Pull)
                                 │
                                 ▼
                    [ Back Queue Router Table ]
             ┌───────────────────┼───────────────────┐
             ▼                   ▼                   ▼
      [ Back Queue B1 ]   [ Back Queue B2 ]   [ Back Queue Bm ]
       (wikipedia.org)      (nasa.gov)          (github.com)
             └───────────────────┬───────────────────┘
                                 │
                     [ Back Queue Selector ]
                     (Respects Delay Timers)
                                 │
                                 ▼
                      [ Fetcher Worker Pool ]
```

### A. Prioritization (The Front Queues)
* **Value Calculation**: Input URLs pass through a Prioritizer that calculates a score based on PageRank, traffic frequency, and host authority.
* **Queue Mapping**: URLs are mapped to one of $N$ Priority Front Queues ($Q_1$ to $Q_n$).
* **Biased Selection**: The Front Queue Selector pulls URLs from these queues using a biased probability distribution. Higher-priority queues ($Q_1$) have a much higher likelihood of being pulled from, ensuring high-value pages are prioritized.

### B. Politeness (The Back Queues)
* **Domain Isolation**: To avoid DDoS-ing any target host, we map URLs such that a single Back Queue ($B_1$ to $B_m$) contains URLs from **exactly one host domain** (e.g. all `wikipedia.org` URLs sit in $B_1$).
* **Router Table**: The Router Table maps hostnames to their respective back queues.
* **Crawl Delay Timer**: The Back Queue Selector manages active pulling. When a fetcher thread extracts a URL from $B_1$ (e.g. Wikipedia), it locks the queue and records a timestamp. The queue remains locked for a politeness buffer time (e.g. 1000ms).
* If another thread attempts to fetch a URL for Wikipedia before the timer expires, the selector bypasses $B_1$ and pulls from another active, unlocked domain queue (e.g. $B_2$).

---

## 2. Space-Efficient Deduplication (Bloom Filters)

At 1 billion pages/month, check operations for visited URLs and duplicate content hashes cannot perform round-trip disk lookups.

* **Bloom Filter Structure**: We load a space-efficient Bloom Filter into RAM (using **Redis Bloom** or clustered Memcached).
* **Speed & Efficiency**: Bloom filters perform membership queries in $\mathcal{O}(k)$ time (where $k$ is the number of hash functions) using only a few bits per element.
* **Behavior**:
  * **No False Negatives**: If the filter returns `does not exist`, it is guaranteed to be unique. The page is crawled and added to the filter.
  * **Low False Positives**: If the filter returns `exists`, there is a tiny probability of a false match. In this case, we either drop the URL (acceptable trade-off at scale) or query Cassandra to verify.

---

## 3. Server-Side Rendering (SSR) processing

Modern websites compile content dynamically using client-side JavaScript (React, Angular, Vue). A simple raw HTTP client fetcher only receives empty target containers.

* **Headless Browser Pool**: Fetcher servers run headless browser threads (e.g. Puppeteer, Playwright, or headless Chrome).
* **Execution Block**: Workers download the initial markup, execute the Javascript scripts, and construct the final Document Object Model (DOM).
* **Link Extraction**: The URL extractor parses the fully rendered DOM to extract links, ensuring dynamic web elements are crawled.

---

## 4. Consistent Hashing & DNS Cache

* **Consistent Hashing**: Fetcher worker nodes are organized on a consistent hash ring. URLs are routed to specific fetcher nodes based on the hash of their host domain. This guarantees local cache locality for DNS records and robots.txt rules while minimizing re-sharding penalties if a fetcher node fails.
* **Custom DNS Caching**: Fetchers bypass standard OS DNS network calls by using an internal DNS Resolver Cache. Resolving `wikipedia.org` takes less than 1ms from local memory, avoiding standard DNS resolution round-trip network lag.
