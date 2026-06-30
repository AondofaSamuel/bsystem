# Storage and Replication

## 1. Purpose

The Storage and Replication subsystem stores backup artifacts, verifies copies, tracks repository health, applies retention, and provides protection status.

Version 1 is local-first. Cloud storage is optional future work.

## 2. Repository Types

Supported in Version 1:

| Type | Example | Notes |
|---|---|---|
| LocalDisk | `D:\Backups` | Suitable for small deployments or staging. |
| UNCShare | `\\BACKUP01\Backups` | Primary local network repository. |
| NAS | `\\NAS01\EBROP` | Recommended for larger storage. |
| SecondaryServer | `\\BACKUP02\Backups` | Resilience against primary repository loss. |
| OfflineMedia | `E:\MonthlyBackups` | Air-gapped protection when disconnected. |

## 3. Repository Metadata

Each repository records:

- Repository ID.
- Name.
- Type.
- Path.
- Required/optional status.
- Capacity.
- Free space.
- Health status.
- Last successful write.
- Last scrub date.
- Bandwidth limit.
- Concurrent copy limit.
- Retention policy.

## 4. Backup Copies

Each backup run may produce multiple backup copies.

A copy records:

- Backup run ID.
- Repository ID.
- Copy path.
- Copy status.
- Package checksum.
- Verification result.
- Replication start time.
- Replication completion time.
- Error message.

## 5. Protection Status

Overall protection status is computed from repository copy status.

| Status | Meaning |
|---|---|
| Protected | All required repositories have verified copies. |
| PartiallyProtected | At least one required copy is missing or pending, but one verified copy exists. |
| Unprotected | No verified repository copy exists. |
| Failed | Backup creation or all required copies failed. |

## 6. Replication Workflow

1. Agent completes local backup package.
2. Agent verifies package checksum.
3. Agent copies package to primary repository.
4. Agent verifies primary repository copy.
5. Agent copies package to secondary repository.
6. Agent verifies secondary copy.
7. Agent marks offline copy pending if media unavailable.
8. Scheduler recalculates protection status.

Default replication is sequential to reduce network congestion.

## 7. Offline Repository Workflow

Offline repositories are expected to be unavailable most of the time.

When offline media is missing:

- Copy status becomes PendingOfflineMedia.
- Protection status may become PartiallyProtected.
- Notification severity is Warning, not Critical, unless policy says offline copy is overdue.

When media is connected:

1. Administrator opens pending backup copy.
2. Administrator selects Retry Offline Replication.
3. Agent verifies media identity.
4. Agent copies pending artifacts.
5. Agent verifies checksum.
6. Copy status becomes Success.

## 8. Retention and Pruning

Retention is included in Version 1.

Default policy:

- Hourly: 48 hours.
- Daily: 30 days.
- Weekly: 12 weeks.
- Monthly: 12 months.
- Yearly: configurable.

Pruning rules:

- Never delete the last verified copy.
- Never delete backups under legal hold.
- Never delete backups needed for restore chain continuity.
- Always audit deletion.
- Prefer deleting optional copies before required copies if storage pressure exists.

## 9. Repository Scrubbing

Repository scrubbing detects bit-rot and silent corruption.

Monthly scrub flow:

1. Select candidate backup copies.
2. Read package from repository.
3. Recalculate checksum.
4. Compare with anchored checksum.
5. Mark copy Healthy or Corrupted.
6. Alert on mismatch.

A corrupted copy must not be used for restore unless an administrator explicitly overrides policy.

## 10. Repository Failure Handling

### Repository Unavailable

The copy is marked failed or pending depending on repository type. Required repository failures affect protection status.

### Repository Full

The system should run retention evaluation first. If space remains insufficient, backup copy fails with RepositoryFull.

### Checksum Mismatch

The copy is rejected. A retry may be attempted. Persistent mismatch marks the copy Corrupted.

## 11. Repository Security

Repositories should store encrypted packages only. Repository permissions must prevent ordinary application users from modifying backup data.

Recommended approach:

- Dedicated service account for agent writes.
- Read-only access for restore where possible.
- Separate admin permissions from application permissions.
- Audit repository deletions.

## 12. Future Storage Providers

Future providers may include:

- S3-compatible object storage.
- Azure Blob.
- Immutable/WORM storage.
- Tape library integration.

These must be added through the repository provider interface, not hardcoded into backup logic.
