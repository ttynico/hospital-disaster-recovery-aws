# Hospital Disaster Recovery Using AWS Backup

## Project Overview

This project demonstrates how I designed and tested a disaster recovery solution for a simulated hospital environment using AWS (Amazon Web Services).

The goal was to protect critical patient records stored on an Amazon EBS (Elastic Block Store) volume and prove that the data could be restored after a ransomware attack, accidental deletion, corruption, or storage failure.

## Business Scenario

A hospital stores patient records on cloud-based infrastructure.

These records may include:

- Patient charts
- Medication history
- Laboratory results
- MRI and X-ray records
- Surgery schedules
- Billing information

A ransomware attack encrypts the hospital's primary storage volume, preventing doctors and nurses from accessing important patient information.

As the Cloud Engineer, my responsibility was to create a secure backup and recovery system that would allow the hospital to restore patient data and continue operations.

## Business Problem

Without a reliable backup strategy, the hospital could experience:

- Permanent loss of patient records
- Delayed medical treatment
- Extended system downtime
- Operational disruption
- Compliance risks
- Financial loss
- Damage to patient trust

A backup is only valuable if it can be successfully restored. This project therefore includes both backup creation and a documented restore procedure.

## Solution

I created an AWS Backup disaster recovery workflow that:

- Stores backups inside an encrypted, tamper-proof Backup Vault (Vault Lock, compliance mode)
- Runs automated daily backups
- Retains recovery points for 90 days
- Uses tags to identify resources requiring protection
- Supports immediate on-demand backups
- Restores an Amazon EBS volume from a recovery point (via a documented, manual restore procedure — see below)

## AWS Architecture
```
Doctors and Nurses
        |
        v
Amazon EC2 Application Server
        |
        v
Amazon EBS Patient Records Volume
        |
        v
AWS Backup Plan
        |
        v
Encrypted Backup Vault
        |
        v
Recovery Point / EBS Snapshot
        |
        v
AWS Restore Job
        |
        v
Recovered Amazon EBS Volume
```

## What This Deploys

- **Backup Vault** (`AWS::Backup::BackupVault`) — A secure, isolated storage container for backups, with **Vault Lock** enabled in compliance mode (immutable after a 3-day cooling-off period).
- **IAM Role** (`AWS::IAM::Role`) — Grants AWS Backup the permissions it needs to back up and restore tagged resources.
- **Backup Plan** (`AWS::Backup::BackupPlan`) — Runs daily backups at 5:00 AM UTC, retaining each backup for 90 days.
- **Backup Selection** (`AWS::Backup::BackupSelection`) — Automatically includes any resource tagged `Backup: true` in the backup plan (EC2, RDS, EFS, DynamoDB, etc.).

## Vault Lock (Compliance Mode)

This template enables AWS Backup Vault Lock with:
- `MinRetentionDays: 90`
- `MaxRetentionDays: 90`
- `ChangeableForDays: 3`

**This is a one-way door.** Once the 3-day cooling-off period passes, the retention policy becomes permanent and cannot be loosened or removed by anyone — including the AWS account root user or AWS Support. This is intentional: it protects backups against ransomware, insider threats, and accidental or malicious deletion, which is especially critical for healthcare data.

If you want to test the mechanics first without a permanent lock, deploy to a separate sandbox vault before using this template against production.

## Usage

1. Tag any AWS resource you want backed up:
```
   Key: Backup
   Value: true
```
2. Deploy the stack:
```bash
   aws cloudformation deploy \
     --template-file cloudformation/backup-template.yaml \
     --stack-name hospital-backup-stack \
     --capabilities CAPABILITY_NAMED_IAM
```

3. Backups will run automatically every day at 5:00 AM UTC for any tagged resource, and expire after 90 days.

## Restore Procedure (Manual)

Restoring is not automated by this template — it's performed on demand when needed, either through the AWS Backup console or CLI:

```bash
aws backup start-restore-job \
  --recovery-point-arn <RECOVERY_POINT_ARN> \
  --iam-role-arn <BACKUP_ROLE_ARN> \
  --metadata '{"file-system-id":"<TARGET_VOLUME_ID>"}'
```

Restore testing should be performed periodically (not just during an actual emergency) to confirm recovery points are valid and the process works as expected.

## Testing & Validation

This backup and restore workflow was tested end-to-end using a sandbox version of this template (no permanent Vault Lock, shorter retention) to safely validate the mechanics before relying on the production configuration:

1. Deployed the sandbox stack (vault, IAM role, backup plan, tag-based selection).
2. Created a test EBS volume tagged `Backup: true`.
3. Triggered an on-demand backup job — completed successfully, producing a recovery point (snapshot).
4. Triggered a restore job from that recovery point — completed successfully, producing a new, fully restored EBS volume.
5. Cleaned up all test resources (volumes, recovery point, and sandbox stack) after validation.

This confirms the complete backup-to-restore cycle works as designed, not just in theory.

## Requirements

- AWS CLI configured with appropriate permissions
- IAM permissions to create Backup Vaults, Backup Plans, Backup Selections, and IAM Roles

## Notes

- This template is designed for a healthcare/hospital context, where regulatory compliance (e.g., HIPAA-adjacent data protection expectations) requires tamper-proof backup retention.
- Review the [AWS Backup Vault Lock documentation](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html) before deploying, since compliance-mode locks are irreversible.
