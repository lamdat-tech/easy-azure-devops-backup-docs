# Use Cases and Scenarios

Real-world scenarios and solutions using the Azure DevOps Backup & Restore utility.

**Note:** This utility supports **backup and restore operations only**. It does not provide migration, synchronization, or multi-organization management features.

## Table of Contents

1. [Disaster Recovery](#disaster-recovery)
2. [Compliance and Auditing](#compliance-and-auditing)
3. [Continuous Backup Strategy](#continuous-backup-strategy)
4. [Selective Data Management](#selective-data-management)

---

## Disaster Recovery

### Scenario 1: Complete Organization Recovery

**Context:** Your Azure DevOps organization experiences catastrophic data loss or corruption.

**Requirements:**
- Restore all projects and resources
- Minimize downtime (RTO: 4 hours)
- Minimize data loss (RPO: 6 hours)

**Solution:**

**Step 1: Maintain regular backups**
```yaml
# Daily incremental backup at 2 AM
name: DailyBackup
schedules:
  - cron: "0 2 * * *"
steps:
  - task: AzureDevOpsBackupTask@0
    inputs:
      BackupMode: 'Incremental'
      Verbose: true
```

**Step 2: Weekly full backup**
```yaml
# Weekly full backup on Sunday at 1 AM
name: WeeklyFullBackup
schedules:
  - cron: "0 1 * * 0"
steps:
  - task: AzureDevOpsBackupTask@0
    inputs:
      BackupMode: 'Full'
```

**Step 3: When disaster strikes - Restore**
```bash
# 1. Validate latest backup
adobackup.exe restore-all --dry-run -v

# 2. Restore all resources
adobackup.exe restore-all \
  --OrganizationUrl "https://dev.azure.com/yourorg" \
  --Pat "your-pat" \
  --BackupRoot "D:\ADOBackups" \
  -v

# 3. Verify restoration
# Check projects, repos, work items manually

# 4. Communicate to team
# Notify users of recovery completion
```

**Expected Outcome:**
- All projects restored
- Git history preserved
- Work items with full history
- Build definitions restored
- Downtime: 2-4 hours depending on size

---

### Scenario 2: Single Project Recovery

**Context:** One project was accidentally deleted or corrupted, but others are fine.

**Solution:**

```bash
# Restore only the affected project
adobackup.exe restore-all \
  -p "DeletedProject" \
  --BackupRoot "D:\ADOBackups" \
  --dry-run \
  -v

# After dry-run validation, restore for real
adobackup.exe restore-all \
  -p "DeletedProject" \
  -v
```

**Expected Outcome:**
- Single project restored in 15-30 minutes
- Other projects unaffected
- Minimal downtime



---

## Compliance and Auditing

### Scenario 3: Regulatory Compliance Archival

**Context:** Financial services company needs to maintain 7-year archive of all project data for regulatory compliance.

**Requirements:**
- Monthly full backups
- Immutable storage
- Audit trail
- Easy retrieval

**Solution:**

**Monthly archival backup:**
```yaml
# Monthly full backup on 1st of month
name: MonthlyArchival
schedules:
  - cron: "0 1 1 * *"
steps:
  - task: AzureDevOpsBackupTask@0
    inputs:
      BackupMode: 'Full'
      BackupRoot: '$(Backup.Root)\Archive\$(Date:yyyy-MM)'
      Verbose: true
  
  # Upload to immutable Azure Blob Storage
  - task: AzureCLI@2
    inputs:
      azureSubscription: 'Compliance-Archive'
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        az storage blob upload-batch \
          --account-name compliancearchive \
          --destination archives \
          --source $(Backup.Root)/Archive/$(Date:yyyy-MM) \
          --metadata "backup-date=$(Date:yyyy-MM)" "retention=7years"
```

**Generate compliance report:**
```powershell
# Monthly compliance report
$report = @{
    Month = Get-Date -Format "yyyy-MM"
    BackupCompleted = $true
    BackupSize = (Get-ChildItem "D:\Archives\2025-01" -Recurse | Measure-Object -Property Length -Sum).Sum / 1GB
    Projects = @("Project1", "Project2")
    UploadedToArchive = $true
}

$report | ConvertTo-Json | Out-File "compliance-report-$(Get-Date -Format 'yyyy-MM').json"
```

**Expected Outcome:**
- Monthly archives maintained
- Compliance requirements met
- Easy retrieval for audits

---

### Scenario 4: Audit Trail for Change Management

**Context:** Track all changes to production pipelines and configurations.

**Solution:**

**Daily backup with comprehensive logging:**
```bash
# Backup with detailed logs
adobackup.exe backup-all \
  -p "Production" \
  --BackupRoot "D:\AuditBackups\$(Date:yyyyMMdd)" \
  -v 2>&1 | Tee-Object "D:\AuditLogs\backup-$(Date:yyyyMMdd).log"
```

**Compare changes between backups:**
```powershell
# Script to detect changes
$today = Get-ChildItem "D:\AuditBackups\20250128\Production\builds"
$yesterday = Get-ChildItem "D:\AuditBackups\20250127\Production\builds"

Compare-Object $today $yesterday -Property Name, LastWriteTime | 
    Where-Object { $_.SideIndicator -eq '=>' } |
    ForEach-Object { 
        Write-Host "Build definition changed: $($_.Name)"
    }
```

---

## Continuous Backup Strategy

### Scenario 11: 24/7 Continuous Protection

**Context:** Critical organization requires continuous backup with minimal RPO.

**Requirements:**
- RPO: 4 hours maximum
- Automated backups every 4 hours
- Automatic validation
- Alerting on failures

**Solution:**

**Multi-frequency backup schedule:**
```yaml
name: ContinuousBackup

schedules:
  # Every 4 hours for work items (most volatile)
  - cron: "0 */4 * * *"
    displayName: Work Items Backup
    
  # Every 6 hours for builds
  - cron: "0 */6 * * *"
    displayName: Builds Backup
    
  # Daily for Git (less volatile)
  - cron: "0 2 * * *"
    displayName: Git Backup

jobs:
- job: BackupWorkItems
  steps:
    - task: AzureDevOpsBackupTask@0
      inputs:
        BackupMode: 'Incremental'
        SkipGit: true
        SkipBuilds: true
        SkipVariables: true
        SkipQueries: true
        Verbose: true

- job: ValidateBackup
  dependsOn: BackupWorkItems
  steps:
    - powershell: |
        # Validate backup completed successfully
        $metadataPath = "$(Backup.Root)/metadata"
        if (!(Test-Path $metadataPath)) {
          Write-Error "Backup validation failed"
          exit 1
        }
        # Check backup is recent (within last 5 hours)
        $metadata = Get-Content "$metadataPath/workitems/metadata.json" | ConvertFrom-Json
        $lastBackup = [DateTime]::Parse($metadata.LastBackupDate)
        $hours = ((Get-Date) - $lastBackup).TotalHours
        if ($hours -gt 5) {
          Write-Error "Backup is stale: $hours hours old"
          exit 1
        }
        Write-Host "Backup validation passed"
      displayName: 'Validate Backup'

- job: AlertOnFailure
  condition: failed()
  steps:
    - task: SendEmail@1
      inputs:
        To: 'ops-team@company.com'
        Subject: 'ALERT: Backup Failed'
        Body: 'Continuous backup failed. Review logs immediately.'
```

**Expected Outcome:**
- Maximum 4-hour data loss window
- Automated validation
- Immediate alerting on issues

---

### Scenario 5: Multi-Region Backup Strategy

**Context:** Enterprise with global operations requires geographically distributed backups.

**Solution:**

```bash
# Primary backup (Region 1 - US East)
adobackup.exe backup-all \
  --BackupRoot "D:\Backups\Primary" \
  -v

# Replicate to Region 2 (US West)
robocopy "D:\Backups\Primary" "\\region2-server\Backups\Secondary" /MIR /Z

# Replicate to Region 3 (Europe) - Azure Blob
az storage blob upload-batch \
  --account-name backupseu \
  --destination adobackups \
  --source "D:\Backups\Primary"

# Replicate to Region 4 (Asia) - Azure Blob
az storage blob upload-batch \
  --account-name backupsasia \
  --destination adobackups \
  --source "D:\Backups\Primary"
```

---

## Selective Data Management

### Scenario 6: Backup Only Critical Assets

**Context:** Large organization wants to backup only business-critical projects to save storage and time.

**Solution:**

**Define tiers:**
- **Tier 1 (Critical):** Customer-facing, production projects - backup every 4 hours
- **Tier 2 (Important):** Development projects - backup daily
- **Tier 3 (Archive):** Old projects - backup weekly

**Tier 1 backup:**
```bash
adobackup.exe backup-all \
  -p "CustomerPortal,MainApp,PaymentService" \
  -i \
  --BackupRoot "D:\Backups\Tier1" \
  -v
```

**Tier 2 backup:**
```bash
adobackup.exe backup-all \
  -p "Development,Staging" \
  -i \
  --BackupRoot "D:\Backups\Tier2" \
  -v
```

**Tier 3 backup:**
```bash
adobackup.exe backup-all \
  -p "Archive2023,Archive2024" \
  --BackupRoot "D:\Backups\Tier3" \
  -v
```

---

### Scenario 7: Backup Without Large Git Repositories

**Context:** Git repositories are backed up separately (GitHub, GitLab, etc.). Only need DevOps-specific resources.

**Solution:**

```bash
# Backup everything except Git
adobackup.exe backup-all \
  --include-builds \
  --include-workitems \
  --include-variables \
  --include-queries \
  -v

# Storage savings: 50-80% depending on repo sizes
```



---

## Summary Matrix

| Use Case | Frequency | Backup Mode | Resources | Complexity |
|----------|-----------|-------------|-----------|------------|
| Disaster Recovery | Daily | Incremental | All | Medium |
| Compliance Archival | Monthly | Full | All | Medium |
| Continuous Backup | Every 4h | Incremental | Tiered | High |
| Selective Backup | Daily | Incremental | Critical only | Medium |

---

## Additional Resources

- [Getting Started Guide](./getting-started.md)
- [Command Reference](./command-reference.md)
- [Best Practices](./best-practices.md)
- [Pipeline Integration](./pipeline-integration.md)

---

**Last Updated:** January 2025
