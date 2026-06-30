# EBROP — Enterprise Backup & Restore Orchestration Platform

This repository contains the Software Architecture Specification for **EBROP**, a local-first enterprise backup and restore orchestration platform for Windows-based infrastructure.

EBROP is designed around trusted Windows agents, secure orchestration, local repositories, multi-repository replication, AES-256 encrypted backup artifacts, retention-aware scheduling, manual and automated restore workflows, and disaster recovery of the platform itself.

## Documentation

Start here:

- [Documentation Index](docs/00-index.md)
- [Architecture Decisions](docs/01-architecture-decisions.md)
- [System Architecture](docs/02-system-architecture.md)
- [Agent Architecture](docs/03-agent-architecture.md)
- [Backup Engine](docs/04-backup-engine.md)
- [Restore Engine](docs/05-restore-engine.md)
- [Storage and Replication](docs/06-storage-replication.md)
- [Security and Cryptography](docs/07-security-cryptography.md)
- [Database Design](docs/08-database-design.md)
- [API Contracts](docs/09-api-contracts.md)
- [Operations and Disaster Recovery](docs/10-operations-dr.md)

## Version 1 Scope

Version 1 is local-first and does **not** depend on S3, Azure Blob, or public cloud subscriptions. Supported storage targets are local disks, UNC shares, NAS, secondary backup servers, and offline removable drives.

## Core Architecture

```text
Dashboard/API (.NET 8)
        ↓
Scheduler + Job Queue
        ↓
Trusted C# Windows Agents over mTLS
        ↓
Native Backup Providers: VSS, robocopy, SQL Server, MySQL/MariaDB
        ↓
Compression → Encryption → Checksum Verification
        ↓
Primary Repository → Secondary Repository → Offline Repository
```

## Engineering Baseline

The agreed baseline is:

- .NET 8 backend.
- C# Windows Service agents.
- Mutual TLS agent identity.
- Scheduling and retention in Version 1.
- Manual restore in Version 1.
- Automated restore orchestration in later versions.
- Multi-repository replication with protection status.
- Self-backup manifest for catalog recovery.
- Key escrow and recovery process.
- Repository integrity verification.
- Agent lifecycle management, update and replacement workflows.
- VSS-aware file backup.
- SQL Server and MySQL/MariaDB support.
- Local-first deployment model.
