# Best Practices Guide

Recommendations and patterns for effectively using the Azure DevOps Backup & Restore utility.

## Table of Contents

1. [Backup Strategy](#backup-strategy)
2. [Storage Management](#storage-management)
3. [Security](#security)
4. [Performance Optimization](#performance-optimization)
5. [Disaster Recovery](#disaster-recovery)
6. [Compliance and Auditing](#compliance-and-auditing)
7. [Common Patterns](#common-patterns)

## Backup Strategy

### 1. Implement the 3-2-1 Backup Rule

📦 **Keep 3 copies of your data**
- 1 primary (your Azure DevOps organization)
- 2 backups (local backup folder and offsite copy)

💾 **Use 2 different media types**
- Local disk (fast recovery)
- Cloud storage (offsite protection)

🌍 **Keep 1 copy offsite**
- Different geographic location
- Different infrastructure

**Important:** The backup task stores all backups in a single folder location (`BackupRoot`). It does **not** automatically create dated copies or manage retention. You are responsible for:
- Copying the `BackupRoot` folder to backup storage (daily, weekly, etc.)
- Managing backup retention policies
- Maintaining multiple generations of backups

**Example — copy to backup storage after each pipeline run:**
```powershell
# Copy the entire backup folder to backup storage (add as a pipeline step)
robocopy "$(Backup.Root)" "E:\BackupStorage\ADOBackups-$(Get-Date -Format 'yyyyMMdd')" /MIR /Z /R:3

# Or upload to cloud storage
az storage blob upload-batch --destination adobackups --source "$(Backup.Root)"
```

### 2. Use Incremental Backups

**Full Backup Mode** — set `BackupMode: 'Full'` in the task input:
```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    BackupMode: 'Full'
```

**Incremental Backup Mode** — set `BackupMode: 'Incremental'`:
```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    BackupMode: 'Incremental'
```

**Benefits:**
- Faster backup times (only new/changed data)
- Reduced API calls to Azure DevOps
- Lower API rate limit impact
- Enables continuous backup strategy

**Important:** Both modes update files in the same `BackupRoot` folder. The difference is what data is retrieved from Azure DevOps:
- **Full mode** retrieves all resources
- **Incremental mode** retrieves only new/changed resources (based on metadata from last backup)

To maintain historical versions, you must copy the `BackupRoot` folder to your backup storage after each run.

### 3. Backup Frequency Recommendations

| Resource Type | Recommended Frequency | Backup Mode |
|---------------|----------------------|-------------|
| Git Repositories | Daily | Full |
| Build Definitions | Daily | Incremental |
| Build History | Daily | Incremental |
| Work Items | Every 6-12 hours | Incremental |
| Pipeline Variables | Daily | Full |
| Queries | Weekly | Full |

**Example Schedule:**
- **06:00 AM** - Incremental work items → Copy to BackupStorage\Morning\{date}
- **02:00 AM** - Incremental builds → Copy to BackupStorage\Night\{date}
- **03:00 AM** - Full all resources → Copy to BackupStorage\Full\{date}
- **06:00 PM** - Incremental work items → Copy to BackupStorage\Evening\{date}
- **Sunday 01:00 AM** - Full backup → Copy to BackupStorage\Weekly\{date}

### 4. Backup Retention Policy

**Recommended Retention:**
- Daily backups: Keep 7 days
- Weekly backups: Keep 4 weeks
- Monthly backups: Keep 12 months
- Yearly backups: Keep indefinitely

**Since `adobackup` stores everything in a single folder, you must implement retention using your own backup tools:**

**Implementation Script** (`backup-and-archive.ps1`):
```powershell
param(
    [string]$BackupRoot = "D:\ADOBackups",
    [string]$ArchiveRoot = "E:\ADOBackupArchive",
    [int]$DaysToKeepDaily = 7,
    [int]$WeeksToKeepWeekly = 4,
    [int]$MonthsToKeepMonthly = 12
)

# This script runs as a pipeline step AFTER AzureDevOpsBackupTask completes

# Step 1: Copy BackupRoot to archive storage with timestamp
$timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
$dayOfWeek = (Get-Date).DayOfWeek
$dayOfMonth = (Get-Date).Day

# Determine archive location based on schedule
if ($dayOfMonth -eq 1) {
    # First day of month = monthly backup
    $archivePath = Join-Path $ArchiveRoot "Monthly\$(Get-Date -Format 'yyyy-MM')"
} elseif ($dayOfWeek -eq 'Sunday') {
    # Sunday = weekly backup
    $archivePath = Join-Path $ArchiveRoot "Weekly\$(Get-Date -Format 'yyyy-Www')"
} else {
    # Daily backup
    $archivePath = Join-Path $ArchiveRoot "Daily\$(Get-Date -Format 'yyyyMMdd')"
}

Write-Host "Archiving backup to: $archivePath"
robocopy $BackupRoot $archivePath /MIR /Z /R:3

# Step 3: Clean up old archives based on retention policy
$now = Get-Date

# Clean daily backups
Get-ChildItem "$ArchiveRoot\Daily" -Directory | 
    Where-Object { $_.CreationTime -lt $now.AddDays(-$DaysToKeepDaily) } |
    ForEach-Object {
        Write-Host "Removing old daily backup: $($_.Name)"
        Remove-Item $_.FullName -Recurse -Force
    }

# Clean weekly backups
Get-ChildItem "$ArchiveRoot\Weekly" -Directory | 
    Where-Object { $_.CreationTime -lt $now.AddDays(-($WeeksToKeepWeekly * 7)) } |
    ForEach-Object {
        Write-Host "Removing old weekly backup: $($_.Name)"
        Remove-Item $_.FullName -Recurse -Force
    }

# Clean monthly backups
Get-ChildItem "$ArchiveRoot\Monthly" -Directory | 
    Where-Object { $_.CreationTime -lt $now.AddMonths(-$MonthsToKeepMonthly) } |
    ForEach-Object {
        Write-Host "Removing old monthly backup: $($_.Name)"
        Remove-Item $_.FullName -Recurse -Force
    }

Write-Host "Backup and archival completed successfully"
```

## Storage Management

### 1. Disk Space Planning

**Understanding Storage Requirements:**

The backup task uses a **single folder** (`BackupRoot`) for all backups. When you run incremental backups, it updates files in place rather than creating new dated folders.

**Estimate Storage Requirements:**

| Resource Type | Space Multiplier | Notes |
|---------------|------------------|-------|
| Git Repositories | 1.2x repo size | Includes bare clone overhead |
| Build History | ~1-5 MB per build | Depends on logs and artifacts |
| Work Items | ~10-50 KB per item | Includes history and attachments |
| Build Definitions | ~50-100 KB each | JSON configuration |
| Variables | ~10-20 KB per group | Small footprint |
| Queries | ~5-10 KB each | WIQL definitions |

**Example Calculation for BackupRoot:**
- 50 GB Git repositories ? ~60 GB
- 10,000 builds ? ~10-50 GB
- 100,000 work items ? ~1-5 GB
- 500 build definitions ? ~25-50 MB
- **Total BackupRoot: ~75-120 GB**

**Additional Storage for Backup Retention:**
If maintaining 7 daily + 4 weekly + 12 monthly copies:
- Daily copies (7): ~525-840 GB
- Weekly copies (4): ~300-480 GB  
- Monthly copies (12): ~900-1,440 GB
- **Total Archive Storage: ~1.7-2.8 TB**

**Recommendation:** 
- Plan for **2x estimated size** for `BackupRoot` to account for growth
- Plan for **retention storage** separately based on your backup strategy

### 2. Storage Architecture

**Recommended Setup:**
```
D:\ADOBackups\                    ← BackupRoot (working backup folder)
├── ProjectA\
├── ProjectB\
├── metadata\
└── logs\

E:\BackupArchive\                 ← Archive storage (your retention copies)
├── Daily\
│   ├── 20250128\
│   ├── 20250129\
│   └── 20250130\
├── Weekly\
│   ├── 2025-W04\
│   └── 2025-W05\
└── Monthly\
    ├── 2024-12\
    └── 2025-01\
```

**Key Concepts:**
1. **BackupRoot (Working Folder)**
   - Single location where `adobackup` stores/updates backups
   - Always contains the latest backup state
   - Incremental mode updates files in place
   - Size = current backup data only

2. **Archive Storage (Your Responsibility)**
   - Historical copies of the BackupRoot folder
   - Managed by your scripts/tools (robocopy, rsync, cloud sync, etc.)
   - Implements retention policy
   - Size = BackupRoot � number of retained copies

### 3. Storage Location Best Practices

✅ **DO:**
- Use dedicated drives for BackupRoot
- Use SSDs for BackupRoot (faster backup operations)
- Use separate storage for archive/retention
- Implement RAID for redundancy on archive storage
- Monitor disk space continuously
- Use compression for old archives

❌ **DON'T:**
- Store BackupRoot on system drive
- Use network drives for BackupRoot (slow)
- Store archives in same location as BackupRoot
- Ignore disk space warnings

### 4. Backup Storage Strategies

**Strategy 1: File-Level Incremental (Recommended)**
```powershell
# Add as a pipeline step after AzureDevOpsBackupTask
# Copy only changed files to archive (file-level incremental)
robocopy "$(Backup.Root)" "E:\Archive\Daily\$(Get-Date -Format 'yyyyMMdd')" /MIR /Z
```
✅ Fast, efficient storage usage

**Strategy 2: Full Copies**
```powershell
# Add as a pipeline step after AzureDevOpsBackupTask
# Create complete copy of BackupRoot
Copy-Item "$(Backup.Root)" -Destination "E:\Archive\$(Get-Date -Format 'yyyyMMdd')" -Recurse
```
✅ Simple, independent restore points
❌ Higher storage usage

**Strategy 3: Cloud Sync**
```bash
# Add as a pipeline step after AzureDevOpsBackupTask
# Sync to Azure Blob Storage with versioning enabled
az storage blob upload-batch \
    --destination adobackups \
    --source "$(Backup.Root)" \
    --account-name mystorageaccount
```
✅ Offsite protection, automatic versioning
❌ Network dependent, costs

**Strategy 4: Snapshot-Based (ZFS, Btrfs, Storage Spaces)**
```powershell
# Add as a pipeline step after AzureDevOpsBackupTask
# Create filesystem snapshot (instant, space-efficient)
New-VolumeSnapshot -DriveLetter D -SnapshotName "Backup-$(Get-Date -Format 'yyyyMMdd')"
```
✅ Instant snapshots, space-efficient
❌ Requires specific filesystem

### 5. Backup Organization

**Recommended BackupRoot Structure (created by the backup task):**
```
D:\ADOBackups\                    ← Your BackupRoot parameter
├── ProjectA\
│   ├── git\
│   │   └── repo-name.git\
│   ├── builds\
│   │   ├── definitions\
│   │   └── history\
│   ├── workitems\
│   ├── variables\
│   └── queries\
├── ProjectB\
│   └── ...
├── metadata\                     ← Used for incremental backups
│   ├── builds\
│   ├── workitems\
│   └── ...
└── logs\                         ← Backup execution logs
```

**Your Archive Structure (your responsibility):**
```
E:\ADOBackupArchive\
├── Daily\
│   ├── 20250128\                 ← Copy of BackupRoot from Jan 28
│   ├── 20250129\                 ← Copy of BackupRoot from Jan 29
│   └── 20250130\                 ← Copy of BackupRoot from Jan 30
├── Weekly\
│   ├── 2025-W04\                 ← Sunday copy
│   └── 2025-W05\
├── Monthly\
│   ├── 2024-12\                  ← First of month copy
│   └── 2025-01\
└── Restore\
    └── Staging\                  ← Test restore location
```

## Security

### 1. PAT Token Management

**Create Separate PATs:**

**Backup PAT (Read-Only):**
- Code: Read
- Build: Read
- Work Items: Read
- Variable Groups: Read
- Project and Team: Read

**Restore PAT (Read-Write):**
- Code: Read & Write
- Build: Read & Write
- Work Items: Read & Write
- Variable Groups: Read & Write
- Project and Team: Read & Write

**Security Guidelines:**
- Set minimum required scopes
- Set expiration dates (max 90 days)
- Rotate tokens regularly
- Store in secure vault (Azure Key Vault, HashiCorp Vault)
- Never commit to source control
- Use separate service accounts for automation

**Example: Using Azure Key Vault with Pipeline Tasks**

Store secrets in Azure Key Vault and link them to your pipeline variable group via the Azure DevOps Library. Mark them as secret variables, then reference them in the task:
```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'  # Retrieved from linked Key Vault secret
    BackupRoot: '$(Backup.Root)'
```

### 2. Backup Encryption

**For Sensitive Data:**

**Option 1: BitLocker (Windows)**
```powershell
# Enable BitLocker on backup drive
Enable-BitLocker -MountPoint "D:" -EncryptionMethod Aes256 -UsedSpaceOnly
```

**Option 2: VeraCrypt**
- Create encrypted container for backups
- Mount before backup operations
- Unmount after completion

**Option 3: Azure Storage Encryption**
- Upload backups to Azure Blob Storage
- Use customer-managed keys (CMK)
- Enable soft delete for recovery

### 3. Access Control

**File System Permissions:**
```powershell
# Set restrictive permissions on backup directory
$acl = Get-Acl "D:\ADOBackups"
$acl.SetAccessRuleProtection($true, $false)  # Disable inheritance
$acl.Access | ForEach-Object { $acl.RemoveAccessRule($_) }  # Remove all

# Add only necessary permissions
$permission = "DOMAIN\BackupServiceAccount", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow"
$accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule $permission
$acl.AddAccessRule($accessRule)

Set-Acl "D:\ADOBackups" $acl
```

### 4. Audit Logging

**Enable Comprehensive Logging:**

Set `Verbose: true` in the task inputs and publish logs as pipeline artifacts:
```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    Verbose: true

- task: PublishBuildArtifacts@1
  displayName: 'Upload Backup Logs'
  condition: always()
  inputs:
    PathtoPublish: '$(Backup.Root)/logs'
    ArtifactName: 'BackupLogs-$(Build.BuildNumber)'
```

## Performance Optimization

### 1. Parallelism Tuning

Use the `MaxParallelism` task input to control concurrent operations. Default is 4.

```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    MaxParallelism: '8'   # High-performance agent: 8-12
                          # Standard agent: 4-8
                          # Rate-limited: 1-2
```

**Considerations:**
- More parallelism = faster backups BUT higher CPU/memory/network usage
- Risk of rate limiting from Azure DevOps API
- Optimal value depends on:
  - Server resources (CPU cores, RAM)
  - Network bandwidth
  - Azure DevOps tier/limits
  - Number of concurrent users

**Recommendation:**
- Start with default (4)
- Monitor performance and rate limiting
- Increase gradually if no issues
- Reduce if experiencing rate limits or resource constraints

### 2. Selective Backups

**Backup Only What You Need:**

**Example 1: Backup only critical projects**
```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    Projects: 'Production,Customer-Facing'
```

**Example 2: Backup only specific components**
```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    BackupAll: false
    IncludeBuilds: true
    IncludeWorkItems: true
    IncludeVariables: true
    IncludeQueries: true
```

**Example 3: Limit build history**
```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    Days: '30'  # Only backup last 30 days of builds
```

### 3. Network Optimization

**For Large Organizations:**

**Option 1: Run on Azure VM**
- Deploy backup server in same Azure region as DevOps
- Reduces latency and improves throughput
- Can use Azure-to-Azure high-speed network

**Option 2: Schedule During Off-Peak Hours**
```yaml
# Azure Pipeline scheduled trigger
schedules:
  - cron: "0 2 * * *"  # 2 AM local time
    displayName: Daily Backup
    branches:
      include:
        - main
```

**Option 3: Use Express Route**
- For large-scale enterprise scenarios
- Dedicated private connection to Azure

## Disaster Recovery

### 1. Define Recovery Objectives

**RTO (Recovery Time Objective):**
- How quickly can you restore after disaster?
- Target: < 4 hours for critical systems

**RPO (Recovery Point Objective):**
- How much data loss is acceptable?
- Target: < 6 hours (run backups every 6 hours)

### 2. Disaster Recovery Plan

**Document These Procedures:**

1. **Backup Verification**
   - How often to test restores
   - Validation criteria
   - Responsible parties

2. **Restore Procedures**
   - Step-by-step instructions
   - Decision tree (partial vs full restore)
   - Rollback procedures

3. **Communication Plan**
   - Who to notify
   - Escalation paths
   - Status updates

4. **Post-Recovery**
   - Validation checklist
   - Root cause analysis
   - Documentation updates

### 3. Regular DR Testing

**Monthly Test — dry run to validate:**
```yaml
- task: AzureDevOpsRestoreTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    Projects: 'CriticalProject'
    TargetOrganizationUrl: 'https://dev.azure.com/testorg'
    TargetPat: '$(DR.AdoPat.RW)'
    TargetProject: 'DR-Test'
    DryRun: true
    Verbose: true
```

**Quarterly Full Test — actual restore:**
```yaml
- task: AzureDevOpsRestoreTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    TargetOrganizationUrl: 'https://dev.azure.com/drorg'
    TargetPat: '$(DR.AdoPat.RW)'
    DryRun: false
    Verbose: true
```

**Document Results:**
- Restore duration
- Issues encountered
- Success criteria
- Lessons learned

### 4. Backup Validation

**Automated Validation Script** (`validate-backup.ps1`):
```powershell
param([string]$BackupRoot = "D:\ADOBackups")

$errors = @()

# Check metadata exists
$metadataPath = Join-Path $BackupRoot "metadata"
if (!(Test-Path $metadataPath)) {
    $errors += "Metadata directory not found"
}

# Check recent backup date
$metadataFiles = Get-ChildItem $metadataPath -Filter "*.json" -Recurse
$latestBackup = $metadataFiles | 
    Sort-Object LastWriteTime -Descending | 
    Select-Object -First 1

$hoursSinceBackup = ((Get-Date) - $latestBackup.LastWriteTime).TotalHours
if ($hoursSinceBackup -gt 24) {
    $errors += "Latest backup is $hoursSinceBackup hours old (>24 hours)"
}

# Check disk space
$drive = (Get-Item $BackupRoot).PSDrive
$freeSpaceGB = [math]::Round($drive.Free / 1GB, 2)
if ($freeSpaceGB -lt 50) {
    $errors += "Low disk space: $freeSpaceGB GB remaining"
}

# Check backup integrity
$projects = Get-ChildItem $BackupRoot -Directory | Where-Object { $_.Name -notmatch "metadata|logs" }
foreach ($project in $projects) {
    $gitPath = Join-Path $project.FullName "git"
    $buildsPath = Join-Path $project.FullName "builds"
    
    if (!(Test-Path $gitPath)) {
        $errors += "Missing git backup for project: $($project.Name)"
    }
    if (!(Test-Path $buildsPath)) {
        $errors += "Missing builds backup for project: $($project.Name)"
    }
}

# Report results
if ($errors.Count -eq 0) {
    Write-Host "✅ Backup validation PASSED" -ForegroundColor Green
    exit 0
} else {
    Write-Host "❌ Backup validation FAILED" -ForegroundColor Red
    $errors | ForEach-Object { Write-Host "  - $_" -ForegroundColor Red }
    exit 1
}
```

## Compliance and Auditing

### 1. Audit Trail

**What to Log:**
- Backup start/end times
- Resources backed up
- Success/failure status
- Errors and warnings
- User who initiated backup
- Source and destination details

**Centralized Logging Example:**

Publish pipeline logs to an artifact store and forward to your central logging system using a pipeline step:
```powershell
# Forward backup logs to central logging (add as a pipeline step)
Get-Content "$(Backup.Root)\logs\*.log" |
  Tee-Object -FilePath "\\logserver\adobackups\backup-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
```

### 2. Compliance Requirements

**GDPR Considerations:**
- Ensure backups include mechanisms for data deletion
- Document data retention periods
- Implement encryption for PII data
- Maintain audit logs of data access

**SOC 2 Requirements:**
- Regular backup testing and validation
- Documented backup and restore procedures
- Access controls and monitoring
- Incident response procedures

### 3. Reporting

**Weekly Backup Report Script** (`generate-backup-report.ps1`):
```powershell
param([string]$BackupRoot = "D:\ADOBackups")

$report = @{
    Date = Get-Date -Format "yyyy-MM-dd HH:mm"
    BackupsLast7Days = @()
    TotalBackupSize = 0
    FailedBackups = @()
}

# Analyze last 7 days of backups
$logs = Get-ChildItem "$BackupRoot\logs" -Filter "*.log" | 
    Where-Object { $_.LastWriteTime -gt (Get-Date).AddDays(-7) }

foreach ($log in $logs) {
    $content = Get-Content $log.FullName -Raw
    $success = $content -match "Backup completed successfully"
    
    $report.BackupsLast7Days += @{
        Date = $log.LastWriteTime
        Success = $success
        Size = (Get-ChildItem $BackupRoot -Recurse | Measure-Object -Property Length -Sum).Sum / 1GB
    }
    
    if (!$success) {
        $report.FailedBackups += $log.Name
    }
}

$report.TotalBackupSize = [math]::Round((Get-ChildItem $BackupRoot -Recurse | 
    Measure-Object -Property Length -Sum).Sum / 1GB, 2)

# Generate HTML report
$html = @"
<html>
<body>
    <h1>Azure DevOps Backup Report</h1>
    <p><strong>Report Date:</strong> $($report.Date)</p>
    <p><strong>Total Backup Size:</strong> $($report.TotalBackupSize) GB</p>
    <p><strong>Backups Last 7 Days:</strong> $($report.BackupsLast7Days.Count)</p>
    <p><strong>Failed Backups:</strong> $($report.FailedBackups.Count)</p>
    $(if ($report.FailedBackups.Count -gt 0) {
        "<h2>Failed Backups:</h2><ul>" + 
        ($report.FailedBackups | ForEach-Object { "<li>$_</li>" }) + 
        "</ul>"
    })
</body>
</html>
"@

$html | Out-File "$BackupRoot\reports\backup-report-$(Get-Date -Format 'yyyyMMdd').html"
```

## Common Patterns

### Pattern 1: Continuous Backup with Daily Archival (24/7)

```yaml
# Run every 6 hours
schedules:
  - cron: "0 */6 * * *"
    displayName: Continuous Incremental Backup
    branches:
      include:
        - main

steps:
  - task: AzureDevOpsBackupTask@0
    env:
      ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
    inputs:
      Pat: '$(Backup.AdoPat.RO)'
      BackupMode: 'Incremental'
      BackupRoot: '$(Backup.Root)'
      BackupAll: true
      Verbose: true

  - powershell: |
      $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
      $archivePath = "$(Archive.Root)\Incremental\$timestamp"
      robocopy "$(Backup.Root)" $archivePath /MIR /Z /R:3 /W:5
      Write-Host "Archived to: $archivePath"
    displayName: 'Archive Backup'
```

### Pattern 2: Multi-Tier Backup with Separate Archives

Use separate pipeline runs for each tier, each targeting a different `BackupRoot` path and project scope:

```yaml
# Tier 1: Critical projects (every 4 hours)
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: 'D:\ADOBackups\Tier1'
    Projects: 'Production,CustomerFacing'
    BackupMode: 'Incremental'

- powershell: robocopy "D:\ADOBackups\Tier1" "E:\Archive\Tier1\$(Get-Date -Format 'yyyyMMdd-HHmm')" /MIR
  displayName: 'Archive Tier1'
```

### Pattern 3: Hot/Warm/Cold Storage Strategy

**Understanding the Workflow:**
1. The backup task updates the **hot storage** (BackupRoot)
2. Your pipeline scripts copy BackupRoot to **warm storage** (recent history)
3. Old warm copies are moved to **cold storage** (long-term archive)

**Implementation (add as pipeline steps after AzureDevOpsBackupTask):**
```powershell
$hotStorage = "$(Backup.Root)"
$warmStorage = "E:\ADOBackups\Warm"
$coldStorage = "\\azure-blob\ADOBackups\Cold"

# Step 1: Copy hot storage to warm storage with today's date
$today = Get-Date -Format "yyyyMMdd"
robocopy $hotStorage "$warmStorage\$today" /MIR /Z

# Step 3: Move old warm backups (>30 days) to cold storage
Get-ChildItem $warmStorage -Directory | 
    Where-Object { $_.CreationTime -lt (Get-Date).AddDays(-30) } |
    ForEach-Object {
        Write-Host "Moving to cold storage: $($_.Name)"
        Move-Item $_.FullName -Destination $coldStorage
    }

# Step 4: Optionally compress cold storage
Get-ChildItem $coldStorage -Directory | 
    Where-Object { $_.Name -notlike "*.zip" } |
    ForEach-Object {
        $zipPath = "$coldStorage\$($_.Name).zip"
        Compress-Archive -Path $_.FullName -DestinationPath $zipPath
        Remove-Item $_.FullName -Recurse -Force
    }
```

### Pattern 4: Cross-Region Replication

**Add replication steps after AzureDevOpsBackupTask in your pipeline:**
```powershell
# Replicate to secondary region (file server)
robocopy "$(Backup.Root)" "\\region2-server\ADOBackups\$(Get-Date -Format 'yyyyMMdd')" /MIR /Z /R:3

# Replicate to Azure Blob Storage (Europe)
az storage blob upload-batch `
    --destination adobackups `
    --source "$(Backup.Root)" `
    --account-name backupseu
```

### Pattern 5: Smart Archival — Weekly Full + Daily Incremental

Use a PowerShell script as a pipeline step to archive after each run:

```powershell
# archive-backup.ps1 — run as a pipeline step after AzureDevOpsBackupTask
param(
    [string]$BackupRoot = "$(Backup.Root)",
    [string]$ArchiveRoot = "E:\Archive"
)

$dayOfMonth = (Get-Date).Day
$dayOfWeek  = (Get-Date).DayOfWeek

if ($dayOfMonth -eq 1) {
    $archivePath = "$ArchiveRoot\Monthly\$(Get-Date -Format 'yyyy-MM')"
} elseif ($dayOfWeek -eq 'Sunday') {
    $archivePath = "$ArchiveRoot\Weekly\$(Get-Date -Format 'yyyy-Www')"
} else {
    $archivePath = "$ArchiveRoot\Daily\$(Get-Date -Format 'yyyyMMdd')"
}

Write-Host "Archiving to: $archivePath"
robocopy $BackupRoot $archivePath /MIR /Z /R:3

# Retention: daily=7d, weekly=4w, monthly=12m
Get-ChildItem "$ArchiveRoot\Daily"   -Directory | Where-Object { $_.CreationTime -lt (Get-Date).AddDays(-7)     } | Remove-Item -Recurse -Force
Get-ChildItem "$ArchiveRoot\Weekly"  -Directory | Where-Object { $_.CreationTime -lt (Get-Date).AddDays(-28)    } | Remove-Item -Recurse -Force
Get-ChildItem "$ArchiveRoot\Monthly" -Directory | Where-Object { $_.CreationTime -lt (Get-Date).AddMonths(-12)  } | Remove-Item -Recurse -Force

Write-Host "Archival completed successfully"
```

### Pattern 6: Azure Pipeline with Multiple Archive Destinations

**Complete Pipeline Example:**
```yaml
name: SmartBackupPipeline_$(Date:yyyyMMdd)$(Rev:.r)

schedules:
  - cron: "0 2 * * *"
    displayName: Daily Backup and Archive
    branches:
      include:
        - main

pool:
  name: 'Default'  # Self-hosted agent with storage access

variables:
  - group: 'ADO Backup Restore'
  - name: 'BackupRoot'
    value: 'D:\ADOBackups'
  - name: 'ArchiveRoot'
    value: 'E:\ADOBackupArchive'

jobs:
- job: BackupAndArchive
  displayName: 'Backup and Archive'
  steps:
    # Step 1: Determine backup mode (full on Sunday, incremental otherwise)
    - powershell: |
        $dayOfWeek = (Get-Date).DayOfWeek
        if ($dayOfWeek -eq 'Sunday') {
          Write-Host "##vso[task.setvariable variable=BackupMode]Full"
          Write-Host "##vso[task.setvariable variable=ArchiveType]Weekly"
        } else {
          Write-Host "##vso[task.setvariable variable=BackupMode]Incremental"
          Write-Host "##vso[task.setvariable variable=ArchiveType]Daily"
        }
      displayName: 'Determine Backup Mode'

    # Step 2: Run adobackup (updates BackupRoot)
    - task: AzureDevOpsBackupTask@0
      displayName: 'Run $(BackupMode) Backup'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
      inputs:
        Pat: '$(Backup.AdoPat.RO)'
        BackupRoot: '$(BackupRoot)'
        Verbose: true
        BackupMode: '$(BackupMode)'

    # Step 3: Archive to local storage
    - powershell: |
        $timestamp = Get-Date -Format "yyyyMMdd"
        $archivePath = "$(ArchiveRoot)\$(ArchiveType)\$timestamp"
        
        Write-Host "Archiving backup to: $archivePath"
        New-Item -ItemType Directory -Path $archivePath -Force | Out-Null
        robocopy "$(BackupRoot)" $archivePath /MIR /Z /R:3 /W:5
        
        if ($LASTEXITCODE -le 7) { 
          Write-Host "Archive completed successfully"
          exit 0
        } else {
          Write-Error "Archive failed with exit code $LASTEXITCODE"
          exit 1
        }
      displayName: 'Archive to Local Storage'

    # Step 4: Copy to network storage
    - powershell: |
        $timestamp = Get-Date -Format "yyyyMMdd"
        $networkPath = "\\fileserver\ADOBackups\$(ArchiveType)\$timestamp"
        
        Write-Host "Copying to network storage: $networkPath"
        robocopy "$(BackupRoot)" $networkPath /MIR /Z /R:3
      displayName: 'Copy to Network Storage'
      continueOnError: true

    # Step 5: Upload to Azure Blob Storage
    - task: AzureCLI@2
      displayName: 'Upload to Azure Storage'
      inputs:
        azureSubscription: 'BackupStorage'
        scriptType: 'pscore'
        scriptLocation: 'inlineScript'
        inlineScript: |
          $timestamp = Get-Date -Format "yyyyMMdd"
          az storage blob upload-batch `
            --account-name adobackups `
            --destination "backups/$(ArchiveType)/$timestamp" `
            --source "$(BackupRoot)" `
            --overwrite `
            --metadata "backup-date=$timestamp" "type=$(ArchiveType)"

    # Step 6: Clean up old archives
    - powershell: |
        # Clean daily (keep 7 days)
        Get-ChildItem "$(ArchiveRoot)\Daily" -Directory | 
          Where-Object { $_.CreationTime -lt (Get-Date).AddDays(-7) } |
          ForEach-Object { 
            Write-Host "Removing old daily backup: $($_.Name)"
            Remove-Item $_.FullName -Recurse -Force 
          }
        
        # Clean weekly (keep 4 weeks)
        Get-ChildItem "$(ArchiveRoot)\Weekly" -Directory | 
          Where-Object { $_.CreationTime -lt (Get-Date).AddDays(-28) } |
          ForEach-Object { 
            Write-Host "Removing old weekly backup: $($_.Name)"
            Remove-Item $_.FullName -Recurse -Force 
          }
      displayName: 'Cleanup Old Archives'

    # Step 7: Validate backup integrity
    - powershell: |
        $metadataPath = "$(BackupRoot)\metadata"
        if (!(Test-Path $metadataPath)) {
          Write-Error "Metadata directory not found!"
          exit 1
        }
        Write-Host "  Backup validation passed"
      displayName: 'Validate Backup'

    # Step 8: Upload logs
    - task: PublishBuildArtifacts@1
      condition: always()
      inputs:
        PathtoPublish: '$(BackupRoot)/logs'
        ArtifactName: 'BackupLogs-$(Build.BuildNumber)'
```

## Summary Checklist

✅ **Backup Strategy**
- [ ] Implement 3-2-1 backup rule (primary + 2 copies)
- [ ] Use incremental backups for frequent operations
- [ ] Define backup frequency per resource type
- [ ] Implement YOUR OWN retention policy (the backup task doesn't manage this)
- [ ] Copy BackupRoot to archive storage after each backup run

💾 **Storage**
- [ ] Plan adequate disk space for BackupRoot (2x estimated size)
- [ ] Plan separate storage for archive/retention copies
- [ ] Use dedicated backup drives for BackupRoot
- [ ] Organize archives logically (daily/weekly/monthly)
- [ ] Monitor disk space continuously
- [ ] Implement automated archival scripts

📁 **Understanding the Storage Model**
- [ ] The backup task uses a SINGLE BackupRoot folder
- [ ] Incremental mode updates files IN PLACE (not separate dated folders)
- [ ] YOU are responsible for copying BackupRoot to create historical versions
- [ ] Implement your own backup tool for archival (robocopy, rsync, cloud sync, etc.)
- [ ] Test your archival and retention scripts regularly

🔒 **Security**
- [ ] Use separate read-only and read-write PATs
- [ ] Store credentials in secure vault
- [ ] Implement backup encryption for archives
- [ ] Set restrictive file permissions on BackupRoot and archives
- [ ] Enable comprehensive audit logging

⚡ **Performance**
- [ ] Tune parallelism based on resources
- [ ] Use selective backups where appropriate
- [ ] Schedule during off-peak hours
- [ ] Monitor for rate limiting

🔄 **Disaster Recovery**
- [ ] Define RTO/RPO objectives
- [ ] Document restore procedures FROM ARCHIVES
- [ ] Test restores monthly using archived copies
- [ ] Validate backups automatically
- [ ] Maintain communication plan

📊 **Compliance**
- [ ] Maintain audit trail
- [ ] Document compliance requirements
- [ ] Generate regular reports
- [ ] Review and update procedures quarterly

---

**Important Reminder:**
The backup task manages retrieving data from Azure DevOps and storing it in your `BackupRoot` folder. **You** are responsible for:
- Copying `BackupRoot` to backup storage (daily, weekly, etc.)
- Managing retention policies
- Ensuring multiple generations of backups exist
- Storing copies offsite
- Implementing the 3-2-1 backup rule

**See Also:**
- [Pipeline Integration](./pipeline-integration.md)
- [Troubleshooting Guide](./troubleshooting.md)
