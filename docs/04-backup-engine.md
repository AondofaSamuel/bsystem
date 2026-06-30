# Backup Engine

## 1. Purpose

The Backup Engine creates verified, compressed, encrypted backup artifacts from configured protected resources. It runs inside the Windows Agent and is invoked only through validated jobs from the orchestration server.

## 2. Backup Pipeline

The standard backup pipeline is:

1. Load job definition.
2. Validate policy.
3. Run pre-flight checks.
4. Acquire resource lock.
5. Prepare local staging directory.
6. Create VSS snapshot if required.
7. Execute provider backup.
8. Calculate source checksum.
9. Compress artifact.
10. Encrypt artifact.
11. Calculate package checksum.
12. Replicate to repositories.
13. Verify repository copies.
14. Cleanup staging based on policy.
15. Publish audit events.

## 3. Provider Interface

Each backup provider implements the same contract:

```csharp
public interface IBackupProvider
{
    Task<ValidationResult> ValidateAsync(BackupJobContext context);
    Task<BackupResult> ExecuteBackupAsync(BackupJobContext context);
    Task<VerificationResult> VerifyAsync(BackupArtifact artifact);
    Task<RestorePlan> CreateRestorePlanAsync(BackupArtifact artifact, RestoreTarget target);
    Task CleanupAsync(BackupJobContext context);
}
```

Providers must not decide scheduling, retention or repository policy. They only perform resource-specific backup and verification.

## 4. File Backup Provider

The File Provider protects folders such as application source, upload folders, media folders and configuration directories.

Recommended robocopy flags:

```cmd
robocopy "SOURCE" "DESTINATION" /MIR /COPYALL /DCOPY:DAT /R:2 /W:2 /MT:16 /FFT
```

Where appropriate:

```cmd
/ZB
```

or:

```cmd
/B
```

### VSS Usage

VSS should be used when the protected folder may contain locked or active files. The agent creates a snapshot, exposes the snapshot path, runs robocopy against the snapshot, then releases the snapshot.

If VSS is required by policy and VSS creation fails, the backup must fail unless the policy explicitly permits fallback.

## 5. SQL Server Backup Provider

SQL Server backup uses native SQL Server commands, not file copying of MDF/LDF files.

Full backup example:

```sql
BACKUP DATABASE [DatabaseName]
TO DISK = N'D:\Staging\DatabaseName_full.bak'
WITH INIT, COMPRESSION, CHECKSUM, STATS = 10;
```

Verification:

```sql
RESTORE VERIFYONLY
FROM DISK = N'D:\Staging\DatabaseName_full.bak'
WITH CHECKSUM;
```

Version 1 supports full backups and manual restore. Later versions add differential, transaction log backup and point-in-time restore.

Windows Authentication is preferred. SQL Authentication is supported only where necessary and credentials must be encrypted.

## 6. MySQL/MariaDB Backup Provider

Version 1 uses logical backup:

```cmd
mysqldump --single-transaction --routines --triggers --events --quick database_name
```

The provider must capture stderr, exit code, dump size and duration. A zero-size dump is always failure.

For large databases, the roadmap includes Percona XtraBackup and binary log based point-in-time recovery.

## 7. Checksum Anchoring

Checksums must be anchored before repository replication. The agent calculates a source checksum before transfer and stores it in the metadata database. Repository copies are verified against the anchored checksum or package checksum depending on verification stage.

Recommended hashes:

- SHA-256 minimum.
- SHA-512 optional.

## 8. Compression

Compression occurs before encryption. Providers output to a staging area. The Compression Provider packages the artifact into a compressed archive.

Compression settings:

- Compression enabled by default.
- Compression level configurable per policy.
- Compression thread count limited by agent configuration.

## 9. Encryption

Encryption occurs after compression. Artifacts are encrypted with a Data Encryption Key. The DEK is wrapped by a Key Encryption Key controlled by the Security Engine.

The repository should never receive unencrypted production data.

## 10. Error Handling

Backup errors are classified as:

- ConfigurationError
- PreflightError
- ProviderExecutionError
- VerificationError
- CompressionError
- EncryptionError
- ReplicationError
- RetentionError

Each failure must include enough context for the administrator to act.

## 11. Idempotency

Backup jobs should be safe to retry. If a previous attempt produced a partial staging artifact, the next attempt must either resume safely or clean the staging area before restarting.

## 12. Exit Code Mapping

Robocopy exit codes 0 through 7 are not automatically failure. Exit code 8 and above indicates failure.

SQL and MySQL tools follow normal process exit conventions unless provider-specific behavior is documented.
