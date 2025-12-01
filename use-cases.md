# Use Cases and Scenarios

Real-world scenarios and solutions using the Azure DevOps Backup & Restore utility.

## Table of Contents

1. [Disaster Recovery](#disaster-recovery)
2. [Organization Migration](#organization-migration)
3. [Project Cloning](#project-cloning)
4. [Development Environment Setup](#development-environment-setup)
5. [Compliance and Auditing](#compliance-and-auditing)
6. [Continuous Backup Strategy](#continuous-backup-strategy)
7. [Selective Data Management](#selective-data-management)
8. [Multi-Organization Management](#multi-organization-management)

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

## Organization Migration

### Scenario 3: Migrate to New Azure DevOps Organization

**Context:** Your company is:
- Merging with another company
- Moving to different Azure tenant
- Restructuring DevOps setup

**Requirements:**
- Migrate all data to new organization
- Preserve history and relationships
- Minimize service interruption

**Solution:**

**Phase 1: Preparation**
```bash
# 1. Create target organization in Azure DevOps
# 2. Create target projects (must exist before restore)
# 3. Configure PAT tokens for both organizations
```

**Phase 2: Backup source organization**
```bash
adobackup.exe backup-all \
  --OrganizationUrl "https://dev.azure.com/sourceorg" \
  --Pat "source-pat" \
  --BackupRoot "D:\Migration\Backup" \
  -v
```

**Phase 3: Test restore to target**
```bash
# Dry run to validate
adobackup.exe restore-all \
  --BackupRoot "D:\Migration\Backup" \
  --target-org "https://dev.azure.com/targetorg" \
  --target-pat "target-pat" \
  --dry-run \
  -v
```

**Phase 4: Production migration**
```bash
# Schedule during maintenance window
# Announce downtime to users

# Execute migration
adobackup.exe restore-all \
  --BackupRoot "D:\Migration\Backup" \
  --target-org "https://dev.azure.com/targetorg" \
  --target-pat "target-pat" \
  -v

# Verify all resources migrated successfully
# Update user access and permissions
# Update pipeline service connections
# Update Git remote URLs
```

**Phase 5: Cutover**
```bash
# Update team documentation with new URLs
# Redirect old organization (if possible)
# Monitor for issues
```

**Expected Outcome:**
- Complete organization migrated
- All history preserved
- 4-8 hours total migration time
- User reconfiguration required (Git remotes, etc.)

---

### Scenario 4: Selective Migration (Specific Projects)

**Context:** Migrate only certain projects to new organization.

**Solution:**

```bash
# Backup specific projects only
adobackup.exe backup-all \
  -p "Project1,Project2,Project3" \
  --OrganizationUrl "https://dev.azure.com/sourceorg" \
  --Pat "source-pat" \
  --BackupRoot "D:\SelectiveMigration" \
  -v

# Restore to target organization
adobackup.exe restore-all \
  -p "Project1,Project2,Project3" \
  --target-org "https://dev.azure.com/targetorg" \
  --target-pat "target-pat" \
  --BackupRoot "D:\SelectiveMigration" \
  -v
```

---

## Project Cloning

### Scenario 5: Clone Project for Development/Testing

**Context:** Need a copy of production project for:
- Testing new processes
- Training developers
- Experimentation

**Solution:**

```bash
# Backup production project
adobackup.exe backup-all \
  -p "Production" \
  --BackupRoot "D:\Clone" \
  -v

# Restore to new project
adobackup.exe restore-all \
  -p "Production" \
  --target-project "Development" \
  --BackupRoot "D:\Clone" \
  -v
```

**Cleanup sensitive data (optional):**
```bash
# After restore, sanitize data in Development project
# - Remove production secrets from variables
# - Clear production connection strings
# - Update work items to remove customer data
```

**Expected Outcome:**
- Identical copy of production
- Safe environment for testing
- 30-60 minutes to complete

---

### Scenario 6: Create Training Environment

**Context:** Onboard new team members with realistic environment.

**Solution:**

```bash
# Clone production to training project
adobackup.exe backup-all \
  -p "RealProject" \
  -v

adobackup.exe restore-all \
  -p "RealProject" \
  --target-project "Training" \
  -v

# Customize training project
# - Add sample work items
# - Create practice branches
# - Set up sample pipelines
```

---

## Development Environment Setup

### Scenario 7: Rapid Development Environment Provisioning

**Context:** Need to quickly set up multiple development environments for new team or project.

**Solution:**

**Create template project:**
```bash
# Set up one "perfect" project with all configurations
# Backup as template
adobackup.exe backup-all \
  -p "DevTemplate" \
  --BackupRoot "D:\Templates" \
  -v
```

**Provision new environments:**
```bash
# For each new team/developer
adobackup.exe restore-all \
  -p "DevTemplate" \
  --target-project "Team-A-Dev" \
  -v

adobackup.exe restore-all \
  -p "DevTemplate" \
  --target-project "Team-B-Dev" \
  -v
```

**Expected Outcome:**
- Consistent development environments
- Rapid provisioning (10-20 minutes per environment)
- Standardized configurations

---

### Scenario 8: Restore Specific Resources Only

**Context:** Need to copy only build definitions from one project to another.

**Solution:**

```bash
# Backup source project
adobackup.exe backup-all \
  -p "SourceProject" \
  --BackupRoot "D:\Backups" \
  -v

# Restore only build definitions
adobackup.exe restore-all \
  -p "SourceProject" \
  --target-project "TargetProject" \
  --skip-git \
  --skip-workitems \
  --skip-variables \
  --skip-queries \
  -v
```

---

## Compliance and Auditing

### Scenario 9: Regulatory Compliance Archival

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

### Scenario 10: Audit Trail for Change Management

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

### Scenario 12: Multi-Region Backup Strategy

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

### Scenario 13: Backup Only Critical Assets

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

### Scenario 14: Backup Without Large Git Repositories

**Context:** Git repositories are backed up separately (GitHub, GitLab, etc.). Only need DevOps-specific resources.

**Solution:**

```bash
# Backup everything except Git
adobackup.exe backup-all \
  --skip-git \
  -v

# Storage savings: 50-80% depending on repo sizes
```

---

## Multi-Organization Management

### Scenario 15: Managing Multiple Client Organizations

**Context:** MSP (Managed Service Provider) managing Azure DevOps for multiple clients.

**Requirements:**
- Backup all client organizations
- Segregated storage
- Individual reporting
- Automated scheduling

**Solution:**

**Backup script for multiple orgs:**
```powershell
# clients.json
$clients = @(
    @{ Name = "Client-A"; Org = "https://dev.azure.com/clienta"; Pat = "pat-a" },
    @{ Name = "Client-B"; Org = "https://dev.azure.com/clientb"; Pat = "pat-b" },
    @{ Name = "Client-C"; Org = "https://dev.azure.com/clientc"; Pat = "pat-c" }
)

foreach ($client in $clients) {
    Write-Host "Backing up $($client.Name)..."
    
    $backupRoot = "D:\MSP-Backups\$($client.Name)"
    
    & adobackup.exe backup-all `
        --OrganizationUrl $client.Org `
        --Pat $client.Pat `
        --BackupRoot $backupRoot `
        -i `
        -v
    
    # Generate client report
    $report = @{
        Client = $client.Name
        Date = Get-Date
        Success = $LASTEXITCODE -eq 0
        BackupSize = (Get-ChildItem $backupRoot -Recurse | Measure-Object -Property Length -Sum).Sum / 1GB
    }
    
    $report | ConvertTo-Json | Out-File "D:\MSP-Reports\$($client.Name)-$(Get-Date -Format 'yyyyMMdd').json"
}
```

**Azure Pipeline for MSP:**
```yaml
name: MSP-MultiClient-Backup

schedules:
  - cron: "0 3 * * *"
    displayName: Daily Client Backups

jobs:
- job: ClientA
  steps:
    - task: AzureDevOpsBackupTask@0
      inputs:
        OrganizationUrl: 'https://dev.azure.com/clienta'
        Pat: '$(ClientA.Pat)'
        BackupRoot: '$(Backup.Root)/ClientA'

- job: ClientB
  steps:
    - task: AzureDevOpsBackupTask@0
      inputs:
        OrganizationUrl: 'https://dev.azure.com/clientb'
        Pat: '$(ClientB.Pat)'
        BackupRoot: '$(Backup.Root)/ClientB'
```

---

## Summary Matrix

| Use Case | Frequency | Backup Mode | Resources | Complexity |
|----------|-----------|-------------|-----------|------------|
| Disaster Recovery | Daily | Incremental | All | Medium |
| Organization Migration | One-time | Full | All | High |
| Project Cloning | As needed | Full | All | Low |
| Dev Environment Setup | As needed | Full | Selective | Low |
| Compliance Archival | Monthly | Full | All | Medium |
| Continuous Backup | Every 4h | Incremental | Tiered | High |
| Selective Backup | Daily | Incremental | Critical only | Medium |
| Multi-Org Management | Daily | Incremental | All per org | High |

---

## Additional Resources

- [Getting Started Guide](./getting-started.md)
- [Command Reference](./command-reference.md)
- [Best Practices](./best-practices.md)
- [Pipeline Integration](./pipeline-integration.md)

---

**Last Updated:** January 2025
