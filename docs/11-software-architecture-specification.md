# EBROP Software Architecture Specification

Version: 1.0  
Status: Authoritative implementation baseline  
Product: Enterprise Backup & Restore Orchestration Platform (EBROP)  
Primary deployment model: Local-first Windows enterprise environment

## 1. Purpose and Authority

This Software Architecture Specification is the controlling engineering document for EBROP implementation. It consolidates the repository documentation, the architectural review conversation, the direct ChatGPT continuation thread, and the later documentation artifacts, including the consolidated SAS v2 and architecture addendum, into one baseline.

Developers must treat this document as the source of truth when implementation details are not yet present in code. Existing topic files in `docs/01-architecture-decisions.md` through `docs/10-operations-dr.md` remain supporting chapters. Where those files are shorter or less explicit, this SAS controls.

The objective of Version 1 is not to rival mature commercial products. The objective is to produce a reliable internal backup and restore orchestration platform for Windows-based infrastructure with trusted agents, verified encrypted backups, local multi-repository replication, retention, manual restore, and catalog recovery.

## 2. Final Architecture Decisions That Survived Review

The following decisions survived the review process and are binding for Version 1 unless the project owner explicitly amends them.

| ID | Final decision | Binding implementation consequence | Superseded proposal |
|---|---|---|---|
| AD-001 | EBROP is a platform, not a script or simple dashboard. | Implement orchestration, agents, policies, catalog, restore, replication, retention, audit, and recovery as first-class subsystems. | Single-purpose backup script or web page that runs copy commands. |
| AD-002 | Version 1 is local-first. | Storage providers for Version 1 are local disk, UNC share, NAS, secondary server, and offline media. | S3, Azure Blob, or public cloud as required Version 1 dependencies. |
| AD-003 | The orchestration server uses .NET 8. | Backend services, scheduler integration, API, SignalR, and background work are implemented in .NET 8. | PHP orchestration server for backup execution. |
| AD-004 | Protected servers run C# Windows Service agents. | All privileged backup and restore work happens inside local agents. | PHP agents, scheduled PHP scripts, or dashboard-initiated remote command execution. |
| AD-005 | The dashboard never executes privileged backup commands remotely. | Dashboard/API creates jobs and policy; agents execute `robocopy`, VSS, SQL Server, MySQL, compression, encryption, and restore locally. | `exec`, WinRM, SSH, WMI, or web UI directly invoking production backup commands. |
| AD-006 | Agent trust uses mTLS with certificate approval. | Agent installation generates certificate identity; administrators approve fingerprints; revoked agents cannot submit results or receive jobs. | Static API keys as the only agent trust mechanism. |
| AD-007 | Scheduling and retention ship in Version 1. | A persistent scheduler and pruning engine are mandatory before production use. | Manual-only backup jobs, external Windows Task Scheduler, or retention deferred to later. |
| AD-008 | Compression and encryption happen locally before transfer. | Agents stage provider output, compress it, encrypt it, and only then transfer packages to repositories. | Raw dumps or raw file copies transferred across the network before protection. |
| AD-009 | Multi-repository replication is core Version 1 behavior. | Each backup run can have multiple copy records and computed protection status. | One vague "backup repository" treated as sufficient protection. |
| AD-010 | Offline repositories are expected to be disconnected. | Missing offline media produces pending/warning state, not automatic critical failure. | Treating disconnected offline drive as ordinary repository failure. |
| AD-011 | Manual restore is required in Version 1. | The platform must browse, validate, confirm, dispatch, execute, and audit restore jobs. | Backup-only Version 1 or fully automated restore before manual restore is stable. |
| AD-012 | Automated restore orchestration is later work. | Maintenance mode, automatic service stop/start, dependency sequencing, rollback automation, and health-driven auto-rollback are roadmap items built on manual restore. | Implementing unsafe one-click production overwrite in Version 1. |
| AD-013 | VSS-aware file backup is required. | File provider must use VSS where policy or workload requires consistent locked-file capture. | Robocopy alone for active Windows/IIS application folders. |
| AD-014 | SQL Server uses native database backup. | Use SQL Server `BACKUP DATABASE` with `CHECKSUM` and `RESTORE VERIFYONLY`; do not copy MDF/LDF files while running. | File-copying SQL Server data files as normal backup. |
| AD-015 | MySQL/MariaDB Version 1 uses logical backup. | Use `mysqldump --single-transaction --routines --triggers --events --quick`; physical backup/PITR is future work. | Building custom database readers or requiring Percona XtraBackup in Version 1. |
| AD-016 | SQL Server Version 1 supports full backup and manual restore. | Differential, transaction log, and point-in-time restore are roadmap items. | Promising full PITR for all database engines in Version 1. |
| AD-017 | Key escrow is mandatory. | Encrypted backups must remain recoverable after management server loss through a documented DEK/KEK and escrow process. | Storing the only master key in the EBROP database or server disk. |
| AD-018 | Repository integrity verification is mandatory. | Periodic scrub jobs recalculate repository hashes and mark corrupted copies. | Verifying only at initial write time and assuming repositories remain healthy. |
| AD-019 | Agent lifecycle management is mandatory. | Track versions, updates, revocation, replacement, and retirement. | Manual RDP-based update of every agent or orphaned replacement agents. |
| AD-020 | Source workload protection is mandatory. | Agents enforce CPU, I/O, network, compression-thread, and concurrency limits. | Only network throttling, or no local workload protection. |
| AD-021 | Restore safety validation is mandatory. | Dry-run, disk-space checks, overwrite conflict choices, database existence checks, and audit are required. | Silent overwrite or implicit SQL `WITH REPLACE`. |
| AD-022 | SignalR is for state, not bulk logs. | Progress and state transitions use SignalR; high-volume logs use batched REST ingestion. | Streaming raw logs over SignalR during heavy transfers. |
| AD-023 | Self-backup manifest is mandatory. | EBROP exports catalog, policies, repository metadata, key-wrapping metadata, and recovery instructions to repositories. | Backups exist but cannot be identified after dashboard loss. |
| AD-024 | Auditability is mandatory. | Privileged and destructive actions are append-only through normal application paths and must include actor, target, decision, result, and correlation ID. | Untraceable administrative changes. |

