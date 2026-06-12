# Tradeoff Matrix: Key Design Decisions

Matchmaking networks must resolve high-density spatial indexing alongside write-heavy matching logic under low latency constraints. Below is the tradeoff matrix for the Tinder system design.

---

## 1. Geospatial Partitioning: Google S2 vs. Geohashing

| Dimension | Google S2 Geometry (Selected) | Geohashing |
| :--- | :--- | :--- |
| **Space Mapping** | Projects Earth's sphere onto a cube, dividing it hierarchically into S2 Cells. | Slices Earth's coordinates into a flat 2D grid of alphanumeric character cells. |
| **Spatial Locality** | **High**: The Hilbert Space Curve ensures coordinates near each other physically map to similar 1D cell strings. | **Medium**: Standard geohash strings exhibit grid edge discontinuities, where adjacent coordinates map to completely different strings. |
| **Grid Cell Sizing** | **Dynamic**: Levels (0-30) allow small cells in urban hubs and large cells in rural areas. | **Static**: Grid size is fixed by string length, requiring ad-hoc logic to balance sparse/dense regions. |
| **Complexity** | **High**: Requires S2 library integrations and spherical math conversions. | **Low**: Simple string prefix queries (`LIKE '9q8yyz%'`) supported out-of-the-box by any DB. |

* **Architectural Decision**: We select **Google S2** to leverage dynamic cell sizing. This prevents "hot sharding" in high-density urban areas.

---

## 2. Profile Storage: NoSQL (DynamoDB) vs. Relational (PostgreSQL)

We evaluated where to store main user profile data and preferences:

* **NoSQL (Amazon DynamoDB) (Selected)**:
  * **Pros**: Highly scalable with single-digit millisecond latency. Flexible schema makes it easy to add profile fields (e.g., interests, prompts) without database migration locks.
  * **Cons**: Lack of complex relational joins.
* **Relational (PostgreSQL)**:
  * **Pros**: Supports ACID transactional integrity.
  * **Cons**: Scaling horizontally requires sharding on `user_id`. Updating database schemas across millions of user accounts requires complex online schema migrations.

* **Architectural Decision**: We choose **DynamoDB** to store profile metadata to support high scale and schema flexibility. We offload geospatial queries to **Elasticsearch Geo Shards**.

---

## 3. Match Evaluation: Async Queue Ingestion vs. Sync In-Thread Matching

We evaluated how to process swipe events and match notifications:

* **Async Queue Ingestion (Kafka + Redis Likes Cache) (Selected)**:
  * **Pros**: A swipe event is immediately acknowledged. The heavy logic (writing to Swipe DB, checking Redis sets for mutual likes, and broadcasting notifications) is handled by background workers. Prevents database lock contention during high traffic peaks.
  * **Cons**: Introduces eventual consistency. A mutual match notification may take a few hundred milliseconds to arrive on the client.
* **Sync In-Thread Matching**:
  * **Pros**: Guarantees immediate consistency.
  * **Cons**: Blocks client connection thread while querying and updating database records, which degrades gateway performance during traffic bursts.

* **Architectural Decision**: We choose the **Async Queue Ingestion** path via **Kafka** and **Redis** to ensure high availability and prevent write bottlenecks.
