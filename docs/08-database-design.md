# Database Design

## 1. Purpose

The EBROP database stores configuration, identity, job state, backup catalog, repository copies, restore jobs, audit logs, retention policies and system recovery metadata.

The database is not the backup repository. It is the control-plane catalog.

## 2. Core Entities

Core entities:

- users
- roles
- systems
- servers
- agents
- agent_certificates
- repositories
- backup_jobs
- backup_runs
- backup_copies
- restore_jobs
- retention_policies
- job_logs
- audit_logs
- system_manifests
- encryption_keys
- key_escrow_packages

## 3. Systems

A system represents an application or workload.

Fields:

```sql
CREATE TABLE systems (
    id BIGINT PRIMARY KEY IDENTITY,
    name NVARCHAR(150) NOT NULL,
    code NVARCHAR(50) NOT NULL UNIQUE,
    description NVARCHAR(MAX) NULL,
    environment NVARCHAR(30) NOT NULL,
    rpo_minutes INT NULL,
    rto_minutes INT NULL,
    status NVARCHAR(30) NOT NULL,
    created_at DATETIME2 NOT NULL,
    updated_at DATETIME2 NULL
);
```

## 4. Servers

```sql
CREATE TABLE servers (
    id BIGINT PRIMARY KEY IDENTITY,
    hostname NVARCHAR(150) NOT NULL,
    ip_address NVARCHAR(50) NULL,
    os_type NVARCHAR(50) NOT NULL,
    location NVARCHAR(150) NULL,
    status NVARCHAR(30) NOT NULL,
    created_at DATETIME2 NOT NULL,
    updated_at DATETIME2 NULL
);
```

## 5. Agents

```sql
CREATE TABLE agents (
    id BIGINT PRIMARY KEY IDENTITY,
    server_id BIGINT NOT NULL,
    agent_name NVARCHAR(150) NOT NULL,
    agent_version NVARCHAR(50) NULL,
    state NVARCHAR(50) NOT NULL,
    last_seen_at DATETIME2 NULL,
    approved_by BIGINT NULL,
    approved_at DATETIME2 NULL,
    replaced_by_agent_id BIGINT NULL,
    created_at DATETIME2 NOT NULL
);
```

## 6. Agent Certificates

```sql
CREATE TABLE agent_certificates (
    id BIGINT PRIMARY KEY IDENTITY,
    agent_id BIGINT NOT NULL,
    thumbprint NVARCHAR(200) NOT NULL,
    subject NVARCHAR(300) NULL,
    valid_from DATETIME2 NULL,
    valid_to DATETIME2 NULL,
    state NVARCHAR(30) NOT NULL,
    revoked_at DATETIME2 NULL,
    created_at DATETIME2 NOT NULL
);
```

## 7. Repositories

```sql
CREATE TABLE repositories (
    id BIGINT PRIMARY KEY IDENTITY,
    name NVARCHAR(150) NOT NULL,
    repository_type NVARCHAR(50) NOT NULL,
    path NVARCHAR(1000) NOT NULL,
    is_required BIT NOT NULL DEFAULT 1,
    health_status NVARCHAR(50) NOT NULL,
    capacity_bytes BIGINT NULL,
    free_bytes BIGINT NULL,
    bandwidth_limit_kbps INT NULL,
    max_concurrent_copies INT NULL,
    last_checked_at DATETIME2 NULL,
    created_at DATETIME2 NOT NULL
);
```

## 8. Backup Jobs

```sql
CREATE TABLE backup_jobs (
    id BIGINT PRIMARY KEY IDENTITY,
    system_id BIGINT NOT NULL,
    agent_id BIGINT NOT NULL,
    name NVARCHAR(150) NOT NULL,
    job_type NVARCHAR(50) NOT NULL,
    schedule_expression NVARCHAR(150) NULL,
    retention_policy_id BIGINT NOT NULL,
    is_enabled BIT NOT NULL DEFAULT 1,
    created_by BIGINT NULL,
    created_at DATETIME2 NOT NULL,
    updated_at DATETIME2 NULL
);
```

## 9. Backup Runs

```sql
CREATE TABLE backup_runs (
    id BIGINT PRIMARY KEY IDENTITY,
    backup_job_id BIGINT NOT NULL,
    system_id BIGINT NOT NULL,
    agent_id BIGINT NOT NULL,
    status NVARCHAR(50) NOT NULL,
    protection_status NVARCHAR(50) NOT NULL,
    artifact_name NVARCHAR(300) NULL,
    source_checksum NVARCHAR(200) NULL,
    package_checksum NVARCHAR(200) NULL,
    size_bytes BIGINT NULL,
    started_at DATETIME2 NULL,
    completed_at DATETIME2 NULL,
    error_message NVARCHAR(MAX) NULL,
    created_at DATETIME2 NOT NULL
);
```

## 10. Backup Copies

```sql
CREATE TABLE backup_copies (
    id BIGINT PRIMARY KEY IDENTITY,
    backup_run_id BIGINT NOT NULL,
    repository_id BIGINT NOT NULL,
    copy_path NVARCHAR(1000) NULL,
    status NVARCHAR(50) NOT NULL,
    checksum NVARCHAR(200) NULL,
    verified_at DATETIME2 NULL,
    replicated_at DATETIME2 NULL,
    error_message NVARCHAR(MAX) NULL,
    created_at DATETIME2 NOT NULL
);
```

## 11. Restore Jobs

```sql
CREATE TABLE restore_jobs (
    id BIGINT PRIMARY KEY IDENTITY,
    backup_run_id BIGINT NOT NULL,
    requested_by BIGINT NOT NULL,
    approved_by BIGINT NULL,
    agent_id BIGINT NOT NULL,
    restore_type NVARCHAR(50) NOT NULL,
    target_path NVARCHAR(1000) NULL,
    target_database NVARCHAR(150) NULL,
    conflict_policy NVARCHAR(50) NULL,
    status NVARCHAR(50) NOT NULL,
    dry_run_result NVARCHAR(MAX) NULL,
    started_at DATETIME2 NULL,
    completed_at DATETIME2 NULL,
    error_message NVARCHAR(MAX) NULL,
    created_at DATETIME2 NOT NULL
);
```

## 12. Audit Logs

```sql
CREATE TABLE audit_logs (
    id BIGINT PRIMARY KEY IDENTITY,
    actor_user_id BIGINT NULL,
    actor_agent_id BIGINT NULL,
    action NVARCHAR(150) NOT NULL,
    entity_type NVARCHAR(100) NULL,
    entity_id BIGINT NULL,
    ip_address NVARCHAR(100) NULL,
    user_agent NVARCHAR(500) NULL,
    details NVARCHAR(MAX) NULL,
    created_at DATETIME2 NOT NULL
);
```

## 13. System Manifests

```sql
CREATE TABLE system_manifests (
    id BIGINT PRIMARY KEY IDENTITY,
    manifest_version NVARCHAR(50) NOT NULL,
    manifest_path NVARCHAR(1000) NULL,
    checksum NVARCHAR(200) NOT NULL,
    exported_at DATETIME2 NOT NULL,
    exported_by BIGINT NULL
);
```

## 14. Indexing Guidance

Recommended indexes:

- backup_runs(backup_job_id, started_at)
- backup_runs(system_id, status)
- backup_copies(backup_run_id, repository_id)
- restore_jobs(backup_run_id, status)
- agents(server_id, state)
- audit_logs(created_at)
- audit_logs(entity_type, entity_id)

## 15. Data Retention

Audit logs should have longer retention than operational logs. Backup catalog retention should follow backup retention policy, but deleted backup metadata should remain as audit history.