## 3. Version 1 Scope

Version 1 is complete only when EBROP can:

1. Register users, roles, systems, servers, repositories, agents, policies, backup jobs, and restore jobs.
2. Install a C# Windows Service agent on each protected Windows server.
3. Register agents with generated certificates and administrator-approved fingerprints.
4. Communicate between server and agents over mTLS.
5. Schedule backup jobs using a persistent .NET scheduler.
6. Apply retention policies automatically.
7. Run file backups using VSS plus robocopy when VSS is required.
8. Run SQL Server full backups using native SQL Server backup with checksum verification.
9. Run MySQL/MariaDB logical backups using `mysqldump`.
10. Compress and encrypt artifacts locally before transfer.
11. Replicate encrypted packages to primary, secondary, and offline repositories.
12. Track per-copy status and overall protection status.
13. Run manual file, SQL Server, and MySQL/MariaDB restores with pre-restore validation.
14. Export and replicate self-backup manifests.
15. Maintain key escrow so encrypted backups survive platform loss.
16. Scrub repository copies for bit-rot or silent corruption.
17. Manage agent update, replacement, revocation, and retirement workflows.
18. Emit metrics, logs, alerts, reports, and audit events.

Version 1 explicitly excludes:

1. Public cloud dependency.
2. Linux agents.
3. VM image-level backup.
4. Kubernetes-native backup.
5. Bare-metal recovery.
6. Deduplication.
7. Immutable/WORM cloud storage.
8. Tape library support.
9. Full point-in-time restore for all engines.
10. Commercial parity with Veeam, Commvault, Rubrik, or Cohesity.

## 4. Technology Baseline

The Version 1 implementation baseline is:

| Layer | Required baseline |
|---|---|
| Backend | .NET 8, ASP.NET Core Web API |
| UI | Razor Pages, MVC, Blazor, React, or another UI that calls the API only. UI must not execute backup logic. |
| Scheduler | Hangfire with persistent SQL-backed storage. Quartz.NET is a permitted substitution only by explicit architecture amendment. |
| Agent | C# .NET Worker Service hosted as Windows Service |
| Agent local state | SQLite or a durable local store under `C:\ProgramData\EBROP\Agent` |
| Realtime events | SignalR for state transitions and lightweight progress |
| Bulk logs | REST ingestion in batches |
| Metadata database | SQL Server as default; MySQL/MariaDB may be supported through EF Core provider abstraction |
| File backup | VSS snapshot when required, robocopy for copy |
| SQL Server | Native `BACKUP DATABASE`, `RESTORE VERIFYONLY`, SQL Server driver/SMO/sqlcmd |
| MySQL/MariaDB | `mysqldump` and `mysql` client |
| Crypto | Envelope encryption using per-artifact DEK and protected KEK |
| Packaging | Versioned EBROP package format with manifest, encrypted payload, checksums, and key metadata |

## 5. Topology and Trust Boundaries

### 5.1 Logical Topology

```text
Administrator
    |
    v
Dashboard / API Server (.NET 8)
    |
    v
Scheduler + Durable Job Store
    |
    v
Agent Registry + Repository Manager + Audit Service
    |
    v
C# Windows Agent over mTLS
    |
    +-- VSS
    +-- robocopy
    +-- SQL Server backup/restore
    +-- mysqldump/mysql
    +-- compression
    +-- encryption
    +-- repository transfer
    v
Primary Repository -> Secondary Repository -> Offline Repository
```

### 5.2 Deployment Topology

Production deployments should separate these roles:

1. EBROP management server: hosts API, dashboard, scheduler, SignalR hub, notification service, and audit service.
2. Metadata database: stores policy, catalog, job state, audit, key metadata, and manifest records.
3. Primary repository: first verified storage target.
4. Secondary repository: independent copy on a different disk/server/location where possible.
5. Offline repository: removable or disconnected media for air-gapped protection.
6. Protected servers: each has one installed EBROP agent.

The dashboard server and primary repository should not be the same physical server in production. If unavoidable for a small pilot, the deployment must still replicate to a secondary repository before being treated as protected.

### 5.3 Trust Boundaries

