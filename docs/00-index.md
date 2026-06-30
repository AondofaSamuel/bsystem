# EBROP Software Architecture Specification

## Purpose

This documentation set is the authoritative engineering specification for the Enterprise Backup & Restore Orchestration Platform. It is written to guide implementation, review, testing, deployment, and operations.

The platform is not a simple backup script. It is an orchestration system that manages trusted agents, schedules, backup policies, restore workflows, replication, verification, retention, audit trails, and disaster recovery procedures.

## Document Set

| Document | Purpose |
|---|---|
| `01-architecture-decisions.md` | Final decisions agreed during architecture review. |
| `02-system-architecture.md` | Platform topology, components, trust boundaries and data flows. |
| `03-agent-architecture.md` | Windows Agent design, lifecycle, state machine, identity and update model. |
| `04-backup-engine.md` | File, SQL Server, MySQL backup pipelines and provider model. |
| `05-restore-engine.md` | Manual restore, automated restore, rollback and validation flows. |
| `06-storage-replication.md` | Local repositories, multi-copy replication, offline media, retention and scrubbing. |
| `07-security-cryptography.md` | mTLS, RBAC, audit logging, encryption, key escrow and recovery. |
| `08-database-design.md` | Core schema, tables, indexes and data ownership model. |
| `09-api-contracts.md` | REST endpoints, SignalR events, job payloads and error contracts. |
| `10-operations-dr.md` | Installation, monitoring, agent replacement, catalog recovery and runbooks. |

## Version 1 Definition of Done

Version 1 is complete when EBROP can:

1. Register protected systems, servers, repositories and agents.
2. Trust agents through certificate approval and mTLS.
3. Schedule backup jobs with retention policies.
4. Back up files using VSS plus robocopy.
5. Back up SQL Server databases using native SQL Server backup.
6. Back up MySQL/MariaDB databases using logical backup.
7. Compress and encrypt artifacts locally before network transfer.
8. Replicate verified artifacts to multiple repositories.
9. Track protection status: Protected, Partially Protected, Unprotected or Failed.
10. Execute manual restores with pre-restore safety checks.
11. Export and replicate a self-backup manifest.
12. Maintain audit logs for every privileged operation.
13. Provide operational dashboards for backup compliance and repository health.

## Non-Goals for Version 1

Version 1 will not attempt to compete with mature commercial products in every workload category. It does not include VM image-level backups, Kubernetes-native backup, Linux agents, immutable cloud storage, or public cloud dependency. These are future extension points.

## Design Philosophy

EBROP follows these principles:

- Prefer native platform capabilities over custom low-level backup implementations.
- Never execute arbitrary commands supplied by users.
- Keep privileged execution on the target server through a local agent.
- Treat restore as a first-class feature from Version 1.
- Encrypt before transfer.
- Verify every copy.
- Assume repositories can fail.
- Assume agents can be replaced.
- Assume the orchestration server itself can be lost.
- Design every destructive action to be auditable.
