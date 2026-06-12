# Tinder System Design Specification

This document details the system design, capacity planning, and architectural topology for a real-time location-based matchmaking application (like Tinder), optimized for dynamic geospatial sharding, bidirectional WebSockets, and decoupled match loop queues.

---

## 1. Core Requirements & Scale Estimations

### Functional Requirements
* **Profile & Media Ingestion**: Users can create profiles including bios, preferences, and photos.
* **Discovery Feed**: Deliver localized profiles within a maximum search radius of 100 miles, filtered by age and gender preferences.
* **Match Loop**: Support swipe right (like) and swipe left (skip) actions, generating instant mutual match notifications.
* **Passport Mode**: Users can virtually change their location coordinates to view discovery decks in other regions.

### Non-Functional Requirements
* **High Horizontal Scale**: Support millions of active users and billions of daily swipes.
* **Ultra-Low Latency**: Rendering recommendation card decks must execute under $150\text{ ms}$.
* **Real-time Synchronization**: Mutual match alerts must trigger immediately on both active devices.

### Scale Capacity Calculations

We base our math on **50 Million Daily Active Users (DAU)**.

#### Throughput & Swipe Volume
* Assuming an average user makes **30 swipes per day**:
  $$\text{Daily Swipes} = 50,000,000 \times 30 = 1,500,000,000\text{ swipes/day}$$
* **Average Swipe Write QPS**:
  $$\text{Average Swipe QPS} = \frac{1,500,000,000}{86,400\text{ seconds}} \approx 17,361\text{ writes/second}$$
* **Peak Swipe QPS** (at peak hours, assuming a $3\times$ multiplier):
  $$\text{Peak Swipe QPS} = 17,361 \times 3 \approx 52,000\text{ writes/second}$$

#### Read Volume (Recommendation Requests)
* Assuming users request a new card deck of 20 profiles 15 times daily:
  $$\text{Daily Read Requests} = 50,000,000 \times 15 = 750,000,000\text{ reads/day}$$
* **Average Recommendation Read QPS**:
  $$\text{Average Read QPS} = \frac{750,000,000}{86,400\text{ seconds}} \approx 8,680\text{ reads/second}$$

---

## 2. System Architecture Topology

The diagram below details the Ingestion Path, Discovery Flow, real-time WebSocket Match Loop, and the Passport Mode update flow.

![High-Level System Architecture](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Tinder:%20System%20Design/architecture-diagram.png)

### Playbook Mindmap

For a conceptual overview of the component dependencies and S2 strategies, see the Playbook Mindmap:

![System Design Mindmap](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Tinder:%20System%20Design/mindmap.jpg)

---

## 3. Data Schema & Relationships

The system uses a combination of DynamoDB (for user profiles), Elasticsearch (for geospatial sharded discovery queries), and PostgreSQL/Cassandra (for swiping logs and matches).

![Database ER Diagram](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Tinder:%20System%20Design/database-schema.png)

* **Users**: Profile credentials and preference parameters.
* **User_Locations**: Tracks coordinates and S2 cell ID strings.
* **Swipes**: Records swipes (Likes/Skips) for matching validation.
* **Matches**: Tracks mutual matches to establish communication channels.

---

## 4. Playbook Modules

For deep technical details on specific aspects of the design, navigate to the following documents:

1. [API Design Matrix](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Tinder:%20System%20Design/api-design.md) - Specifying REST contracts, WebSocket swipe events, and Passport location JSON formats.
2. [S2 Geometry & Match Loop Strategy](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Tinder:%20System%20Design/scaling-notes.md) - In-depth breakdown of Hilbert curves, dynamic hot-spot cells, and in-memory Redis match queues.
3. [Bottleneck & Tradeoff Analysis](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Tinder:%20System%20Design/tradeoffs.md) - Evaluating S2 Cells vs. Geohashes, DynamoDB vs. SQL profile metadata, and async Kafka queues.