| Boundary | Required protection |
|---|---|
| User to dashboard | HTTPS, authenticated session, RBAC, audit |
| Dashboard/API to agent | mTLS, approved certificate, revocation check, job authorization |
| Agent to local resources | Least privilege service account, configured paths/databases only |
| Agent to repository | Dedicated repository credentials, encrypted packages only, copy verification |
| Metadata DB | least privilege application account, encrypted secrets, backups |
| Key escrow | protected export process, restricted recovery role, audit |

Internal network location is not trust. Agents, users, repositories, and server processes must authenticate explicitly.

## 6. Domain States and Statuses

### 6.1 Agent States

Agents must implement these states:

| State | Meaning | Job execution allowed |
|---|---|---|
| PendingApproval | Agent registered but fingerprint not approved | No |
| Approved | Certificate trusted but not currently connected | No until online |
| Online | Connected and healthy | Yes if idle |
| Idle | Online and no active job | Yes |
| RunningBackup | Backup provider or packaging is active | No additional same-resource job |
| Replicating | Repository copy or verification is active | Controlled by concurrency policy |
| WaitingForOfflineMedia | Online copies complete; offline copy pending | Yes for unrelated jobs if policy allows |
| RunningRestore | Restore is active | No backup on same resource |
| Maintenance | Operator or policy has paused work | No |
| Updating | Signed update in progress | No |
| Replacing | Old agent is being replaced | No |
| Revoked | Certificate not trusted | No |
| Retired | Historical record only | No |

Agents in `PendingApproval`, `Revoked`, or `Retired` must not accept jobs or submit successful job results.

### 6.2 Backup Run Status

Backup run status records provider/package execution:

| Status | Meaning |
|---|---|
| Queued | Job exists but agent has not accepted it |
| Assigned | Job assigned to a specific agent |
| Running | Agent has accepted and started |
| Packaging | Compression/encryption/checksum phase |
| Replicating | At least one repository copy in progress |
| Completed | Provider and required finalization completed |
| Failed | Provider, packaging, or all required copies failed |
| Cancelled | Operator or policy stopped job before completion |
| Stalled | No progress heartbeat within configured timeout |

### 6.3 Backup Copy Status

Backup copy status records per-repository state:

| Status | Meaning |
|---|---|
| Pending | Copy is planned but not started |
| Copying | Agent is writing package |
| Verifying | Agent is recalculating destination checksum |
| Success | Copy exists and checksum matches |
| Failed | Copy failed after retries |
| Corrupted | Scrub or restore verification found mismatch |
| PendingOfflineMedia | Offline repository media is expected but not connected |
| Pruned | Copy was deleted by retention policy |
| LegalHold | Copy is protected from pruning |

### 6.4 Protection Status

Protection status is computed from copy statuses:

| Status | Meaning |
|---|---|
| Protected | All required repositories have verified copies |
| PartiallyProtected | At least one verified copy exists, but one or more required copies are missing, pending, failed, or offline |
| Unprotected | No verified copy exists |
| Failed | Backup creation failed or all required copies failed |

## 7. Dashboard and API Server

### 7.1 Responsibilities

The Dashboard/API server owns the control plane:

1. Authentication and RBAC.
2. System, server, agent, repository, policy, backup job, and restore job configuration.
3. Agent registration approval and certificate revocation.
4. Scheduler and durable job creation.
5. Backup catalog and restore-point browsing.
6. Repository health and protection status reporting.
7. Key escrow export/import workflows.
8. Audit, reporting, and notifications.

The server must not:

1. Execute `robocopy`, `sqlcmd`, `mysqldump`, VSS, compression, encryption, or restore directly against protected systems.
2. Accept arbitrary shell commands from users.
3. Trust agent reports from unapproved or revoked certificates.
4. Mark a backup protected without verified repository copies.

### 7.2 RBAC Roles

Version 1 must include at least:

| Role | Capabilities |
|---|---|
| SuperAdmin | Full configuration, key escrow, restore approval, agent revocation |
| BackupAdmin | Manage systems, jobs, schedules, repositories, and policies |
| RestoreOperator | Request and execute approved restore jobs |
| Auditor | View reports, logs, audit, and evidence |
| Viewer | Read-only dashboard access |

Production restores and key escrow export require elevated permission and audit. The implementation may require second-person approval for production restore, but at minimum must record requesting user, approving user where applicable, reason, selected restore behavior, and result.

## 8. Scheduler and Job Queue

### 8.1 Scheduler Baseline

Version 1 uses Hangfire with persistent storage. The scheduler must survive application restarts and must not lose due jobs. Scheduled work includes:

1. Backup jobs.
2. Retention/pruning jobs.
3. Repository health checks.
4. Repository scrubbing.
5. Self-backup manifest export.
6. Agent update notifications.
7. Alert evaluation.

### 8.2 Dispatch Model

Agents communicate outbound over mTLS. Version 1 should prefer a pull or long-poll model to avoid inbound firewall requirements:

```text
Agent -> POST /api/v1/agents/{id}/heartbeat
Agent -> GET  /api/v1/agents/{id}/jobs/next
Agent -> POST /api/v1/jobs/{jobId}/progress
Agent -> POST /api/v1/jobs/{jobId}/logs/batch
Agent -> POST /api/v1/jobs/{jobId}/complete
```

