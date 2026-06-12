# Tradeoff Matrix: Key Design Decisions

Geospatial platform design requires balancing index complexity, write speeds, and regional query load distribution. Below is the tradeoff evaluation for the Proximity Service.

---

## 1. Spatial Indexing: Geohashing vs. Quadtrees

| Dimension | Geohashing (Selected) | Quadtrees |
| :--- | :--- | :--- |
| **Data Structure** | Converts 2D coordinates into a 1D alphanumeric string representing grid quadrants. | Tree structure where each node splits into four child quadrants recursively based on node density. |
| **Index Storage** | Saved as a standard string column in standard databases with standard B-tree index layouts. | Must be loaded and maintained in-memory or in specialized database extensions. |
| **Grid Boundaries** | **Static**: Grid cell boundaries are fixed regardless of the count of business listings. | **Dynamic**: Densely populated areas have small quadrants; empty areas have large quadrants. |
| **Boundary Problems** | A point near a grid edge may have its closest neighbor in an adjacent cell. Requires querying 8 neighboring cells. | Solves grid edge issues via tree traversal, but query overhead increases as tree depth grows. |
| **Ease of Sharding** | Simple. Can easily slice data horizontally by the geohash string prefix. | Hard. Partitioning a dynamic tree structure across multiple machines is mathematically complex. |

* **Architectural Decision**: We select **Geohashing** due to its ease of database index representation, simple string prefix matching, and straightforward sharding.

---

## 2. Core Location Engine: Postgres/PostGIS vs. NoSQL Spatial Indices

We evaluated storing location metadata in a relational database with PostGIS against NoSQL engines (e.g., Cassandra or Redis Geospatial):

* **PostgreSQL + PostGIS (Selected)**:
  * **Pros**: Provides standard ACID transactions for core business listings. Native spatial indexing (R-tree/GIST indexes) and spatial SQL query capabilities (e.g., `ST_Distance`, `ST_Contains`) simplify exact distance calculations.
  * **Cons**: Relational scaling (replication, sharding) introduces operational complexity compared to NoSQL.
* **NoSQL (e.g., Redis GEORADIUS or Cassandra Geohashes)**:
  * **Pros**: High QPS write and read speeds.
  * **Cons**: Lack of relational join support. Performing compound queries (e.g., "Find restaurants within 2km that have at least 4 stars AND have reviews containing 'sushi'") requires fetching IDs and joining datasets at the application layer, which increases network overhead.

* **Architectural Decision**: We use **PostgreSQL + PostGIS** as our single source of truth for location metadata, paired with **Elasticsearch** to handle text and category filters and **Redis** to cache high-frequency search results.

---

## 3. Database Sharding: Attraction_ID vs. Geohash Prefix

To partition our location database across multiple nodes:

### Option A: Sharding by `Attraction_ID`
* **Pros**: Evenly distributes write and read operations across all database shards.
* **Cons**: Because location data is scattered across all shards, a geographic search query must be broadcast to every single database shard (scatter-gather model) to compile the results, which increases latency and network overhead.

### Option B: Sharding by `Geohash Prefix` (Selected)
* **Pros**: Slices shards based on geographic location (e.g., the first 3 characters of the geohash string, like `9q8` for Northern California). A user query is routed to exactly one or two shards, avoiding scatter-gather overhead.
* **Cons**: Creates hotspots. Shards covering dense urban areas (e.g., San Francisco, New York) will experience much higher traffic and storage requirements than shards covering rural areas.

* **Architectural Decision**: We shard by **Geohash Prefix** to keep query routing efficient. To mitigate hotspotting, we allocate larger server instances (or dedicate separate shards) to high-density geohash regions.
