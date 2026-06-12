# Hotel Booking System Design Specification

This document details the system design, capacity planning, and architectural topology for a Hotel Reservation System (such as Airbnb, Booking.com, or Expedia), optimized for strict transactional consistency to prevent double bookings under high concurrency.

---

## 1. Core Requirements & Scale Estimations

### Functional Requirements
* **Hotel Management**: Hotel administrators can add/remove hotels, rooms, and update pricing or inventory metrics.
* **Search & Book**: Customers can search for lodging via destination, check-in/checkout dates, guest counts, and filters, then reserve or cancel rooms.
* **Notifications**: Users receive real-time confirmations via email or push notifications.

### Non-Functional Requirements
* **Strict Consistency**: No double bookings are allowed. The system must use strong transaction isolation during booking checkouts.
* **Low Latency on Search**: Lodging searches must render under $100\text{ ms}$.
* **High Availability**: Booking services must remain resilient even if external notification services fail.

### Scale Capacity Calculations

We base our math on **50 Million Daily Active Users (DAU)**.

#### Throughput & Query Volume
* **Hotel Admin updates**:
  $$\text{Admin updates/day} = 1,000 \implies \text{Write QPS} = \frac{1,000}{86,400} \approx 0.0116\text{ writes/second}$$
* **Review Submission writes** (2 reviews/user/month):
  $$\text{Write QPS} = \frac{100,000,000\text{ reviews}}{2,592,000\text{ seconds/month}} \approx 38.5\text{ writes/second}$$
* **Hotel Search reads** (20 searches/user/month):
  $$\text{Read QPS} = \frac{1,000,000,000\text{ searches}}{2,592,000\text{ seconds/month}} \approx 385,800\text{ reads/second}$$

#### Storage Footprint
* Average hotel profile with media: $5,000\text{ KB}$.
* Average text review: $2\text{ KB}$.
* **Annual Storage accumulation**:
  $$\text{Hotel Metadata/year} \approx 82.5\text{ GB/year}$$
  $$\text{Reviews/year} = 100,000,000 \times 12\text{ months} \times 2\text{ KB} \approx 2.4\text{ TB/year}$$
  $$\text{Total Storage Requirement} \approx 2.6\text{ TB/year}$$

---

## 2. System Architecture Topology

The diagram below details the Admin Ingestion Path, Discovery Search Path, and Concurrency-Safe Reservation Path.

![High-Level System Architecture](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Hotel%20Booking:%20System%20Design/architecture-diagram.png)

### Playbook Mindmap

For a conceptual overview of the component dependencies and sharding strategies, see the Playbook Mindmap:

![System Design Mindmap](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Hotel%20Booking:%20System%20Design/mindmap.jpg)

---

## 3. Data Schema & Relationships

The system uses a sharded PostgreSQL relational database to manage inventory tables, and Cassandra/DynamoDB to manage review logs.

![Database ER Diagram](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Hotel%20Booking:%20System%20Design/database-schema.png)

* **Hotels**: Lodging properties catalog.
* **Room_Types**: Configurations of spaces (e.g., Deluxe Suite, Standard King).
* **Room_Pricing**: Dynamic pricing values per day.
* **Room_Inventory**: Tracks available room count per type per date.
* **Reservations**: Transactional bridge linking guests to reserved room type date blocks.

---

## 4. Playbook Modules

For deep technical details on specific aspects of the design, navigate to the following documents:

1. [API Design Matrix](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Hotel%20Booking:%20System%20Design/api-design.md) - Specifying REST contracts, catalog uploads, and reservation requests with idempotency keys.
2. [Concurrency & Sharding Strategy](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Hotel%20Booking:%20System%20Design/scaling-notes.md) - Deep dive on local single-node transactions, Serializable Isolation, Pessimistic Row Locking, and Redis inventory cache layers.
3. [Bottleneck & Tradeoff Analysis](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Hotel%20Booking:%20System%20Design/tradeoffs.md) - Evaluating Pessimistic vs. Optimistic locking, Serializable vs. Read Committed isolation, and sharding keys.
