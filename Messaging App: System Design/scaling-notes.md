# Scaling Strategy: Stateful Duplexing, Snowflake ID, and Presence

Scaling a real-time messaging platform to support 100 million Daily Active Users (DAU) and 58,000 write QPS requires decoupling HTTP networks from stateful sockets, managing server discovery, and indexing presence heartbeats efficiently.

---

## 1. Stateful vs. Stateless Infrastructure Decoupling

The chat platform splits its workload into two decoupled operational tiers:

### Stateless Tier (HTTP Engines)
* Handles authentication, user profile management, configurations, and media uploads.
* Decoupled using an API Gateway and Load Balancer. It scales horizontally.

### Stateful Tier (WebSocket Gateway)
* Real-time packet routing requires persistent, bidirectional WebSocket TCP connections.
* Once a device connects, its socket stays open on an explicit chat server instance (e.g., Chat Server 1). That instance keeps the socket handle in its memory stack, making it a stateful tier.

---

## 2. Service Discovery & Routing (Zookeeper + Redis Mapping)

Because chat server allocations are stateful, a client needs to map to a specific server instance, and other nodes must know where that client is connected.

* **Zookeeper Discovery**: **Apache Zookeeper** acts as the cluster coordinator, tracking the health, availability, and resource utilization of all Chat Server instances. When a client performs a handshake request, the load balancer queries Zookeeper to select the least-loaded chat server.
* **Client-to-Server Mapping Table**: A global, highly scalable key-value database (**Redis Cluster** or DynamoDB) stores active sessions. It maps client identifiers to their active chat servers:
  $$\text{Key: } \mathtt{client\_id\_A} \implies \text{Value: } \mathtt{chat\_server\_1}$$
  This mapping lets the system locate and route messages to online users in $\mathcal{O}(1)$ time.

---

## 3. Snowflake Unique ID Generation

To order messages globally across distributed chat clusters without database bottlenecks, the system uses a **Snowflake ID Generation Service** (based on Twitter's Snowflake algorithm):

```
+-------------------+-------------------+-------------------+
| Timestamp (41b)   | Node ID (10b)     | Sequence (12b)    |
+-------------------+-------------------+-------------------+
```

* **41 bits**: Timestamp in milliseconds (gives 69 years of lifespan).
* **10 bits**: Node identifier (supports 1024 unique generator nodes).
* **12 bits**: Local sequence counter (rolls over at 4096 messages per millisecond).
* **Benefit**: Generates time-sorted, globally unique 64-bit integer IDs without requiring database locks or cross-datacenter synchronization.

---

## 4. Real-Time Packet Delivery Flow

When Client A sends a message to Client B, the system handles the delivery route dynamically:

```
[ Client A ] ──► (WebSocket) ──► [ Chat Server 1 ] ──► [ Snowflake ID Service ]
                                       │
                              (Commit Cassandra)
                                       │
                             Query Mapping Table
                                       │
                      ┌────────────────┴────────────────┐
                      ▼ (B is Online)                   ▼ (B is Offline)
               [ Chat Server 2 ]                [ Kafka Buffer Queue ]
                      │                                 │
                 (WebSocket)                            ▼
                  Client B                     [ Notification Service ]
                                                        │
                                                        ▼
                                                    [ FCM / APNs ]
```

1. **Ingest**: Client A sends a packet over WebSocket to **Chat Server 1**.
2. **Persistence**: Chat Server 1 requests a Snowflake ID, appends it to the message metadata, and commits it asynchronously to the **Cassandra Cluster**.
3. **Route Lookup**: Chat Server 1 queries the Redis Mapping Table for Client B's server location.
4. **Online Routing**: If Client B is mapped to **Chat Server 2**, Chat Server 1 forwards the packet directly to Chat Server 2, which transmits it over Client B's active WebSocket connection.
5. **Offline Routing**: If Client B is offline, the message is routed to an **Apache Kafka Queue**. The Notification Service consumes this event and triggers a mobile push notification using Firebase Cloud Messaging (FCM) or Apple Push Notification Service (APNs).

---

## 5. Presence Heartbeat Engine

Tracking user status via connect/disconnect hooks creates heavy network noise and fails to detect sudden connection drops (e.g. entering a tunnel).

* **Heartbeat Mechanism**: Clients send a lightweight UDP packet (heartbeat) to a **Presence Server** every 5 seconds.
* **Presence Store**: The Presence Server records this heartbeat in a Redis cache with a 30-second TTL:
  `SETEX presence:user_123 30 "online"`
* **Status Eviction**: If a user's heartbeat fails to arrive within 30 seconds, Redis evicts the key. The Presence Server detects this event, updates the database, and broadcasts a status update to the active chat queues of the user's contacts.
