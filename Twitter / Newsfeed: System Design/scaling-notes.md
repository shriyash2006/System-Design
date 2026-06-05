# Scaling Strategy: Twitter / Newsfeed at Scale

Scaling a microblogging network like Twitter requires managing highly asymmetric read/write patterns: 4,600 write QPS versus 23,000 read QPS, amplified by massive celebrity hotkey fan-outs. This document outlines the core scaling architecture.

---

## 1. The Hybrid Push/Pull Fan-Out Model

The primary bottleneck in feed generation is the **Fan-out** process: delivering a published tweet to the home timelines of all followers. We implement a **Hybrid Fan-out Model** to scale this operation.

```
                         [ New Tweet Published ]
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                 [ Regular User ]       [ Celebrity / VIP ]
                         │                     │
                (Push / Write-Path)       (Pull / Read-Path)
                         │                     │
                  Pushed to Kafka              │
                         │                     │
                 Fan-Out Workers               │
                         │                     │
                Writes to Followers'           │
                Redis Timeline Caches          │
                         │                     │
                         ▼                     ▼
             [ Followers Request Feed ] ──────► Merge Celebrity
             (Read pre-generated cache)         Tweets dynamically
```

### Write Path: Push Model (Fan-Out on Write)
1. **Ingestion**: When a standard user publishes a tweet, the Tweet Service writes it to the database and publishes a message to **Apache Kafka**.
2. **Workers**: Fan-out workers consume the message and query the User Graph Service (PostgreSQL) to retrieve the user's followers.
3. **Cache Insertion**: The workers insert the tweet ID directly into the Redis home timeline caches of all followers. Redis stores timelines as list structures:
   `LPUSH timeline:user_<follower_id> <tweet_id>`
   Lists are capped (e.g. max 800 items) to limit memory growth.
4. **Result**: Reading a timeline is extremely fast ($\mathcal{O}(1)$), since it only requires querying the pre-computed Redis list.

### Read Path: Pull Model (Fan-Out on Read)
* **The Celebrity Problem**: When a celebrity with 80M followers publishes a tweet, using the push model would require fan-out workers to execute 80M Redis cache writes. This causes write queues to back up and degrades system performance.
* **Celebrity Rule**: If a user is flagged as a celebrity (e.g. >50,000 followers), the push pipeline is bypassed. The celebrity's tweet is written only to their own user timeline.
* **Read-Time Merging**: When a follower requests their feed, the Timeline Service pulls the pre-generated home timeline from Redis and dynamically queries the user timelines of the celebrities they follow. The service merges these tweets and returns the sorted result.

---

## 2. Multi-Layer Caching Architecture

Caching is critical to achieving sub-200ms timeline reads:

* **Redis Timeline Caches**: Active user timelines (users who logged in within the last 3 days) are cached in Redis. Inactive timelines are evicted and regenerated from databases only on demand.
* **User Graph Cache**: To optimize follower queries, the social graph is cached in Redis using Sets.
* **Media CDN**: Images and videos are stored in Amazon S3 and cached globally at edge nodes via CDNs (Cloudflare, Fastly).

---

## 3. Database Sharding & Partitioning

We segment data across storage engines based on usage patterns:

* **Cassandra (Tweets)**: Cassandra stores the raw tweet metadata. It is partitioned by `user_id`, allowing all tweets from a single user to sit on the same physical node for fast user-timeline lookups.
* **PostgreSQL (Followers & Relations)**: To handle relational graph operations, we shard PostgreSQL databases on `user_id`. Shards are split using consistent hashing to balance load across nodes.

---

## 4. Real-Time Search Indexing

To support search capabilities:
* New tweets published to Kafka are consumed by search workers in parallel.
* The workers push the text tokenized data to an **Elasticsearch** cluster.
* Users can query the Elasticsearch cluster directly to search tweets by keyword or hashtag.
