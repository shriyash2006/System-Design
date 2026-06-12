# Scaling Notes: Concurrency-Safe Reservations & Sharding

To prevent double bookings while servicing millions of searches and transactions, the Hotel Booking System partitions database layers and enforces strict transactional isolation boundaries.

---

## 1. Concurrency-Safe Reservation Flow

Double booking happens when two users attempt to reserve the final room type on overlapping dates concurrently, and the system executes both write operations before validating inventory limits.

```
[Customer Client] ──> [Reservation Service]
                            │
              (Step 1: Check Idempotency Key in Redis)
                            │
              (Step 2: Start Postgres Local Transaction)
              (Step 3: Execute Row-Level Lock 'FOR UPDATE')
                            │
        (Step 4: Check room_inventory for date range)
                            │
         ┌──────────────────┴──────────────────┐
         ▼ (inventory available)               ▼ (inventory = 0)
    [Decrement inventory]               [Rollback Transaction]
   [Insert Reservation row]            [Return 'Sold Out' error]
         │
    (Step 5: Commit & release lock)
```

### Isolation & Locking Mechanics
1. **Idempotency Check**: The client passes a unique `X-Idempotency-Key` (UUIDv4) in the header. The Reservation Service queries Redis. If the key exists, the request returns the cached result of the previous booking attempt, preventing duplicate charge inserts.
2. **Serializable Isolation**: The database session initiates under the **Serializable Isolation Level**. This ensures concurrent transactions yield the same output as if they were executed sequentially.
3. **Pessimistic Row-Level Locking**: The service locks database records for the requested dates using `SELECT ... FOR UPDATE` on the `room_inventory` table:
   ```sql
   BEGIN TRANSACTION;
   SELECT available_rooms 
   FROM room_inventory 
   WHERE room_type_id = :room_type_id 
     AND date BETWEEN :check_in AND :check_out 
   FOR UPDATE;
   ```
4. **Validation & Atomic Decr**: The application reads the locked inventory values.
   * If `available_rooms > 0` for **every** day in the range:
     ```sql
     UPDATE room_inventory 
     SET available_rooms = available_rooms - 1 
     WHERE room_type_id = :room_type_id 
       AND date BETWEEN :check_in AND :check_out;
     
     INSERT INTO reservations (...) VALUES (...);
     COMMIT;
     ```
   * If any day in the range returns `available_rooms = 0`, the transaction rolls back, releasing the locks and returning a "Sold Out" error.

---

## 2. Sharding Strategy: Sharding by `Hotel_ID`

To achieve high throughput, we avoid **Distributed Locking** (e.g., ZooKeeper lock trees or 2-Phase Commit protocols) which introduce network latency and block threads.

* **Shard Partition Key**: The relational database is sharded using `Hotel_ID`.
* **Co-located Tables**: All tables related to a specific hotel (`Room_Types`, `Room_Pricing`, `Room_Inventory`, `Reservations`) sit on the **same physical database instance**.
* **ACID Guarantee**: Transactions are executed as high-speed local database operations on a single node, bypassing distributed transaction latency.

---

## 3. Asynchronous Write & Sync Read Pipelines

### Admin Write Path
1. An administrator edits hotel profiles or rates.
2. The Admin Service streams media to **Amazon S3** and publishes change events to **Apache Kafka**, partitioned by `Hotel_ID` to guarantee strict message ordering.
3. A background worker commits change logs to the primary sharded **PostgreSQL Master**.
4. The worker updates the **Elasticsearch** search indexes. This maintains eventual consistency while keeping the booking path responsive.

### Discovery Read Path
1. User searches are routed to **Elasticsearch** for faceted and full-text keyword queries.
2. Matching **Room_Type_IDs** are checked against a **Redis Cache** holding pre-computed inventory maps.
3. On a cache miss, the service queries PostgreSQL Read Replicas to verify availability.

---

## 4. Payment Pending State & Fault Isolation

* **Payment TTL State**: When a reservation is created, the system sets the status to `PENDING_PAYMENT` and decrements inventory. A Redis scheduler monitors a **10-minute TTL**. If the payment webhook (from Stripe/PayPal) is not received within this window, the reservation is canceled and the inventory is incremented back.
* **Cascading Fault Isolation (Circuit Breakers)**: Services like the Notification Service are decoupled using Kafka. A circuit breaker (e.g., Resilience4j) is placed at the checkout Gateway to prevent dependency failures from blocking the booking path.
