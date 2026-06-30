# System Architecture

## 1. Architectural Overview

EBROP is an orchestration platform. The central server makes policy decisions, schedules work, tracks state and exposes the administrative interface. The protected servers perform backup and restore work locally through agents.

The system is divided into these major components:

1. Dashboard and API Server
2. Scheduler and Job Processor
3. Agent Registry
4. Trusted Windows Agents
5. Backup Engine
6. Restore Engine
7. Repository Manager
8. Security and Key Management
9. Audit and Logging Service
10. Notification Service

## 2. Logical Topology

```text
Administrator
     ↓
Dashboard / API Server
     ↓
Scheduler + Job Queue
     ↓
Trusted Agent over mTLS
     ↓
Native Providers: VSS, robocopy, SQL Server, MySQL
     ↓
Compressed + Encrypted Artifact
     ↓
Primary Repository → Secondary Repository → Offline Repository
```

## 3. Component Responsibilities

### Dashboard/API Server

The server provides user interface, API endpoints, authentication, authorization, job management, reporting, logs and system configuration. It does not perform privileged backup actions directly.

### Scheduler

The scheduler owns execution windows, recurring jobs, retries, dependency ordering, pruning jobs and repository scrubbing. The scheduler must be persistent, meaning jobs survive application restarts.

### Job Queue

The queue stores jobs that are waiting, running, failed, completed or stalled. Jobs are addressed to specific agents and protected resources.

### Agent Registry

The registry stores agent identity, certificate thumbprint, state, version, hostname, IP address, assigned systems, last heartbeat and replacement history.

### Windows Agent

The agent is a Windows Service installed on each protected server. It performs local backup and restore operations, including VSS snapshots, robocopy, SQL Server backup, MySQL dump, compression, encryption and repository transfer.

### Repository Manager

The repository manager abstracts storage targets and tracks backup copies independently. It calculates protection status from verified copies.

### Security Engine

The security layer manages mTLS, role-based access, certificate approval, key wrapping, key escrow, audit logging and recovery credentials.

## 4. Trust Boundaries

EBROP has four major trust boundaries:

1. User-to-dashboard boundary.
2. Dashboard-to-agent boundary.
3. Agent-to-local-resource boundary.
4. Agent-to-repository boundary.

Each boundary must be explicitly secured.

### User-to-Dashboard

Authenticated users access the dashboard through HTTPS. Role-based authorization determines which operations they can perform.

### Dashboard-to-Agent

All communication uses mTLS. Only approved agents with trusted certificate fingerprints can receive jobs or report results.

### Agent-to-Local-Resource

The agent runs under a controlled Windows service account. The account must have only the permissions needed for configured backup and restore operations.

### Agent-to-Repository

Agents write encrypted backup packages to repositories. Repository permissions must prevent ordinary application users from modifying backup contents.

## 5. Deployment Topology

Recommended deployment:

```text
EBROP Dashboard Server
    - ASP.NET Core API
    - Scheduler
    - SignalR hub
    - Metadata database connection

Metadata Database Server
    - SQL Server or MySQL/MariaDB
    - Stores configuration, catalog, audit and job state

Repository Server 01
    - Primary repository

Repository Server 02
    - Secondary repository

Protected Servers
    - Windows Agent installed
    - Local backup staging folder
```

The dashboard server and primary repository should not be the same physical server in production.

## 6. Data Flow: Backup

1. Scheduler selects a due backup job.
2. Job Processor validates policy and target agent.
3. Job is assigned to the agent.
4. Agent performs pre-flight checks.
5. Agent executes backup provider.
6. Agent calculates source checksum.
7. Agent compresses artifact.
8. Agent encrypts artifact.
9. Agent transfers artifact to primary repository.
10. Agent verifies repository copy.
11. Agent repeats replication for secondary repositories.
12. Scheduler evaluates retention.
13. Audit and metrics are published.

## 7. Data Flow: Restore

1. Administrator selects backup copy or restore point.
2. System performs dry-run validation.
3. Administrator confirms restore options.
4. Restore job is assigned to the correct agent.
5. Agent validates space, target path and database state.
6. Agent retrieves encrypted artifact.
7. Agent decrypts and decompresses artifact in staging.
8. Agent runs provider-specific restore.
9. Agent reports result.
10. Health checks and audit events are recorded.

## 8. State Management

The platform must persist state at multiple levels:

- Job state.
- Agent state.
- Repository state.
- Backup run state.
- Backup copy state.
- Restore job state.
- Key escrow state.
- Manifest export state.

State transitions must be auditable.

## 9. Failure Handling

### Agent Offline

Pending jobs remain queued until the agent reconnects or a timeout threshold is reached. Critical alerts are raised only after policy-defined grace periods.

### Repository Full

Backup copy fails for that repository. If at least one required repository succeeds, the run may become Partially Protected. If no repository succeeds, the run is Unprotected or Failed.

### Checksum Mismatch

The copy is rejected. The system retries based on policy. If retries fail, the backup copy is marked Corrupted or Failed.

### Dashboard Loss

A fresh EBROP instance can import the latest self-backup manifest from a repository, recover catalog metadata and re-trust or replace agents.

## 10. Scaling Model

Version 1 should support dozens to hundreds of agents. To scale safely:

- Agents perform work locally.
- SignalR carries state changes, not raw logs.
- Bulk logs use REST ingestion or batched uploads.
- Replication defaults to sequential copy.
- Agent concurrency limits protect production workloads.
- Scheduler uses persistent storage to avoid losing due jobs.

## 11. Security Posture

The platform assumes that internal networks are not automatically trusted. Backup artifacts are encrypted before transfer. Agents must prove identity. Administrators must be authorized. Destructive actions must be audited.

## 12. Design Constraints

- Version 1 is Windows-first.
- Version 1 is local-first.
- Version 1 does not include VM image backup.
- Version 1 does not rely on public cloud storage.
- Version 1 must support manual restore execution.
- Version 1 must support catalog recovery.
