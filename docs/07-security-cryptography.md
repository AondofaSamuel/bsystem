# Security and Cryptography

## 1. Purpose

Security is central to EBROP because the platform controls backup and restore operations over sensitive enterprise data. A compromised backup platform can cause data loss, unauthorized disclosure, or false recovery confidence.

## 2. Security Objectives

EBROP must:

- Authenticate users and agents strongly.
- Authorize destructive operations explicitly.
- Encrypt backup data before transfer.
- Protect encryption keys with escrow.
- Maintain immutable audit records.
- Prevent arbitrary command execution.
- Support certificate revocation and agent replacement.

## 3. User Authentication and RBAC

Roles:

| Role | Capabilities |
|---|---|
| SuperAdmin | Full system control. |
| BackupAdmin | Manage systems, jobs and repositories. |
| RestoreOperator | Execute approved restore jobs. |
| Auditor | View logs, reports and audit trail. |
| Viewer | Read-only dashboard access. |

Production restore should require elevated permission and may require second-person approval.

## 4. Agent Authentication

Agents authenticate using mutual TLS.

Registration process:

1. Agent generates certificate.
2. Agent submits registration request.
3. Dashboard records fingerprint.
4. Administrator verifies and approves fingerprint.
5. Agent becomes trusted.
6. Future requests require certificate validation.

Static API keys are not sufficient for agent trust.

## 5. Certificate Lifecycle

Certificate states:

- Pending.
- Approved.
- Active.
- Suspended.
- Revoked.
- Replaced.
- Expired.

Revoked certificates must not receive jobs or submit success reports.

## 6. Backup Encryption Model

EBROP uses envelope encryption.

```text
Backup Artifact
      ↓ encrypted with
Data Encryption Key (DEK)
      ↓ wrapped by
Key Encryption Key (KEK)
      ↓ protected by
Certificate Store / Escrow Package
```

Each backup or backup set should have a unique DEK. The DEK is stored only in wrapped form.

## 7. Key Escrow

The platform must survive loss of the dashboard server.

Escrow package should include:

- Wrapped KEK.
- Certificate thumbprint.
- Recovery metadata.
- Manifest version.
- Creation date.
- Authorized recovery procedure reference.

Escrow packages must be copied to protected repositories and also stored through organizational safe/offline process.

## 8. Disaster Recovery Decryption Flow

1. Install fresh EBROP server.
2. Import latest system manifest.
3. Restore certificate or KEK from escrow.
4. Validate escrow package integrity.
5. Rebuild key registry.
6. Confirm ability to decrypt test artifact.
7. Proceed with recovery.

Key recovery must be tested periodically.

## 9. Audit Logging

Every sensitive operation must be audited:

- Login.
- Agent approval.
- Certificate revocation.
- Backup job creation.
- Backup execution.
- Restore request.
- Restore confirmation.
- Repository deletion.
- Retention pruning.
- Key export.
- Manifest import.

Audit records should be append-only through normal application interfaces.

## 10. Command Safety

EBROP must never expose arbitrary command execution. Operators select systems, jobs, databases and repositories that are already registered and validated.

All native tool calls are generated from trusted configuration and escaped properly.

## 11. Secret Storage

Credentials must be encrypted at rest. Where possible:

- Prefer Windows Authentication for SQL Server.
- Prefer service accounts over shared passwords.
- Avoid storing domain administrator credentials.
- Rotate credentials periodically.

## 12. Security Monitoring

Alerts should be raised for:

- Failed agent authentication.
- Unknown certificate fingerprint.
- Revoked agent activity.
- Unauthorized restore attempt.
- Key escrow export.
- Multiple failed login attempts.
- Unexpected repository deletion.
- Checksum mismatch.

## 13. Least Privilege

The agent service account should have only required privileges. Backup mode or local administrator access may be required for VSS and ACL-preserving file backup, but this should be documented per server.

## 14. Secure Defaults

Default configuration:

- Encryption enabled.
- Compression enabled.
- Retention enabled.
- Audit enabled.
- Manual restore validation required.
- Production restore requires confirmation.
- Agent certificates require approval.
