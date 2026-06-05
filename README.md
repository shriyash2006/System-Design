# Distributed Systems Architecture Playbook

Welcome to the Distributed Systems Architecture Playbook. This repository serves as a centralized, production-grade tracking dashboard and repository for engineering specifications, architectural designs, and system topologies.

## Architecture Design Index

Use the table below to track the status, complexity, and locations of different architectural specifications.

| # | Design Topic | Subsystem / Category | Complexity | Status | Architectural Specification |
|---| :--- | :--- | :---: | :---: | :--- |
| **01** | **TinyURL (Distributed URL Shortener)** | High-Level Design / Core Services | Medium | 🟢 Complete | [01-tinyurl/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/01-tinyurl/README.md) |
| **02** | **Twitter / Newsfeed (Social Network)** | High-Level Design / Feed Systems | Medium-Hard | 🟢 Complete | [Twitter / Newsfeed: System Design/README.md](file:///Users/shriyashsahu/.gemini/antigravity/scratch/System-Design/Twitter%20/%20Newsfeed:%20System%20Design/README.md) |

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
└── Twitter / Newsfeed: System Design/  # Twitter / Newsfeed Case Study
    ├── README.md                       # Core entrypoint (Overview, Capacity, Stack, Key Learnings)
    ├── mindmap.jpg                     # High-fidelity design mindmap
    ├── architecture-diagram.png        # System architecture topology diagram
    ├── database-schema.png             # DB schema ER diagram
    ├── api-design.md                   # REST API endpoints design contracts
    ├── scaling-notes.md                # hybrid push/pull fan-out scaling strategy
    └── tradeoffs.md                    # push/pull models, consistency tradeoffs
```

> [!NOTE]
> All designs in this playbook follow a rigorous evaluation methodology, including functional/non-functional requirements analysis, back-of-the-envelope calculations, structural data flow diagrams, coordinator node algorithms, storage engine trade-offs, and critical system safeguards.
