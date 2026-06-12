# Distributed Systems Architecture Playbook

Welcome to the Distributed Systems Architecture Playbook. This repository serves as a centralized, production-grade tracking dashboard and repository for engineering specifications, architectural designs, and system topologies.

## Architecture Design Index

Use the table below to track the status, complexity, and locations of different architectural specifications.

| # | Design Topic | Subsystem / Category | Complexity | Status | Architectural Specification |
|---| :--- | :--- | :---: | :---: | :--- |
| **01** | **TinyURL (Distributed URL Shortener)** | High-Level Design / Core Services | Medium | 🟢 Complete | [01-tinyurl/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/01-tinyurl/README.md) |
| **02** | **Twitter / Newsfeed (Social Network)** | High-Level Design / Feed Systems | Medium-Hard | 🟢 Complete | [Twitter / Newsfeed: System Design/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Twitter%20/%20Newsfeed:%20System%20Design/README.md) |
| **03** | **Web Crawler (Search Ingestion)** | High-Level Design / Scraping Systems | Medium | 🟢 Complete | [Web Crawler: System Design/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Web%20Crawler:%20System%20Design/README.md) |
| **04** | **Dropbox / Google Drive (Cloud Storage)** | High-Level Design / File Systems | Medium-Hard | 🟢 Complete | [Dropbox / Google Drive: System Design/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Dropbox%20/%20Google%20Drive:%20System%20Design/README.md) |
| **05** | **Messaging App (WhatsApp/Discord)** | High-Level Design / Chat Protocols | Medium-Hard | 🟢 Complete | [Messaging App: System Design/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Messaging%20App:%20System%20Design/README.md) |
| **06** | **Instagram App (Scale Media & Feed)** | High-Level Design / Feed Systems | Medium-Hard | 🟢 Complete | [Instagram App: System Design/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Instagram%20App:%20System%20Design/README.md) |
| **07** | **Proximity Service (Yelp/Maps)** | High-Level Design / Location Systems | Medium-Hard | 🟢 Complete | [Proximity Service: System Design/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Proximity%20Service:%20System%20Design/README.md) |
| **08** | **Tinder (Real-time Matchmaking)** | High-Level Design / Matchmaking Systems | Medium-Hard | 🟢 Complete | [Tinder: System Design/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Tinder:%20System%20Design/README.md) |
| **09** | **Hotel Booking (Reservation System)** | High-Level Design / Booking Systems | Medium-Hard | 🟢 Complete | [Hotel Booking: System Design/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Hotel%20Booking:%20System%20Design/README.md) |





---

## Playbook Structure & Organization

```directory
.
├── README.md                           # Global playbook tracking dashboard
├── 01-tinyurl/                         # TinyURL Design Case Study
│   ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
│   ├── mindmap.jpg                     # High-fidelity design mindmap
│   ├── architecture-diagram.png        # System architecture topology diagram
│   ├── database-schema.png             # url_mappings table ER schema diagram
│   ├── api-design.md                   # REST API endpoints design contracts
│   ├── scaling-notes.md                # range-allocation, caching, FPE algorithms
│   └── tradeoffs.md                    # DB sharding, Zookeeper sequence gaps tradeoffs
├── Twitter / Newsfeed: System Design/  # Twitter / Newsfeed Case Study
│   ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
│   ├── mindmap.jpg                     # High-fidelity design mindmap
│   ├── architecture-diagram.png        # System architecture topology diagram
│   ├── database-schema.png             # DB schema ER diagram
│   ├── api-design.md                   # REST API endpoints design contracts
│   ├── scaling-notes.md                # hybrid push/pull fan-out scaling strategy
│   └── tradeoffs.md                    # push/pull models, consistency tradeoffs
├── Web Crawler: System Design/         # Web Crawler Case Study
│   ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
│   ├── mindmap.jpg                     # High-fidelity design mindmap
│   ├── architecture-diagram.png        # System architecture topology diagram
│   ├── database-schema.png             # DB schema ER diagram
│   ├── api-design.md                   # REST API endpoints design contracts
│   ├── scaling-notes.md                # Frontier scheduling, Bloom filters
│   └── tradeoffs.md                    # DB selection, hash deduplication tradeoffs
├── Dropbox / Google Drive: System Design/ # Dropbox / Google Drive Case Study
│   ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
│   ├── mindmap.jpg                     # High-fidelity design mindmap
│   ├── architecture-diagram.png        # System architecture topology diagram
│   ├── database-schema.png             # DB schema ER diagram
│   ├── api-design.md                   # REST API endpoints design contracts
│   ├── scaling-notes.md                # chunking pipelines, long polling
│   └── tradeoffs.md                    # SQL sharding vs NoSQL, WebSockets vs Long polling
└── Messaging App: System Design/       # Messaging App Case Study
    ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
    ├── mindmap.jpg                     # High-fidelity design mindmap
    ├── architecture-diagram.png        # System architecture topology diagram
    ├── database-schema.png             # DB schema ER diagram
    ├── api-design.md                   # WebSocket & API design contracts
    ├── scaling-notes.md                # stateful sockets, Snowflake ID, presence heartbeats
    └── tradeoffs.md                    # WebSockets vs Long polling, storage engine tradeoffs
├── Instagram App: System Design/       # Instagram App Case Study
│   ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
│   ├── mindmap.jpg                     # High-fidelity design mindmap
│   ├── architecture-diagram.png        # System architecture topology diagram
│   ├── database-schema.png             # DB schema ER diagram
│   ├── api-design.md                   # REST API endpoints design contracts
│   ├── scaling-notes.md                # S3 chunking, Kafka ingestion, hybrid push/pull fan-out
│   └── tradeoffs.md                    # Push/Pull model, DB sharding, nested comments
├── Proximity Service: System Design/   # Proximity Service Case Study
│   ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
│   ├── mindmap.jpg                     # High-fidelity design mindmap
│   ├── architecture-diagram.png        # System architecture topology diagram
│   ├── database-schema.png             # DB schema ER diagram
│   ├── api-design.md                   # REST API endpoints design contracts
│   ├── scaling-notes.md                # geohashing prefixes, PostGIS coordinate query setups
│   └── tradeoffs.md                    # Geohashing vs Quadtrees, PostGIS vs NoSQL, sharding plans
├── Tinder: System Design/              # Tinder Case Study
│   ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
│   ├── mindmap.jpg                     # High-fidelity design mindmap
│   ├── architecture-diagram.png        # System architecture topology diagram
│   ├── database-schema.png             # DB schema ER diagram
│   ├── api-design.md                   # WebSocket & API design contracts
│   ├── scaling-notes.md                # S2 geometry cell hashing, WebSocket push notifications
│   └── tradeoffs.md                    # S2 Cells vs Geohash, NoSQL vs SQL profiles, match loops
└── Hotel Booking: System Design/       # Hotel Booking Case Study
    ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
    ├── mindmap.jpg                     # High-fidelity design mindmap
    ├── architecture-diagram.png        # System architecture topology diagram
    ├── database-schema.png             # DB schema ER diagram
    ├── api-design.md                   # REST API endpoints design contracts
    ├── scaling-notes.md                # hotel sharding, pessimistic locks, serializable transactions
    └── tradeoffs.md                    # Pessimistic vs Optimistic locking, isolation types, sharding
```

> [!NOTE]
> All designs in this playbook follow a rigorous evaluation methodology, including functional/non-functional requirements analysis, back-of-the-envelope calculations, structural data flow diagrams, coordinator node algorithms, storage engine trade-offs, and critical system safeguards.
