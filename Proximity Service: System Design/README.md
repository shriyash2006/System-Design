# Proximity Service System Design Specification

This document details the system design, capacity planning, and architectural topology for a Proximity Service (such as Yelp, Google Maps, or TripAdvisor), optimized for fast, geographically scoped queries using coordinate indexing techniques and a decoupled read/write infrastructure.

---

## 1. Core Requirements & Scale Estimations

### Functional Requirements
* **Create Attraction**: Business owners can create points of interest (attractions, businesses, restaurants) with coordinates, categories, and media.
* **Geospatial Search**: Users can search for attractions based on coordinates, categories, a search radius, and review counts.
* **Add Reviews**: Users can submit star ratings and textual reviews for attractions.

### Non-Functional Requirements
* **High Scalability**: Support millions of attractions and active users worldwide.
* **High Availability**: Target $99.999\%$ uptime, accepting eventual consistency for reviews and business metadata updates.
* **Ultra-Low Latency**: Search results must render under $100\text{ ms}$.

### Scale Capacity Calculations

We base our math on **50 Million Daily Active Users (DAU)**.

#### Throughput & Query Volume
* **Business Creation writes**:
  $$\text{Business writes/day} = 1,000 \implies \text{Write QPS} = \frac{1,000}{86,400} \approx 0.0116\text{ writes/second}$$
* **Review Submission writes** (2 reviews/user/month):
  $$\text{Reviews/month} = 50,000,000 \times 2 = 100,000,000\text{ reviews/month}$$
  $$\text{Write QPS} = \frac{100,000,000}{2,592,000\text{ seconds/month}} \approx 38.5\text{ writes/second}$$
* **Geospatial Search reads** (20 searches/user/month):
  $$\text{Searches/month} = 50,000,000 \times 20 = 1,000,000,000\text{ searches/month}$$
  $$\text{Read QPS} = \frac{1,000,000,000}{2,592,000\text{ seconds/month}} \approx 385,800\text{ reads/second}$$

#### Storage Footprint
* Average business metadata + media footprint: $5,000\text{ KB}$.
* Average text review footprint: $2\text{ KB}$.
* **Annual Storage accumulation**:
  $$\text{Business Storage/year} = 1,000\text{ new spots/day} \times 365\text{ days} \times 5,000\text{ KB} \approx 1.825\text{ TB/year}$$
  *(With optimized media compression and deduplication, this indexes down to roughly $82.5\text{ GB/year}$ of core relational metadata).*
  $$\text{Reviews Storage/year} = 100,000,000\text{ reviews/month} \times 12\text{ months} \times 2\text{ KB} \approx 2.4\text{ TB/year}$$
  $$\text{Total Storage Requirement} \approx 2.6\text{ TB/year}$$

---

## 2. System Architecture Topology

The diagram below maps the Attraction Write Path (Creation Flow), the Attraction Read Path (Search Flow), and the Review Ingestion Path.

![High-Level System Architecture](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Proximity%20Service:%20System%20Design/architecture-diagram.png)

### Playbook Mindmap

For a conceptual overview of the component dependencies and strategies, see the Playbook Mindmap:

![System Design Mindmap](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Proximity%20Service:%20System%20Design/mindmap.jpg)

---

## 3. Data Schema & Relationships

The system uses a combination of PostgreSQL (for ACID-compliant location metadata sharded by Geohash Prefix) and Apache Cassandra/DynamoDB (for high-volume append reviews).

![Database ER Diagram](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Proximity%20Service:%20System%20Design/database-schema.png)

* **Users**: Stores profile identifiers.
* **Attractions**: Records business details, precise coordinates, and geohash strings.
* **Media**: Stores business images linked to S3.
* **Reviews**: Records ratings and comments, sharded horizontally by attraction ID in the NoSQL layer.

---

## 4. Playbook Modules

For deep technical details on specific aspects of the design, navigate to the following documents:

1. [API Design Matrix](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Proximity%20Service:%20System%20Design/api-design.md) - Specifying REST contracts, media upload formats, and search queries.
2. [Geospatial Scaling & Ingestion Strategy](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Proximity%20Service:%20System%20Design/scaling-notes.md) - Covering geohash precision mechanics, bounding-box Elasticsearch queries, and PostGIS distance checks.
3. [Bottleneck & Tradeoff Analysis](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Proximity%20Service:%20System%20Design/tradeoffs.md) - Deep dive on Geohashing vs. Quadtrees, PostgreSQL vs. NoSQL spatial queries, and sharding options.