SignalR may notify connected agents that work is available, but durable job state lives in the server database. If SignalR disconnects, agents still discover jobs through polling.

### 8.3 Scheduling Rules

The scheduler must enforce:

1. Do not run two jobs against the same protected resource unless policy explicitly allows it.
2. Respect maintenance windows and business-hour low-impact mode.
3. Defer jobs when the agent is offline, revoked, updating, replacing, or in maintenance.
4. Retry transient failures using configured retry policy with exponential backoff.
5. Mark jobs stalled when no heartbeat/progress is received within timeout.
6. Run retention evaluation only after backup run and copy status are settled.
7. Never prune the last verified usable copy.

## 9. Agent Architecture

### 9.1 Agent Responsibilities

The agent is the execution plane. It runs as a Windows Service on each protected server and must:

1. Generate and protect its certificate identity.
2. Register with the server and wait for approval.
3. Maintain mTLS communication and heartbeat.
4. Receive jobs addressed to its own agent ID.
5. Validate policy, prerequisites, tools, permissions, resources, and repository availability.
6. Acquire locks for protected resources.
7. Execute provider-specific backup and restore.
8. Stage, compress, encrypt, checksum, and transfer artifacts.
9. Verify repository copies.
10. Persist local execution state.
11. Enforce resource controls.
12. Publish structured progress, logs, metrics, and audit events.
13. Support signed updates and replacement workflow.

### 9.2 Internal Components

| Component | Responsibility |
|---|---|
| ConnectionManager | mTLS connection, server identity validation, heartbeat, reconnect backoff |
| RegistrationManager | certificate creation, registration request, local identity persistence |
| JobReceiver | polls/receives assigned jobs, validates job envelope |
| JobProcessor | state machine, dispatch, resource locking, cancellation |
| BackupEngine | provider execution, staging, checksum, compression, encryption |
| RestoreEngine | dry-run, retrieval, decrypt/decompress, provider restore |
| RepositoryManager | copy, verification, offline pending, retry |
| LocalStateStore | SQLite/durable state for active jobs and recoverable progress |
| HealthMonitor | CPU, memory, disk, service, IIS, SQL, repository, custom checks |
| UpdateManager | signed package verification, maintenance-window update, rollback |
| AuditPublisher | event batching and retry |

### 9.3 Local Directory Layout

The agent should use a predictable local layout:

```text
C:\ProgramData\EBROP\Agent\
    config\
        agent.json
        repositories.cache.json
    certs\
        agent.pfx or certificate-store reference
    state\
        agent-state.db
    staging\
        {jobId}\
    packages\
        {jobId}\
    logs\
        agent-.log
    updates\
        current\
        pending\
        rollback\
```

Secrets should not be stored in plaintext JSON. Use DPAPI, Windows Certificate Store, or an equivalent local secret-protection mechanism.

### 9.4 Job Acceptance Rules

Before accepting a job, the agent must verify:

1. mTLS session is valid.
2. Server certificate identity is trusted.
3. Agent certificate is approved and not revoked.
4. Job is addressed to this agent ID.
5. Job type is supported by installed providers.
6. Required paths/databases/repositories are registered or authorized by policy.
7. Required external tools are available.
8. The local service account has required privileges.
9. Business-hour and maintenance-window policy allows execution.
10. Resource lock is available.
11. Concurrency and throttling limits allow execution.

If any check fails, the job must fail or defer with a structured error; the agent must not improvise commands.

### 9.5 Local Failure Recovery

A restarted agent must read `LocalStateStore` before accepting new work. It must never assume an interrupted job succeeded. For each in-flight job it must choose one of:

1. Resume a copy or verification step when the operation is explicitly resumable.
2. Clean staging and restart when retry policy allows.
3. Mark the job failed with `RecoverableAfterAgentRestart`.
4. Require operator review when restore or destructive state is uncertain.

## 10. Backup Engine

### 10.1 Universal Backup Pipeline

Every provider follows this pipeline:

1. Load job definition and correlation ID.
2. Validate policy, resource, credentials, and agent state.
3. Run pre-flight checks.
4. Acquire resource lock.
5. Prepare staging directory.
6. Create VSS snapshot if required.
7. Execute provider backup to staging.
8. Calculate provider output checksum.
9. Compress artifact.
10. Encrypt compressed payload.
11. Calculate encrypted package checksum.
12. Write package manifest.
13. Replicate package to repositories.
14. Verify repository copy checksums.
15. Publish copy and protection status.
16. Run retention evaluation.
17. Clean staging according to policy.
18. Publish audit and metrics.

### 10.2 Provider Contract

Each provider must implement equivalent behavior to:

```csharp
public interface IBackupProvider
{
    string ProviderKey { get; }
    Task<ValidationResult> ValidateAsync(BackupJobContext context, CancellationToken ct);
    Task<BackupResult> ExecuteBackupAsync(BackupJobContext context, CancellationToken ct);
    Task<VerificationResult> VerifyAsync(BackupArtifact artifact, CancellationToken ct);
    Task<RestorePlan> CreateRestorePlanAsync(BackupArtifact artifact, RestoreTarget target, CancellationToken ct);
    Task<RestoreResult> ExecuteRestoreAsync(RestoreJobContext context, CancellationToken ct);
    Task CleanupAsync(JobContext context, CancellationToken ct);
}
```

