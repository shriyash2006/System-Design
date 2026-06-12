# Tradeoff Matrix: Key Design Decisions

Hotel reservation systems must choose between consistency guarantees and horizontal scale. Below is the tradeoff evaluation for the Hotel Booking System.

---

## 1. Concurrency Control: Pessimistic vs. Optimistic Locking

To prevent double bookings of a limited room inventory, we evaluated row-level lock allocation strategies:

| Dimension | Pessimistic Locking (`SELECT FOR UPDATE`) [Selected] | Optimistic Locking (Version Check) |
| :--- | :--- | :--- |
| **Mechanics** | Holds a database lock on the matching inventory rows for the entire transaction block. | Verifies the record version matches during commit. If modified, transaction fails. |
| **High Contention Behavior** | **Excellent**: Requests wait sequentially in queue. No transactions fail due to collision, they just wait for lock release. | **Poor**: High collision rate causes transaction aborts, forcing user retries and checkout failures. |
| **Database Resource Overhead** | **High**: Database connections are held open waiting for locks, increasing thread count. | **Low**: No locks are held, maximizing database thread availability. |
| **User Experience** | Users experience a slight latency pause, but get a confirmed booking if inventory is available. | Users fill in details, pay, and experience random transaction failures on commit due to concurrent edits. |

* **Architectural Decision**: We select **Pessimistic Row-Level Locking** to guarantee that once a user clicks "book", they do not suffer rollback abort failures during high-demand booking windows.

---

## 2. Database Isolation: Serializable vs. Read Committed

* **Serializable Isolation (Selected)**:
  * **Pros**: Eliminates all read anomalies (dirty reads, non-repeatable reads, phantom reads, write skew). The database engine validates that concurrent transactions behave as if executed sequentially.
  * **Cons**: Highest database CPU overhead and abort rates for concurrent conflicting queries.
* **Read Committed Isolation**:
  * **Pros**: High database read throughput.
  * **Cons**: Vulnerable to race conditions where two concurrent transactions read the same available room inventory ($1$ room left) and both write decrements, leading to double booking unless application-level lock trees are implemented.

* **Architectural Decision**: We choose **Serializable Isolation** paired with row locking to ensure absolute consistency for room inventory transitions.

---

## 3. Database Sharding Key: Hotel_ID vs. User_ID

### Option A: Sharding by `Hotel_ID` (Selected)
* **Pros**: Co-locates all tables for a single hotel (`Room_Types`, `Room_Pricing`, `Room_Inventory`, `Reservations`) on a single database node. Allows transactions to run as local database operations on a single node, bypassing distributed 2-Phase Commit (2PC) latencies.
* **Cons**: Hotspotting. Popular hotels (e.g., major resorts during holiday seasons) will overload their specific database node shard, while shards containing small hotels remain underutilized.

### Option B: Sharding by `User_ID`
* **Pros**: Evenly distributes database write and read operations across all nodes.
* **Cons**: Booking a room requires a distributed transaction to update `room_inventory` on the Hotel shard and insert the reservation record on the User shard, introducing 2PC network latency.

* **Architectural Decision**: We shard by **Hotel_ID** to enable high-throughput local transactions. We address hotspotting by using larger server instances for high-density hotel regions.
