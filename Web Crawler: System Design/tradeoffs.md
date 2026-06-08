# Bottlenecks & Tradeoffs: Web Crawler System

Evaluating architectural compromises is essential when designing systems that process petabytes of unstructured web data. This document assesses the core tradeoffs in the Web Crawler architecture.

---

## 1. Storage Choice: Cassandra (NoSQL Column-Store) vs. Relational (RDBMS)

### Option A: Relational Database (e.g. MySQL / PostgreSQL)
* **Pros**:
  * Strong transactions. Easy query definitions for complex relational operations.
* **Cons**:
  * **Scale Bottleneck**: Relational databases fail under the massive write traffic of a global crawler (adding hundreds of millions of states daily).
  * **Sharding Overhead**: Sharding relational databases to handle table growth is operationally complex.

### Option B: Wide-Column NoSQL (Cassandra) (Selected)
* **Pros**:
  * **High Write Throughput**: Optimized for write-heavy workloads via append-only commit logs.
  * **No Master Bottleneck**: Fully distributed P2P node rings allow linear scaling by adding hardware.
  * **Dynamic Schema**: Column-family layout maps well to URL metadata records.
* **Cons**:
  * No support for complex JOIN operations. The application must perform data consolidation logic.

---

## 2. Duplicate Detection: Content Fingerprinting (MD5) vs. Full Matching

```
             [ Page Deduplication Strategy Trade-offs ]

    ┌───────────────────────────────────┐     ┌───────────────────────────────────┐
    │     MD5 Content Fingerprint       │     │       Full HTML Comparison        │
    ├───────────────────────────────────┤     ├───────────────────────────────────┤
    │  - 128-bit hash stored in RAM     │     │  - Exact match validation         │
    │  - Fast O(1) Bloom Filter check   │     │  - High storage & disk read costs │
    │  - Risk of minor hash collision   │     │  - Zero false positives           │
    └───────────────────────────────────┘     └───────────────────────────────────┘
```

### Option A: Full Page Markup Comparison
* **Pros**:
  * 100% accuracy. Zero chance of mistakenly treating distinct pages as duplicates.
* **Cons**:
  * Extremely slow. Requires saving entire HTML files and retrieving them for comparison, saturating disk I/O and network bandwidth.

### Option B: Content Hashing (MD5) + Bloom Filter (Selected)
* **Pros**:
  * **Extreme Speed**: Raw HTML is hashed into a 128-bit key. Comparisons resolve in RAM via Bloom Filters, bypassing storage lookups.
  * **Low Storage Footprint**: Storing a 128-bit fingerprint is orders of magnitude cheaper than retaining megabytes of raw HTML.
* **Cons**:
  * **Hash Collisions**: Theoretically, two distinct pages could produce the same MD5 hash, causing one page to be discarded. This is acceptable at scale, as web crawling is a statistical discovery process.

---

## 3. Crawl Strategy: Headless Browser Rendering (SSR) vs. Static HTML Ingestion

### Option A: Static HTML Fetching (Raw GET Requests)
* **Pros**:
  * Highly performant. Requires minimal CPU cycles and memory. Can run thousands of fetches per second per server node.
* **Cons**:
  * **Incomplete Crawl**: Cannot crawl modern client-side rendered sites (React/Angular). The crawler misses links generated dynamically by JavaScript.

### Option B: Headless Browser Rendering (Puppeteer / Playwright) (Selected Hybrid)
* **Pros**:
  * Fully executes JavaScript, ensuring high crawl coverage across the modern web.
* **Cons**:
  * **High Resource Consumption**: Running headless browser instances requires massive CPU and memory allocations, reducing the pages-per-second throughput per node.
  * **Mitigation**: We implement a hybrid crawler. The fetcher first checks the initial response headers. If no script tags are present or it's a known static site, it bypasses the headless browser.

---

## 4. Bloom Filter False Positives Trade-off

* **Decision**: We use Bloom Filters to verify if a URL has already been seen.
* **The Trade-off**: Bloom filters never yield false negatives (if it says "not seen", it is guaranteed). However, they have a configurable false positive rate (e.g. 0.1%).
* **Consequence**: A false positive means the crawler thinks it has seen a new URL when it hasn't, causing that link to be skipped.
* **Why it is Acceptable**: Wasting a tiny fraction of URLs is a negligible loss compared to the massive memory savings. A Bloom filter holding 1 billion elements takes only ~1.2 GB of RAM, whereas a Hash Set would consume over 30 GB.
