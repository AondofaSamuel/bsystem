# Restore Engine

## 1. Purpose

The Restore Engine restores files, databases and application components from verified backup artifacts. Restore is a first-class capability in Version 1.

Version 1 supports manual restore execution. Later versions add automated maintenance mode, rollback orchestration, dependency sequencing and health-driven validation.

## 2. Restore Modes

EBROP supports the following restore modes:

| Mode | Description |
|---|---|
| Files Only | Restore selected folders or files. |
| Database Only | Restore SQL Server or MySQL/MariaDB database backup. |
| Full Application | Restore files and database together. |
| Test Restore | Restore to isolated/test target. |
| Production Restore | Restore to production target with stricter approval. |

## 3. Manual Restore in Version 1

Manual restore flow:

1. Administrator browses backup catalog.
2. Administrator selects restore point.
3. Administrator selects restore target.
4. System performs dry-run validation.
5. Administrator chooses conflict behavior.
6. Administrator confirms operation.
7. Job is dispatched to the responsible agent.
8. Agent retrieves artifact.
9. Agent decrypts and decompresses artifact in staging.
10. Provider performs restore.
11. Agent reports result.
12. Audit entry is recorded.

In Version 1 the administrator may manually stop services or enable maintenance mode before executing restore.

## 4. Pre-Restore Safety Checks

The agent must validate:

- Destination disk space.
- Restore target accessibility.
- Target path existence.
- Existing file/folder conflict.
- Database existence.
- Required privileges.
- Backup artifact integrity.
- Decryption key availability.
- Repository availability.

No restore may proceed if mandatory safety checks fail.

## 5. Conflict Handling

If target path exists, the administrator must choose one:

- Overwrite existing target.
- Rename existing target.
- Restore to alternate path.
- Cancel.

The system must never silently overwrite a production path.

## 6. Database Restore Safety

If database exists, the administrator must explicitly choose behavior.

SQL Server options include:

- Restore with REPLACE.
- Restore with NORECOVERY.
- Restore to new database name.
- Cancel.

The selected option must be recorded in the audit log.

## 7. Dry Run

Dry Run performs all validations without modifying the target.

Dry Run result includes:

- Required disk space.
- Available disk space.
- Target conflict status.
- Required credentials.
- Artifact status.
- Repository status.
- Expected restore actions.
- Blocking issues.

## 8. File Restore

File restore uses robocopy or controlled extraction from decrypted artifact.

Recommended restore behavior:

- Restore to staging first.
- Verify staging contents.
- Apply chosen target conflict policy.
- Copy to final destination.
- Reapply ACLs where applicable.

## 9. SQL Server Restore

Example restore pattern:

```sql
ALTER DATABASE [DatabaseName] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;

RESTORE DATABASE [DatabaseName]
FROM DISK = N'D:\Staging\DatabaseName.bak'
WITH REPLACE, RECOVERY, CHECKSUM, STATS = 10;

ALTER DATABASE [DatabaseName] SET MULTI_USER;
```

The system must not assume WITH REPLACE. It must be selected explicitly.

## 10. MySQL/MariaDB Restore

Logical restore pattern:

```cmd
mysql database_name < backup.sql
```

Recommended safer option:

- Restore to temporary database first.
- Validate object count and basic queries.
- Swap or promote after administrator confirmation.

## 11. Automated Restore Roadmap

Later versions automate:

- Maintenance mode.
- IIS App Pool stop/start.
- Windows Service stop/start.
- Rollback backup creation.
- Dependency ordering.
- Health validation.
- Auto-rollback on failure.

## 12. Restore Audit Requirements

Every restore must record:

- Requesting user.
- Approving user where applicable.
- Restore source.
- Restore target.
- Conflict choice.
- Database overwrite choice.
- Start and end time.
- Agent ID.
- Result.
- Error details.
