# Operations and Disaster Recovery

## 1. Purpose

This document describes how EBROP is installed, operated, monitored, recovered and maintained.

## 2. Installation Overview

Recommended order:

1. Provision dashboard server.
2. Provision metadata database.
3. Provision primary repository.
4. Provision secondary repository.
5. Configure TLS certificates.
6. Install EBROP dashboard/API.
7. Configure initial administrator.
8. Configure repositories.
9. Install agent on first protected server.
10. Approve agent certificate.
11. Create first backup job.
12. Run test backup.
13. Run test restore.

## 3. Agent Onboarding Runbook

1. Install agent package on server.
2. Confirm service starts.
3. Retrieve certificate fingerprint.
4. Open dashboard agent approval screen.
5. Verify hostname, IP and fingerprint.
6. Approve agent.
7. Assign protected system.
8. Configure paths/databases.
9. Run health check.
10. Run first backup.

## 4. Repository Onboarding Runbook

1. Create repository folder/share.
2. Assign service account permissions.
3. Register repository in dashboard.
4. Run repository health check.
5. Run write/read/delete test.
6. Configure retention policy.
7. Configure bandwidth and concurrency limits.

## 5. Daily Operations

Administrators should review:

- Failed backups.
- Partially protected backups.
- Offline media pending status.
- Repository free space.
- Agent offline alerts.
- Restore jobs.
- Scrubber alerts.
- Key escrow status.

## 6. Weekly Operations

Weekly tasks:

- Review backup compliance report.
- Connect offline media and run pending replication.
- Confirm repository capacity trend.
- Review failed retries.
- Confirm agents are up to date.

## 7. Monthly Operations

Monthly tasks:

- Run repository scrubbing.
- Test restore to isolated target.
- Review audit logs.
- Test manifest export/import process.
- Confirm key escrow package is recoverable.

## 8. Dashboard Loss Recovery

If EBROP dashboard server is lost:

1. Provision fresh server.
2. Install EBROP.
3. Connect to new or restored metadata database.
4. Locate latest system manifest in repository.
5. Import manifest.
6. Restore key escrow package.
7. Validate ability to decrypt test artifact.
8. Re-trust or replace agents as required.
9. Resume scheduling.

## 9. Agent Replacement Runbook

1. Mark old agent as Replacing.
2. Install new agent on replacement server.
3. Approve new certificate as replacement.
4. Transfer system assignments.
5. Validate paths and databases.
6. Run health check.
7. Revoke old certificate.
8. Run test backup.

## 10. Repository Loss Runbook

If primary repository fails:

1. Mark repository unavailable.
2. Stop writing new copies to it.
3. Promote secondary repository where policy permits.
4. Restore repository from verified secondary/offline copy if possible.
5. Recalculate protection status for affected backups.
6. Run repository scrubbing after recovery.

## 11. Key Recovery Runbook

1. Install fresh EBROP if required.
2. Import system manifest.
3. Locate escrow package.
4. Restore certificate/KEK material.
5. Validate checksum of escrow package.
6. Attempt decrypt of test artifact.
7. Record recovery event in audit log.

## 12. Restore Test Runbook

1. Select recent backup.
2. Select isolated restore target.
3. Run dry-run validation.
4. Execute restore.
5. Validate application health.
6. Record restore duration.
7. Compare against RTO.
8. Document issues.

## 13. Monitoring Metrics

Metrics:

- Backup success rate.
- Restore success rate.
- Average backup duration.
- Repository free space.
- Agent online percentage.
- Failed copy count.
- Partially protected backup count.
- Scrubber corruption count.
- Retention pruning count.

## 14. Alerts

Critical alerts:

- No verified copy after backup.
- Key escrow failure.
- Repository corruption.
- Unauthorized restore attempt.
- Revoked agent activity.

Warnings:

- Offline media pending.
- Repository free space low.
- Agent offline within grace period.
- Backup duration unusually high.

## 15. Maintenance Windows

Backups should be scheduled to reduce production impact. Agents should support low-impact mode during business hours and higher throughput during maintenance windows.

## 16. Disaster Recovery Philosophy

The platform must assume failure of any single component:

- Dashboard can fail.
- Agent can fail.
- Repository can fail.
- Network can fail.
- Encryption material can be misplaced.
- Human operators can make mistakes.

The system is designed so these failures do not automatically become data loss.