Providers must not decide scheduling, retention, repository selection, key policy, or RBAC. Those are orchestration concerns.

### 10.3 File Provider

The File Provider protects registered directories such as IIS roots, upload folders, media folders, configuration folders, and application data folders.

Required behavior:

1. Validate each source path is registered for the protected system.
2. Validate destination staging space.
3. If `useVss = true`, create a VSS snapshot and copy from the snapshot path.
4. Use robocopy for file copying.
5. Preserve timestamps, ACLs, owner, auditing, and directory metadata where privileges allow.
6. Map robocopy exit codes correctly: 0 through 7 are not automatic failure; 8 and above are failure.
7. Release VSS snapshot in cleanup even when copy fails.

Recommended robocopy baseline:

```cmd
robocopy "SOURCE" "DESTINATION" /MIR /COPYALL /DCOPY:DAT /R:2 /W:2 /MT:16 /FFT
```

Use `/ZB` or `/B` only when the service account has the required privileges and policy allows backup mode.

If VSS is required by policy and VSS creation fails, the backup must fail unless the policy explicitly permits non-VSS fallback.

### 10.4 SQL Server Provider

Version 1 supports full backups and manual restore.

Required backup behavior:

1. Prefer Windows Authentication through a service account.
2. SQL Authentication is allowed only where necessary and credentials must be encrypted.
3. Use SQL Server native backup, not MDF/LDF file copy.
4. Use `CHECKSUM`.
5. Run `RESTORE VERIFYONLY WITH CHECKSUM`.
6. Record database name, instance, backup type, start/end time, size, checksum, SQL messages, and verification result.

Baseline full backup:

```sql
BACKUP DATABASE [DatabaseName]
TO DISK = N'D:\EBROP\Staging\DatabaseName_full.bak'
WITH INIT, COMPRESSION, CHECKSUM, STATS = 10;
```

Verification:

```sql
RESTORE VERIFYONLY
FROM DISK = N'D:\EBROP\Staging\DatabaseName_full.bak'
WITH CHECKSUM;
```

Future work includes differential backup, transaction log backup, log chain management, and point-in-time restore.

### 10.5 MySQL/MariaDB Provider

Version 1 supports logical backup.

Baseline command:

```cmd
mysqldump --single-transaction --routines --triggers --events --quick database_name
```

Required behavior:

1. Capture stdout to staging artifact.
2. Capture stderr and exit code.
3. Treat zero-size dumps as failure.
4. Record dump size, duration, server version, database name, and warnings.
5. Warn when database size or duration indicates logical backup may no longer meet RPO/RTO.

Future work includes Percona XtraBackup, binary logs, physical backup, and point-in-time restore.

### 10.6 Compression, Encryption, and Package Format

Compression happens before encryption. Repositories must never receive plaintext production data.

Each backup package must contain or reference:

1. Package format version.
2. Backup run ID.
3. System ID.
4. Agent ID.
5. Provider key and provider metadata.
6. Created timestamp.
7. Compression algorithm and settings.
8. Encryption algorithm and key metadata.
9. Wrapped DEK reference.
10. Source checksum of provider output.
11. Package checksum of encrypted bytes.
12. Restore hints and compatibility metadata.

Recommended package extension: `.ebropkg`.

The implementation must store both:

1. Source checksum: calculated over provider output before compression/encryption.
2. Package checksum: calculated over the encrypted package stored in repositories.

Repository copy verification and repository scrub compare package bytes to the anchored package checksum. Restore verification decrypts/decompresses and compares provider output to the anchored source checksum.

## 11. Restore Engine

### 11.1 Version 1 Manual Restore

Manual restore means EBROP executes restore operations, but an administrator may still manage maintenance mode, communication, service shutdown, and final business validation.

Version 1 restore flow:

1. Administrator selects system and restore point.
2. Dashboard lists verified copies and warns about corrupted or missing copies.
3. Administrator selects restore target and restore mode.
4. System creates restore job in dry-run mode by default.
5. Agent validates repository access, package checksum, decryption key availability, target space, target conflicts, provider availability, and privileges.
6. Dry-run returns blocking issues and expected actions.
7. Administrator chooses conflict behavior.
8. Administrator explicitly confirms execution.
9. Agent retrieves package.
10. Agent verifies package checksum.
11. Agent decrypts and decompresses into staging.
12. Agent verifies source checksum.
13. Provider performs restore.
14. Agent records result, logs, metrics, and audit.

### 11.2 Restore Modes

| Mode | Version 1 behavior |
|---|---|
| FilesOnly | Restore selected files/folders or whole file artifact |
| DatabaseOnly | Restore SQL Server or MySQL/MariaDB artifact |
| FullApplication | Restore files and database as separate coordinated jobs; full dependency automation is future |
| TestRestore | Restore to isolated path/database/server |
| ProductionRestore | Requires elevated permission, explicit confirmation, and audit |

### 11.3 Required Safety Checks

Before modifying a target, the agent must validate:

