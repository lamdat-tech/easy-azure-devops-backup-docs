# Use Cases and Scenarios

Real-world scenarios and solutions using the Azure DevOps Backup & Restore utility.

**Note:** This utility supports **backup and restore operations only**. It does not provide migration, synchronization, or multi-organization management features.

## Table of Contents

1. [Disaster Recovery](#disaster-recovery)
2. [Organization Migration](#organization-migration)
3. [Compliance and Auditing](#compliance-and-auditing)
4. [Continuous Backup Strategy](#continuous-backup-strategy)
5. [Selective Data Management](#selective-data-management)

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

Migrate projects and resources to a different Azure DevOps organization.

### Scenario: Cross-Organization Work Items Migration

**Context:** Migrating a project to a new Azure DevOps organization with full work items history and all parent-child/link relationships preserved.

**Requirements:**
- All work items migrated with history
- Parent-child and link relationships preserved across organizations
- Two-phase restore ensures correct ID remapping

**Solution:**

**Step 1: Backup work items from source organization**
```bash
adobackup.exe backup-all ^
  --OrganizationUrl "https://dev.azure.com/sourceorg" ^
  --Pat "source-pat-token" ^
  --BackupRoot "C:\ADOBackups" ^
  --include-workitems ^
  -p "SourceProject" ^
  -v
```

**Step 2: Dry run — preview the cross-org restore**
```bash
adobackup.exe workitems-restore ^
  --OrganizationUrl "https://dev.azure.com/sourceorg" ^
  --Pat "source-pat-token" ^
  --BackupRoot "C:\ADOBackups" ^
  -p "SourceProject" ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "target-pat-token" ^
  --target-project "TargetProject" ^
  --dry-run ^
  -v
```

**Step 3: Execute cross-org work items restore**
```bash
adobackup.exe workitems-restore ^
  --OrganizationUrl "https://dev.azure.com/sourceorg" ^
  --Pat "source-pat-token" ^
  --BackupRoot "C:\ADOBackups" ^
  -p "SourceProject" ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "target-pat-token" ^
  --target-project "TargetProject" ^
  -v
```

**How the two-phase restore works:**
1. **Phase 1:** Creates all work items in the target organization with new IDs assigned by Azure DevOps and builds a source-to-target ID mapping file (`.metadata/id-mapping.json`).
2. **Phase 2:** Relinks all relations, parent-child links, and references using the new target IDs from the mapping file.

**Prerequisites:**
- Target project must exist in the target organization before running the restore
- Specify exactly one source project with `-p` when using `--target-org` or `--target-project`
- Target PAT must have Work Items (Read & Write) permission

**Expected Outcome:**
- All work items recreated in the target organization
- Work item IDs change (Azure DevOps assigns new IDs in the target)
- All parent-child relationships and links preserved via ID mapping
- Idempotent — safe to re-run; already-migrated items are skipped

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

### Scenario 8: Handling Partial Failures with Warning Behavior

**Context:** Large organization with thousands of work items needs flexible error handling during backups.

**Requirements:**
- Critical backups must fail if any resource fails
- Development backups can tolerate some failures
- Archive backups should complete even with issues

**Solution:**

**Critical Production Backup - Fail on any issue:**
```yaml
# Production backup with strict error handling
name: ProductionBackup
schedules:
  - cron: "0 2 * * *"
steps:
  - task: AzureDevOpsBackupTask@0
    inputs:
      Projects: 'ProductionProject'
      BackupMode: 'Incremental'
      WarningBehavior: 'Error'  # Fail task on any partial failures
      Verbose: true
```

```bash
# CLI equivalent
adobackup.exe backup-all \
  -p "ProductionProject" \
  -i \
  --warnings-as-errors \
  -v
```

**Development Backup - Show warnings but continue:**
```yaml
# Development backup with default warning behavior
name: DevBackup
steps:
  - task: AzureDevOpsBackupTask@0
    inputs:
      Projects: 'DevProject'
      BackupMode: 'Incremental'
      WarningBehavior: 'Warning'  # Default: show issues but succeed
      Verbose: true
```

```bash
# CLI equivalent (default behavior)
adobackup.exe backup-all \
  -p "DevProject" \
  -i \
  -v
```

**Archive Backup - Ignore failures:**
```yaml
# Archive backup that must complete regardless
name: ArchiveBackup
steps:
  - task: AzureDevOpsBackupTask@0
    inputs:
      Projects: 'OldProjects'
      BackupMode: 'Full'
      WarningBehavior: 'Ignore'  # Complete even with failures
      Verbose: true
```

```bash
# CLI equivalent
adobackup.exe backup-all \
  -p "OldProjects" \
  --ignore-warnings \
  -v
```

**Expected Outcomes:**

| Scenario | 10 items fail out of 1000 | Task Result |
|----------|---------------------------|-------------|
| Production (`--warnings-as-errors`) | Task fails, pipeline stops | ? Failed |
| Development (default) | Task succeeds with issues | ?? Succeeded with Issues |
| Archive (`--ignore-warnings`) | Task succeeds | ? Succeeded |

**When to use each option:**

- **`WarningBehavior: Error` / `--warnings-as-errors`**
  - Critical production systems
  - Compliance-required backups
  - Zero-tolerance for data loss
  - Gating deployments on successful backups

- **`WarningBehavior: Warning` (Default)**
  - Standard development workflows
  - Balance between visibility and flexibility
  - Want to know about issues without blocking
  - Most common use case

- **`WarningBehavior: Ignore` / `--ignore-warnings`**
  - Archive/historical data where some loss is acceptable
  - Best-effort backups of deprecated projects
  - Large-scale migrations where some failures expected
  - Use with caution - may hide real issues

---

## Summary Matrix

| Use Case | Frequency | Backup Mode | Resources | Complexity | Warning Behavior |
|----------|-----------|-------------|-----------|------------|------------------|
| Disaster Recovery | Daily | Incremental | All | Medium | Warning (default) |
| Compliance Archival | Monthly | Full | All | Medium | Error (strict) |
| Continuous Backup | Every 4h | Incremental | Tiered | High | Warning (default) |
| Selective Backup | Daily | Incremental | Critical only | Medium | Warning (default) |
| Partial Failure Handling | Varies | Varies | Varies | Low | Error/Warning/Ignore |


---

## Test Plans Backup and Restore

### Scenario 1: Backup Test Plans with Last 90 Days of Runs

**Context:** You need to backup test plans for compliance, preserving the last 3 months of test execution history.

**Requirements:**
- Backup all test plans and suites
- Include last 90 days of test runs (default)
- Preserve hierarchical suite structure

**Solution:**

```bash
# Backup test plans with default 90-day test run history
adobackup.exe testplans-backup \
  --OrganizationUrl "https://dev.azure.com/yourorg" \
  --Pat "your-pat" \
  --BackupRoot "D:\ADOBackups" \
  -v

# Backup specific projects only
adobackup.exe testplans-backup \
  -p "Project1,Project2" \
  --test-runs-days 90 \
  -v
```

**Expected Outcome:**
- All test plans and suites backed up hierarchically
- Test runs from last 90 days included
- Test results and metadata preserved
- Backup folder: `{BackupRoot}/test-plans/{project}/{planId}/`

---

### Scenario 2: Full Test History Backup (Compliance/Archival)

**Context:** Regulatory compliance requires complete test execution history for audit purposes.

**Requirements:**
- Backup all test runs regardless of date
- Complete test result history
- Suitable for long-term archival

**Solution:**

```bash
# Backup all test runs (entire history)
adobackup.exe testplans-backup \
  --test-runs-all \
  -v

# Alternative: Specify longer date range (e.g., 2 years)
adobackup.exe testplans-backup \
  --test-runs-days 730 \
  -v
```

**Expected Outcome:**
- All test plans and suites backed up
- **Complete** test run history (all dates)
- Longer execution time (may take hours for extensive history)
- Larger storage requirements

**Recommendation:** Use for monthly/quarterly compliance backups, not daily operations.

---

### Scenario 3: Cross-Project Test Plan Migration

**Context:** You're merging two projects and need to migrate test plans from "OldProject" to "NewProject".

**Requirements:**
- Migrate test plans and suites
- Test case work items already exist in target
- Preserve suite hierarchy

**Solution:**

**Step 1: Backup source project test plans**
```bash
adobackup.exe testplans-backup \
  -p "OldProject" \
  -v
```

**Step 2: Dry run restore to preview**
```bash
adobackup.exe testplans-restore \
  -p "OldProject" \
  --target-project "NewProject" \
  --dry-run \
  -v
```

**Step 3: Restore to target project**
```bash
adobackup.exe testplans-restore \
  -p "OldProject" \
  --target-project "NewProject" \
  -v
```

**Expected Outcome:**
- Test plans created in NewProject with new IDs
- Suite hierarchy preserved
- Test cases associated (if work items exist)
- ID mapping saved to `.metadata/test-plan-id-mapping.json`
- Missing test cases skipped with warnings

**Important:** Ensure test case work items are migrated to NewProject **before** restoring test plans.

---

### Scenario 4: Restore Test Plans Without Test Runs

**Context:** You want to restore test plan structure but not historical test execution data.

**Requirements:**
- Restore test plans and suites only
- Skip test runs (save time and storage)
- Preserve hierarchical structure

**Solution:**

```bash
# Restore plans and suites only (default behavior)
adobackup.exe testplans-restore \
  -p "ProjectA" \
  -v

# Explicitly without runs
adobackup.exe testplans-restore \
  -p "ProjectA" \
  --include-runs false \
  -v
```

**Expected Outcome:**
- Test plans and suites restored
- Test case associations created
- **No test runs restored** (faster execution)
- Suitable for project cloning or structure migration

---

### Scenario 5: Restore Test Plans with Test Runs

**Context:** You need complete test plan restoration including historical execution data.

**Requirements:**
- Restore test plans, suites, and runs
- Preserve execution history
- Maintain test result data

**Solution:**

```bash
# Restore plans, suites, and runs
adobackup.exe testplans-restore \
  -p "ProjectA" \
  --include-runs \
  -v

# Cross-project restore with runs
adobackup.exe testplans-restore \
  -p "SourceProject" \
  --target-project "TargetProject" \
  --include-runs \
  -v
```

**Expected Outcome:**
- Test plans and suites restored
- Test runs restored with original metadata
- Test results linked to new plan/suite IDs
- Execution history preserved

**Note:** Test run IDs are remapped to target project IDs via ID mapping files.

---

### Scenario 6: Handling Suite Type Conversions

**Context:** Restoring test plans to a target project where requirement/query dependencies are missing.

**Challenge:**
- Source has requirement-based suites linked to work item #5678
- Target project doesn't have work item #5678
- Source has query-based suites using query "Regression Bugs"
- Target project doesn't have that query

**Automatic Solution:**

The restore process handles this automatically:

```bash
adobackup.exe testplans-restore \
  -p "SourceProject" \
  --target-project "TargetProject" \
  -v
```

**What Happens:**
1. **Requirement-Based Suite** → Converts to **Static Suite** (logs warning)
2. **Query-Based Suite** → Converts to **Static Suite** (logs warning)
3. **Test cases** are associated as static entries
4. **Hierarchy preserved** exactly as before

**Console Output:**
```
⚠️  WARNING: Suite "Requirements Suite" (requirement-based) → converted to static suite (requirement work item #5678 not found)
⚠️  WARNING: Suite "Bug Query Suite" (query-based) → converted to static suite (query "Regression Bugs" not found)
✅ Test Suite "Requirements Suite": 12/15 test cases restored (3 skipped - work items not found)
```

**Expected Outcome:**
- Test plans fully restored despite missing dependencies
- Suite structure intact (as static suites)
- Warnings logged for audit trail
- No manual intervention required

---

---
|----------|-----------|-------------|-----------|------------|------------------|
| Disaster Recovery | Daily | Incremental | All | Medium | Warning (default) |
| Compliance Archival | Monthly | Full | All | Medium | Error (strict) |
| Continuous Backup | Every 4h | Incremental | Tiered | High | Warning (default) |
| Selective Backup | Daily | Incremental | Critical only | Medium | Warning (default) |
| Partial Failure Handling | Varies | Varies | Varies | Low | Error/Warning/Ignore |

---
