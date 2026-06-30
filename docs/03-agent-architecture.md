# Agent Architecture

## 1. Purpose

The EBROP Agent is a C# Windows Service installed on each protected server. It is the only component allowed to execute privileged backup and restore operations on that server.

The agent exists to avoid unsafe remote execution from the dashboard. It also localizes credentials, reduces network dependency, and allows backup logic to interact with Windows-native APIs.

## 2. Agent Responsibilities

The agent shall:

- Register securely with the orchestration server.
- Maintain mTLS identity.
- Receive jobs addressed to its Agent ID.
- Validate jobs before execution.
- Execute file, SQL Server and MySQL/MariaDB backup providers.
- Execute manual restore jobs.
- Manage local staging directories.
- Compress and encrypt artifacts locally.
- Replicate artifacts to configured repositories.
- Perform checksum verification.
- Report progress, logs, metrics and final job status.
- Enforce local concurrency, CPU, I/O and network limits.
- Support signed updates and replacement workflow.

## 3. Agent Process Model

The agent runs as a Windows Service using .NET Worker Service infrastructure.

Recommended internal services:

| Service | Responsibility |
|---|---|
| ConnectionManager | mTLS session, reconnects, heartbeat. |
| JobReceiver | Receives and validates assigned jobs. |
| JobProcessor | Dispatches work to providers. |
| BackupEngine | Coordinates backup pipeline. |
| RestoreEngine | Coordinates restore pipeline. |
| RepositoryManager | Writes and verifies copies. |
| HealthMonitor | Reports server, service and repository health. |
| UpdateManager | Handles signed agent updates. |
| AuditPublisher | Publishes audit events. |
| LocalStateStore | Persists in-flight job state. |

## 4. Agent State Machine

Valid states:

- PendingApproval
- Approved
- Online
- Idle
- RunningBackup
- Replicating
- WaitingForOfflineMedia
- RunningRestore
- Maintenance
- Updating
- Replacing
- Revoked
- Retired

Agents must never execute jobs while PendingApproval, Revoked or Retired.

## 5. Registration Flow

1. Agent is installed on the protected server.
2. Agent generates a key pair and self-signed certificate.
3. Agent displays or logs certificate fingerprint.
4. Agent contacts dashboard registration endpoint.
5. Dashboard records the agent as PendingApproval.
6. Administrator verifies fingerprint and approves.
7. Dashboard marks certificate as trusted.
8. Agent reconnects using mTLS.
9. Agent becomes Approved and Online.

## 6. Job Acceptance Rules

Before accepting a job, the agent verifies:

- Job is addressed to its Agent ID.
- Job signature or server identity is valid.
- Agent certificate is not revoked.
- Required provider exists.
- Required tools are installed.
- Local policy permits execution at that time.
- Concurrency limits allow the job to run.
- Protected resource is not already locked by another job.

## 7. Local State Store

The agent must persist enough state to recover after restart.

Persisted items:

- Current job ID.
- Job type.
- Execution phase.
- Staging path.
- Artifact path.
- Repository copy progress.
- Error state.
- Last heartbeat.

A restarted agent must never silently assume a job succeeded. It must resume where safe, otherwise fail the job with a recoverable error.

## 8. Update Mechanism

Agent updates are controlled and signed.

Update flow:

1. Dashboard publishes new signed MSI/package.
2. Agent detects update availability.
3. Agent verifies package signature.
4. Agent waits for maintenance window and idle state.
5. Agent installs update silently.
6. Agent restarts.
7. Agent reconnects and reports new version.
8. If update fails, rollback procedure is executed where possible.

The system must not update an agent while a backup or restore job is running.

## 9. Replacement Workflow

When a server is rebuilt or replaced:

1. Administrator marks old agent as Replacing.
2. Old certificate is suspended from receiving jobs.
3. New agent is installed on replacement server.
4. New certificate is approved as replacement for the old agent.
5. Job assignments and configuration are transferred.
6. Old certificate is revoked.
7. History remains linked to the protected server.

## 10. Resource Controls

Each agent supports:

- Maximum concurrent jobs.
- Maximum concurrent repository transfers.
- Bandwidth throttle.
- Compression thread limit.
- Process priority.
- Business-hours low-impact mode.
- Maintenance-window high-performance mode.

Default behavior must favor production workloads over backup throughput.

## 11. Security Restrictions

The agent must not expose a general command execution endpoint. It accepts structured job definitions only. All paths, databases and repositories must be pre-registered or explicitly authorized by policy.

## 12. Logging

Agent logs must be structured and correlated.

Required fields:

- Timestamp.
- Correlation ID.
- Job ID.
- Agent ID.
- Provider.
- Phase.
- Severity.
- Message.
- Metadata.

High-volume logs should be batched. SignalR is reserved for state transitions and summary progress.