1. Destination disk space.
2. Target path existence.
3. Existing file/folder conflict.
4. Database existence.
5. Required privileges.
6. Repository availability.
7. Package checksum.
8. Decryption key availability.
9. Provider compatibility.
10. Expected restore duration where estimable.

If a target path exists, the administrator must choose:

1. Overwrite.
2. Rename existing target.
3. Restore to alternate path.
4. Cancel.

If a database exists, the administrator must choose provider-specific behavior. SQL Server `WITH REPLACE` must never be assumed.

### 11.4 Automated Restore Roadmap

Later versions add:

1. Maintenance mode.
2. IIS app pool stop/start.
3. Windows Service stop/start.
4. Rollback backup before restore.
5. Dependency-aware sequencing.
6. Health validation.
7. Optional automatic rollback on failed validation.

Automated restore must be built on the same validation and audit model as manual restore.

## 12. Storage and Replication

### 12.1 Repository Types

Version 1 repository types:

| Type | Example | Notes |
|---|---|---|
| LocalDisk | `D:\Backups` | Acceptable for pilot/staging, not sole production protection |
| UNCShare | `\\BACKUP01\Backups` | Common primary network repository |
| NAS | `\\NAS01\EBROP` | Suitable for larger local storage |
| SecondaryServer | `\\BACKUP02\Backups` | Independent copy |
| OfflineMedia | `E:\MonthlyBackups` | Expected to be disconnected most of the time |

### 12.2 Repository Metadata

Each repository record must store:

1. Repository ID and code.
2. Name.
3. Type.
4. Path.
5. Required or optional status.
6. Health status.
7. Capacity and free space.
8. Last checked time.
9. Last successful write.
10. Last scrub date.
11. Bandwidth limit.
12. Concurrent copy limit.
13. Retention policy.
14. Offline media identifier where applicable.

### 12.3 Replication Workflow

Default replication is sequential:

1. Agent verifies local package.
2. Copy to primary repository.
3. Verify primary package checksum.
4. Copy to secondary repository.
5. Verify secondary package checksum.
6. If offline repository is connected, verify media identity and copy.
7. If offline repository is missing, create `PendingOfflineMedia` copy status.
8. Recalculate protection status.

Parallel replication is allowed only when policy and throttling permit it.

### 12.4 Offline Repository Workflow

Offline media missing during scheduled backup is not automatically critical. It is a warning unless policy says the offline copy is overdue.

When media is connected:

1. Administrator selects pending offline copy.
2. Administrator triggers retry.
3. Agent verifies media identity.
4. Agent copies package.
5. Agent verifies checksum.
6. Copy status changes to `Success`.
7. Protection status is recalculated.

### 12.5 Retention and Pruning

Default GFS retention:

1. Hourly: 48 hours.
2. Daily: 30 days.
3. Weekly: 12 weeks.
4. Monthly: 12 months.
5. Yearly: configurable.

Retention rules:

1. Never delete the last verified copy.
2. Never delete legal-hold backups.
3. Never delete backups required for restore chain continuity.
4. Never delete metadata required to interpret encrypted packages.
5. Prefer optional copies before required copies under storage pressure.
6. Always audit pruning.

### 12.6 Repository Scrubbing

Repository scrubbing runs periodically, monthly by default:

1. Select candidate backup copies by policy.
2. Read repository package.
3. Recalculate package checksum.
4. Compare with anchored package checksum.
5. Mark copy `Success` or `Corrupted`.
6. Alert on mismatch.
7. Exclude corrupted copy from normal restore selection.

Administrators may override restore from a corrupted copy only through elevated, audited emergency workflow.

## 13. Security and Cryptography

### 13.1 Security Principles

1. No arbitrary command execution.
2. Least privilege for users, dashboard, agents, DB accounts, and repositories.
3. Certificate-based agent trust.
4. Encryption before transfer.
5. Key escrow for disaster recovery.
6. Audit every sensitive action.
7. Explicit restore confirmation.
8. Revocation must take effect immediately for future jobs and reports.

### 13.2 Agent Certificate Lifecycle

Certificate states:

1. Pending.
2. Approved.
3. Active.
4. Suspended.
5. Revoked.
6. Replaced.
7. Expired.

Registration flow:

1. Agent generates key pair and certificate.
2. Agent submits registration request with hostname, IP, version, subject, and fingerprint.
3. Dashboard records agent as `PendingApproval`.
4. Administrator verifies host and fingerprint.
5. Administrator approves certificate.
6. Agent reconnects using mTLS.
7. Agent becomes trusted.

### 13.3 Envelope Encryption

Every backup package uses envelope encryption:

```text
Provider output
    -> compressed
    -> encrypted with Data Encryption Key (DEK)
DEK
    -> wrapped by Key Encryption Key (KEK)
KEK
    -> protected by Windows Certificate Store or approved escrow mechanism
```

Requirements:

1. Each backup package or backup set uses a unique DEK.
2. DEKs are never stored plaintext.
3. Wrapped DEK metadata is stored in catalog/package metadata.
4. KEK recovery is possible through escrow.
5. Key export/import operations require elevated permission and audit.

### 13.4 Key Escrow

