# Architecture Decisions

This document records the key decisions that define EBROP. These decisions are binding for Version 1 unless formally amended.

## ADR-001: Local-First Deployment

**Decision:** EBROP Version 1 shall be fully deployable inside a private network without public cloud subscriptions.

**Rationale:** The target environment contains sensitive internal systems and may have policy, budget or operational restrictions against S3, Azure Blob or other public cloud storage.

**Implications:** Storage providers for Version 1 are local disk, UNC share, NAS, secondary repository server and offline external drive. Cloud providers remain extension points.

## ADR-002: .NET 8 Backend

**Decision:** The orchestration server shall be implemented in .NET 8.

**Rationale:** The system is Windows-heavy and requires robust background processing, SignalR, service integration, security support and long-running jobs.

**Implications:** PHP is not used for the orchestration engine. If a PHP UI is ever built, it must call the .NET API rather than executing backup logic.

## ADR-003: C# Windows Service Agents

**Decision:** Protected servers run a C# Windows Service agent.

**Rationale:** Backup and restore operations require local privileges, VSS access, service control, process management and native Windows integration.

**Implications:** The dashboard never runs robocopy, sqlcmd or mysqldump remotely. It dispatches jobs to agents.

## ADR-004: Mutual TLS Agent Trust

**Decision:** Agents authenticate using mTLS with approved certificate fingerprints.

**Rationale:** Static API keys are insufficient. Agent spoofing can cause false backup reporting or unauthorized restore actions.

**Implications:** Agents generate certificates during installation. Administrators approve fingerprints before the agent becomes trusted. Certificates can be revoked.

## ADR-005: Scheduling and Retention in Version 1

**Decision:** Scheduling and retention/pruning ship together in Version 1.

**Rationale:** Scheduled backups without pruning will eventually exhaust repository storage.

**Implications:** Every successful backup run is evaluated against a retention policy. Pruning is audited and must not delete the only verified copy.

## ADR-006: Manual Restore in Version 1

**Decision:** Manual restore execution is required in Version 1.

**Rationale:** A backup platform that cannot restore is operationally incomplete. Automation can mature later, but native restore execution must exist from the start.

**Implications:** Phase 1 includes browse, select, validate, confirm and restore-to-target flows. Full maintenance mode automation and rollback orchestration are later enhancements.

## ADR-007: Compress and Encrypt Locally Before Transfer

**Decision:** Backup artifacts are compressed and encrypted on the agent before being sent across the network.

**Rationale:** This reduces network load and protects sensitive backup data in transit and at rest.

**Implications:** Repositories store encrypted packages. Key management and escrow are mandatory.

## ADR-008: Multi-Repository Replication

**Decision:** Backups must support multiple repositories and per-copy verification.

**Rationale:** A single backup repository is a single point of failure.

**Implications:** Each backup run has one or more backup copies. Overall status is calculated from copy status.

## ADR-009: Self-Backup Manifest

**Decision:** EBROP exports its own recoverable catalog manifest.

**Rationale:** If the orchestration server is lost, encrypted backup packages must remain identifiable and restorable.

**Implications:** Manifest export is scheduled and replicated to repositories. Recovery procedures must import the manifest into a fresh installation.

## ADR-010: Key Escrow Required

**Decision:** Encryption keys must have a documented escrow and disaster recovery process.

**Rationale:** Losing the master key makes all encrypted backups permanently unusable.

**Implications:** Version 1 uses a DEK/KEK model with KEK protection through Windows Certificate Store or equivalent. Escrow packages must be tested.

## ADR-011: Repository Scrubbing

**Decision:** The platform shall periodically recalculate repository hashes to detect bit-rot or silent corruption.

**Rationale:** A backup verified today can become corrupted months later.

**Implications:** Repository Integrity Verification jobs are part of the operations model.

## ADR-012: Throttling and Workload Protection

**Decision:** The agent enforces network, CPU, I/O and concurrency controls.

**Rationale:** Backup jobs must not degrade production systems.

**Implications:** Agent configuration includes max concurrent jobs, process priority, transfer limits and business-hour behavior.

## ADR-013: Agent Lifecycle Management

**Decision:** EBROP manages agent versions, updates, revocation and replacement.

**Rationale:** Enterprise agents must be patchable without manual RDP to every server.

**Implications:** Agent updates are signed, verified, scheduled and reversible.
