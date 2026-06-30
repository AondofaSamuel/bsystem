# API Contracts

## 1. Purpose

The EBROP API exposes orchestration, configuration, job, repository, agent, backup and restore operations. APIs are used by the dashboard, agents and administrative integrations.

All APIs must be authenticated and authorized.

## 2. API Groups

- Authentication.
- Users and roles.
- Systems.
- Servers.
- Agents.
- Repositories.
- Backup jobs.
- Backup runs.
- Restore jobs.
- Manifests.
- Audit logs.
- Health.

## 3. Agent Registration

### POST `/api/agents/register`

Registers a new agent as PendingApproval.

Request:

```json
{
  "hostname": "ECMS-SERVER-01",
  "ipAddress": "172.16.24.66",
  "agentVersion": "1.0.0",
  "certificateThumbprint": "ABCD1234",
  "certificateSubject": "CN=ECMS-SERVER-01"
}
```

Response:

```json
{
  "agentId": 12,
  "state": "PendingApproval"
}
```

## 4. Agent Heartbeat

### POST `/api/agents/{agentId}/heartbeat`

Request:

```json
{
  "state": "Idle",
  "cpuPercent": 12.5,
  "memoryPercent": 48.0,
  "freeDiskBytes": 91398219221,
  "runningJobs": 0
}
```

## 5. Backup Job Creation

### POST `/api/backup-jobs`

```json
{
  "systemId": 1,
  "agentId": 12,
  "name": "Daily ECMS Database",
  "jobType": "MSSQL_FULL",
  "scheduleExpression": "0 2 * * *",
  "retentionPolicyId": 3,
  "repositories": [1, 2]
}
```

## 6. Dispatch Job to Agent

### GET `/api/agents/{agentId}/jobs/next`

The agent may use a pull model where allowed, but the connection must still be authenticated with mTLS. Long-polling or queue-backed dispatch is acceptable for Version 1.

Response:

```json
{
  "jobId": 981,
  "jobType": "FILE_BACKUP",
  "systemId": 1,
  "resource": {
    "path": "C:\\inetpub\\wwwroot\\ECMS",
    "useVss": true
  },
  "repositories": [
    { "repositoryId": 1, "path": "\\\\BACKUP01\\Backups\\ECMS" },
    { "repositoryId": 2, "path": "\\\\BACKUP02\\Backups\\ECMS" }
  ],
  "compression": { "enabled": true, "level": "balanced" },
  "encryption": { "enabled": true }
}
```

## 7. Job Progress

### POST `/api/jobs/{jobId}/progress`

```json
{
  "agentId": 12,
  "phase": "Replicating",
  "percent": 65,
  "message": "Copying package to secondary repository"
}
```

## 8. Job Completion

### POST `/api/jobs/{jobId}/complete`

```json
{
  "agentId": 12,
  "status": "Success",
  "sourceChecksum": "sha256:...",
  "packageChecksum": "sha256:...",
  "sizeBytes": 883928331,
  "copies": [
    {
      "repositoryId": 1,
      "status": "Success",
      "checksum": "sha256:...",
      "copyPath": "\\\\BACKUP01\\Backups\\ECMS\\artifact.ebrop"
    }
  ]
}
```

## 9. Manual Restore Request

### POST `/api/restore-jobs`

```json
{
  "backupRunId": 1001,
  "restoreType": "DatabaseOnly",
  "targetAgentId": 12,
  "targetDatabase": "ECMS_Restore_Test",
  "conflictPolicy": "RestoreToNewName",
  "dryRun": true
}
```

## 10. SignalR Events

SignalR should carry state transitions, not raw logs.

Events:

- AgentOnline
- AgentOffline
- JobStarted
- JobProgress
- JobCompleted
- JobFailed
- RepositoryUnavailable
- ProtectionStatusChanged
- OfflineMediaRequired
- RestoreStarted
- RestoreCompleted

Example:

```json
{
  "event": "ProtectionStatusChanged",
  "backupRunId": 1001,
  "status": "PartiallyProtected"
}
```

## 11. Error Model

All API errors should follow a common contract.

```json
{
  "errorCode": "RepositoryUnavailable",
  "message": "Primary repository could not be reached.",
  "correlationId": "01J...",
  "details": {}
}
```

## 12. API Security

- User APIs use normal authenticated sessions or bearer tokens.
- Agent APIs require mTLS.
- Restore APIs require explicit authorization.
- Key escrow APIs require elevated privileges and audit.

## 13. Versioning

API URLs should include versioning once external integrations exist:

```text
/api/v1/backup-jobs
```

Internal Version 1 may start without explicit URL versioning but should keep compatibility in mind.