Version 1 uses Windows Certificate Store or equivalent OS-backed key protection with an encrypted escrow package.

Escrow package must include:

1. Wrapped KEK material or certificate export reference.
2. Certificate thumbprint.
3. Key version.
4. Manifest version.
5. Creation date.
6. Recovery metadata.
7. Integrity checksum.
8. Authorized recovery procedure reference.

Shamir Secret Sharing and HSM integration are future options, not Version 1 requirements.

### 13.5 Disaster Decryption Flow

1. Install fresh EBROP server.
2. Import latest system manifest.
3. Restore certificate or KEK from escrow.
4. Validate escrow package integrity.
5. Rebuild key registry.
6. Verify ability to decrypt a test package.
7. Proceed with restore.

## 14. Database and Catalog Design

The metadata database is the control-plane catalog. It is not the repository.

### 14.1 Core Tables

Minimum Version 1 entity set:

| Table | Purpose |
|---|---|
| users | User identities |
| roles | RBAC roles |
| permissions | Fine-grained permissions |
| role_permissions | Role permission mapping |
| systems | Protected applications/workloads |
| servers | Physical/virtual Windows servers |
| agents | Installed EBROP agents |
| agent_certificates | Agent certificate lifecycle |
| repositories | Storage targets |
| retention_policies | GFS and custom retention |
| backup_jobs | Configured backup jobs |
| backup_runs | Executed backup runs |
| backup_copies | Per-repository copies |
| restore_jobs | Restore requests/executions |
| job_logs | Operational logs |
| audit_logs | Security/audit records |
| encryption_keys | DEK/KEK metadata and wrapped key references |
| key_escrow_packages | Escrow exports and validation records |
| system_manifests | Self-backup manifest exports |
| health_checks | Health providers and targets |
| dependency_profiles | Pre/post actions and dependency order |

### 14.2 Required Relationships

1. A system has one or more backup jobs.
2. A server has zero or one active agent, but may have historical agents.
3. An agent has one or more certificates over time.
4. A backup job targets one system and one agent.
5. A backup run belongs to one backup job.
6. A backup run has one or more backup copies.
7. A backup copy belongs to one repository.
8. A restore job references one backup run or copy.
9. Audit logs reference user, agent, entity type, and entity ID where applicable.
10. Encryption key records link to backup runs/packages without storing plaintext keys.

### 14.3 Indexing Requirements

Required indexes:

1. `backup_runs(backup_job_id, started_at)`.
2. `backup_runs(system_id, status)`.
3. `backup_runs(protection_status, completed_at)`.
4. `backup_copies(backup_run_id, repository_id)`.
5. `backup_copies(status, repository_id)`.
6. `restore_jobs(backup_run_id, status)`.
7. `agents(server_id, state)`.
8. `agent_certificates(thumbprint)`.
9. `audit_logs(created_at)`.
10. `audit_logs(entity_type, entity_id)`.

## 15. API and Event Contracts

All new implementation endpoints must be versioned under `/api/v1`.

### 15.1 Required Agent Endpoints

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/agents/register` | TLS server auth | Submit new agent registration |
| POST | `/api/v1/agents/{id}/heartbeat` | mTLS | Report health, version, load, active jobs |
| GET | `/api/v1/agents/{id}/jobs/next` | mTLS | Pull/long-poll next assigned job |
| POST | `/api/v1/jobs/{jobId}/progress` | mTLS | Submit phase/progress update |
| POST | `/api/v1/jobs/{jobId}/logs/batch` | mTLS | Submit batched structured logs |
| POST | `/api/v1/jobs/{jobId}/complete` | mTLS | Submit final result and copy statuses |

### 15.2 Required Administrative Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/v1/agents/{id}/approve` | Approve pending certificate |
| POST | `/api/v1/agents/{id}/revoke` | Revoke active certificate |
| POST | `/api/v1/agents/{id}/replace` | Start replacement workflow |
| POST | `/api/v1/repositories` | Register repository |
| POST | `/api/v1/repositories/{id}/health-check` | Run repository health check |
| POST | `/api/v1/repositories/{id}/scrub` | Run integrity verification |
| POST | `/api/v1/backup-jobs` | Create backup job |
| POST | `/api/v1/backup-jobs/{id}/run` | Run backup now |
| GET | `/api/v1/backup-runs` | Browse backup catalog |
| POST | `/api/v1/backup-runs/{id}/retry-offline` | Retry offline replication |
| POST | `/api/v1/restore-jobs` | Create restore job |
| POST | `/api/v1/restore-jobs/{id}/dry-run` | Run restore validation |
| POST | `/api/v1/restore-jobs/{id}/execute` | Execute confirmed restore |
| POST | `/api/v1/manifests/export` | Export self-backup manifest |
| POST | `/api/v1/manifests/import` | Import manifest during recovery |
| GET | `/api/v1/reports/protection-summary` | Protection dashboard |

### 15.3 SignalR Events

SignalR events:

1. `AgentOnline`.
2. `AgentOffline`.
3. `JobStarted`.
4. `JobProgress`.
5. `JobCompleted`.
6. `JobFailed`.
7. `RepositoryUnavailable`.
8. `ProtectionStatusChanged`.
9. `OfflineMediaRequired`.
10. `RestoreStarted`.
11. `RestoreCompleted`.
12. `RestoreFailed`.

