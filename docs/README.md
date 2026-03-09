# Distributed Architecture — Technical Documentation

This directory contains the technical documentation for the distributed architecture framework, including provisioning steps, component overviews, and the documentation generation protocol.

## Components

| Component | Description | Documentation |
|-----------|-------------|---------------|
| **Multi App** | Horizontal scaling with Parent/Child node architecture | [Provisioning Steps](multi-app/PROVISIONING_STEPS.md) |
| **Remote Controller** | PostgreSQL → C-Store data synchronization tool | [Provisioning Steps](remote-controller/PROVISIONING_STEPS.md) |
| **DC-DR Support** | Disaster Recovery backup using repmgr | [Provisioning Steps](dc-dr/PROVISIONING_STEPS.md) |

## Architecture Reference

- [Architecture Overview](architecture-overview.md) — High-level distributed architecture diagram and component relationships
- [Documentation Protocol](DOCUMENTATION_PROTOCOL.md) — Protocol for generating and maintaining technical documentation

## Quick Navigation

### By Task

| Task | Go To |
|------|-------|
| Set up a new Multi App cluster | [Multi App Provisioning](multi-app/PROVISIONING_STEPS.md) |
| Deploy a Remote Controller instance | [Remote Controller Provisioning](remote-controller/PROVISIONING_STEPS.md) |
| Configure DC-DR replication | [DC-DR Provisioning](dc-dr/PROVISIONING_STEPS.md) |
| Understand the overall architecture | [Architecture Overview](architecture-overview.md) |
| Contribute new documentation | [Documentation Protocol](DOCUMENTATION_PROTOCOL.md) |

### By Technology

| Technology | Used In |
|------------|---------|
| RocksDB | [Multi App](multi-app/PROVISIONING_STEPS.md) — Centralized metadata cache |
| gRPC | [Multi App](multi-app/PROVISIONING_STEPS.md) — Inter-app communication |
| PostgreSQL | [Remote Controller](remote-controller/PROVISIONING_STEPS.md), [DC-DR](dc-dr/PROVISIONING_STEPS.md) |
| C-Store | [Remote Controller](remote-controller/PROVISIONING_STEPS.md) — Columnar read database |
| repmgr | [DC-DR](dc-dr/PROVISIONING_STEPS.md) — PostgreSQL replication manager |
