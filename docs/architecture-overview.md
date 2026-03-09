# Distributed Architecture Overview

## Introduction

The framework supports distributed architecture component deployment, management, and monitoring. It enables horizontal scaling, centralized data synchronization, and disaster recovery for production environments.

## High-Level Architecture

```
                        ┌─────────────────────────────────────────────┐
                        │              Load Balancer                  │
                        └────────────────┬────────────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
              ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
              │  Parent    │       │  Child     │       │  Child     │
              │  App Node  │       │  App Node  │       │  App Node  │
              │            │       │            │       │            │
              │ ┌────────┐ │       │            │       │            │
              │ │TaskEng.│ │       │            │       │            │
              │ └────────┘ │       │            │       │            │
              │ ┌────────┐ │ gRPC  │            │ gRPC  │            │
              │ │RocksDB │◄├───────┤            ├───────┤            │
              │ │ Cache  │ │       │            │       │            │
              │ └────────┘ │       │            │       │            │
              └─────┬──────┘       └─────┬──────┘       └─────┬──────┘
                    │                    │                    │
                    └────────────────────┼────────────────────┘
                                         │
                              ┌──────────▼──────────┐
                              │   PostgreSQL (Write) │
                              │   ┌───────────────┐  │
                              │   │   Triggers     │  │
                              │   └───────┬───────┘  │
                              └───────────┼──────────┘
                                          │
                          ┌───────────────┼───────────────┐
                          │               │               │
                   ┌──────▼──────┐ ┌──────▼──────┐ ┌─────▼───────┐
                   │  Bundled    │ │  Remote     │ │  DR Standby  │
                   │  Controller│ │  Controller │ │  (repmgr)    │
                   │  (Default) │ │  (Optional) │ │              │
                   └──────┬─────┘ └──────┬──────┘ └──────────────┘
                          │              │
                   ┌──────▼─────┐ ┌──────▼──────┐
                   │  C-Store   │ │  C-Store    │
                   │  (Read DB) │ │  (Remote)   │
                   └────────────┘ └─────────────┘
```

## Component Summary

### 1. Multi App

Enables horizontal scaling by adding application server nodes in a Parent–Child topology.

| Aspect | Details |
|--------|---------|
| **Purpose** | Horizontal scaling and load distribution |
| **Node Roles** | Parent (primary), Child (replica) |
| **Technologies** | RocksDB (metadata cache), gRPC (inter-node communication) |
| **Features** | Shared file system, leader election, centralized task scheduling |

**Parent Node** responsibilities:
- Serves user requests
- Runs the TaskEngine (centralized scheduler that fetches and executes tasks from the database)
- Manages the centralized RocksDB cache (child nodes access it via gRPC)

**Child Node** responsibilities:
- Serves user requests (load sharing)
- Accesses centralized RocksDB cache via gRPC calls to the parent
- Receives replicated files via gRPC push/pull

> Full provisioning details: [Multi App — PROVISIONING_STEPS](multi-app/PROVISIONING_STEPS.md)

### 2. Remote Controller

A standalone Java tool that synchronizes data from PostgreSQL to a C-Store columnar database for read-optimized analytics queries.

| Aspect | Details |
|--------|---------|
| **Purpose** | PostgreSQL → C-Store data synchronization |
| **Types** | Bundled (default), Remote (optional) |
| **Technologies** | PostgreSQL triggers, C-Store (columnar extension) |
| **Use Case** | Analytics module with read-heavy workloads |

**Bundled Controller** — Included with the product by default. Writes go to PostgreSQL; reads are served from C-Store. A separate schema handles the sync internally.

**Remote Controller** — Optional deployment on a separate machine. Allows splitting read load for specific workspaces when tables or queries are RAM-intensive.

> Full provisioning details: [Remote Controller — PROVISIONING_STEPS](remote-controller/PROVISIONING_STEPS.md)

### 3. DC-DR Support

Disaster Recovery backup for the primary PostgreSQL database using repmgr (open-source replication manager).

| Aspect | Details |
|--------|---------|
| **Purpose** | Database disaster recovery |
| **Technology** | repmgr (PostgreSQL replication manager) |
| **Failover Modes** | Automatic, Semi-Automatic, Manual |

> Full provisioning details: [DC-DR — PROVISIONING_STEPS](dc-dr/PROVISIONING_STEPS.md)

## Data Flow

### Write Path

```
User Request → App Node → PostgreSQL (Write DB)
                              │
                              ├── Triggers capture changes
                              │
                              └── Controller syncs → C-Store (Read DB)
```

### Read Path

```
User Request → App Node → C-Store (Read DB)
```

### Cache Access (Multi App)

```
Child Node ──gRPC──→ Parent Node ──→ RocksDB Cache
```

### File Replication (Multi App)

```
Parent Node ◄──gRPC──► Child Node
   (push/pull file replication)
```

## Network Communication

| Source | Destination | Protocol | Purpose |
|--------|-------------|----------|---------|
| Child App Node | Parent App Node | gRPC | Cache access, file replication |
| Parent App Node | Child App Node | gRPC | File push replication |
| App Node | PostgreSQL | TCP (5432) | Database writes |
| App Node | C-Store | TCP (5432) | Database reads |
| Bundled Controller | PostgreSQL / C-Store | TCP (5432) | Data sync |
| Remote Controller | PostgreSQL | TCP (5432) | Source data |
| Remote Controller | C-Store | TCP (5432) | Target data |
| Primary PostgreSQL | Standby PostgreSQL | TCP (5432) | repmgr replication |

## Technology Stack

| Technology | Purpose | Component |
|------------|---------|-----------|
| **RocksDB** | Metadata cache (framework-level, not product-level) | Multi App |
| **gRPC** | Inter-app communication and file replication | Multi App |
| **PostgreSQL** | Primary write database | All components |
| **C-Store** | Columnar read database (PostgreSQL extension) | Remote Controller |
| **repmgr** | PostgreSQL replication and failover management | DC-DR |
| **Java** | Application runtime and controller implementation | All components |