Events must include correlation ID, timestamp, entity IDs, state, and summary only. Bulk logs must use REST.

## 16. Logging, Audit, Monitoring, and Alerts

### 16.1 Structured Logs

Every log record must include:

1. Timestamp.
2. Correlation ID.
3. Job ID where applicable.
4. Agent ID where applicable.
5. Repository ID where applicable.
6. Provider.
7. Phase.
8. Severity.
9. Message.
10. Structured metadata.

### 16.2 Audit Events

Audit is required for:

1. Login and failed login.
2. Agent approval, revocation, replacement.
3. Backup job creation/modification/deletion.
4. Repository creation/modification/deletion.
5. Retention policy changes.
6. Backup execution result.
7. Restore request, dry-run, approval, execution, and result.
8. Key escrow export/import.
9. Manifest import/export.
10. Retention pruning.
11. Emergency override.

Audit logs must not be editable through normal application flows.

### 16.3 Alerts

Critical alerts:

1. No verified copy after backup.
2. Repository corruption.
3. Key escrow export failure.
4. Unauthorized restore attempt.
5. Revoked agent activity.
6. Manifest export failure.

Warnings:

1. Offline media pending.
2. Repository free space low.
3. Agent offline within grace period.
4. Backup duration unusually high.
5. Logical database backup exceeding policy threshold.

## 17. Operations and Disaster Recovery

### 17.1 Agent Onboarding

1. Install signed agent package.
2. Start Windows Service.
3. Read certificate fingerprint.
4. Approve fingerprint in dashboard.
5. Assign server/system.
6. Configure paths/databases.
7. Run health check.
8. Run first manual backup.
9. Verify repository copies.

### 17.2 Agent Update

1. Dashboard publishes signed package.
2. Agent detects update availability.
3. Agent downloads package.
4. Agent verifies digital signature.
5. Agent waits for maintenance window and idle state.
6. Agent installs silently.
7. Agent restarts.
8. Agent reconnects and reports version.
9. Failure triggers rollback or manual intervention.

No update may interrupt active backup or restore without explicit operator action.

### 17.3 Agent Replacement

1. Mark old agent as `Replacing`.
2. Suspend job dispatch to old certificate.
3. Install new agent.
4. Register new certificate.
5. Administrator maps new agent as replacement.
6. Transfer job assignments and configuration.
7. Revoke old certificate.
8. Preserve history under protected server record.
9. Run validation backup.

### 17.4 Management Server Loss

1. Provision fresh EBROP server.
2. Install compatible EBROP version.
3. Connect to verified repository.
4. Import latest system manifest.
5. Restore key escrow package.
6. Validate ability to decrypt test artifact.
7. Rebuild catalog and repository mappings.
8. Re-trust or replace agents.
9. Resume scheduling only after validation.

### 17.5 Repository Loss

1. Mark repository unavailable.
2. Stop writing new copies to it.
3. Recalculate protection status.
4. Promote secondary repository if policy permits.
5. Provision replacement repository.
6. Recreate missing copies from verified source copy where possible.
7. Run scrub after recovery.

### 17.6 Key Recovery

1. Import system manifest.
2. Locate escrow package.
3. Validate escrow checksum.
4. Restore certificate/KEK material.
5. Unlock KEK through approved procedure.
6. Test-decrypt sample artifact.
7. Audit recovery event.

## 18. Testing Strategy and Acceptance

Version 1 must not be accepted without tests covering:

1. Agent registration and approval.
2. Rejected unapproved agent.
3. Revoked certificate cannot report success.
4. File backup with VSS.
5. File backup VSS failure policy.
6. Robocopy exit code mapping.
7. SQL Server backup and `RESTORE VERIFYONLY`.
8. MySQL dump success/failure and zero-size dump failure.
9. Compression before encryption.
10. Repository copy checksum verification.
11. Offline media pending and retry.
12. Protection status computation.
13. Retention never deleting the last verified copy.
14. Manual restore dry-run.
15. File restore conflict handling.
16. SQL restore explicit replace/no-recovery choice.
17. Manifest export and import.
18. Key escrow recovery test.
19. Repository scrub corruption detection.
20. Agent replacement workflow.
21. Agent update does not interrupt running jobs.
22. SignalR state events and REST log batching.
23. Audit records for destructive operations.

Every test must verify expected state, logs, audit entries, and dashboard-visible status.

## 19. Roadmap

Post-Version 1 enhancements:

1. SQL Server differential backups.
2. SQL Server transaction log backups.
3. Point-in-time restore.
4. Percona XtraBackup.
5. MySQL/MariaDB binary log recovery.
6. Automated restore orchestration.
7. Dependency-aware auto-rollback.
8. Policy engine refinements.
9. Deduplication.
10. Immutable storage where policy permits.
11. S3-compatible provider.
12. Azure Blob provider.
13. Linux agents.
14. VM image-level backup.
15. Kubernetes/container backup.
16. Plugin SDK.

Future features must not weaken Version 1 principles: local-first deployability, agent-based execution, mTLS trust, encryption before transfer, repository verification, auditability, key escrow, and restore safety.
