# Hospital Disaster Recovery – AWS Backup Infrastructure

An AWS CloudFormation template implementing a tag-based, compliance-locked backup strategy for hospital infrastructure, using AWS Backup.

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

## Requirements

- AWS CLI configured with appropriate permissions
- IAM permissions to create Backup Vaults, Backup Plans, Backup Selections, and IAM Roles

## Notes

- This template is designed for a healthcare/hospital context, where regulatory compliance (e.g., HIPAA-adjacent data protection expectations) requires tamper-proof backup retention.
- Review the [AWS Backup Vault Lock documentation](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html) before deploying, since compliance-mode locks are irreversible.
