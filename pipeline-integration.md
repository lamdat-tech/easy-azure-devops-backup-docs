# Getting Started with Azure DevOps Backup & Restore Pipeline Tasks

**This is the recommended way to use Azure DevOps Backup & Restore.** Pipeline tasks provide automated, scheduled backups integrated directly into your Azure DevOps environment.

> **💡 Start Here:** This guide will walk you through setting up automated backups using Azure Pipeline tasks.
>
> **Alternative:** If you need manual operations or CLI access, see [Getting Started with CLI](./getting-started.md).

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Quick Start Setup](#quick-start-setup)
4. [Backup Pipeline Examples](#backup-pipeline-examples)
5. [Restore Pipeline Examples](#restore-pipeline-examples)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)

## Overview

### Why Use Pipeline Tasks?

Using Azure Pipeline tasks for backup operations provides:
- **Automated backups** on a schedule (daily, weekly, etc.)
- **No infrastructure needed** - runs on Azure-hosted or self-hosted agents
- **Built-in logging** with pipeline logs and artifacts
- **Version control** for pipeline configuration
- **Notifications** on backup failures via pipeline alerts
- **Security** with centralized secret management
- **Continuous backup** strategy with incremental mode
- **Audit trail** with complete pipeline history

### Available Tasks

- **AzureDevOpsBackupTask** - Backup Azure DevOps resources
- **AzureDevOpsRestoreTask** - Restore Azure DevOps resources

## Quick Start Setup

Follow these steps to set up your first automated backup:

### Step 1: Install the Extension

Install the **Azure DevOps Backup & Restore** extension from the Visual Studio Marketplace or deploy it to your organization.

### Step 2: Create Variable Group

In your Azure DevOps project:
1. Go to **Pipelines** → **Library**
2. Click **+ Variable group**
3. Name it **"ADO Backup Restore"**
4. Add the required variables (see [Prerequisites](#prerequisites) below)
5. Mark sensitive values as **Secret**
6. Click **Save**

### Step 3: Create Your First Backup Pipeline

1. Go to **Pipelines** → **Pipelines**
2. Click **New pipeline**
3. Choose **Azure Repos Git** (or your source)
4. Select your repository
5. Choose **Starter pipeline**
6. Replace the YAML with the [Basic Incremental Backup](#example-1-basic-incremental-backup) example below
7. Click **Save and run**

### Step 4: Configure Schedule

The example pipeline includes a daily schedule. Adjust the `cron` expression to match your needs:
- Daily at 2 AM: `"0 2 * * *"`
- Sundays at 1 AM: `"0 1 * * 0"`
- Weekdays at 3 AM: `"0 3 * * 1-5"`

That's it! Your backups will now run automatically according to the schedule.

---

## Prerequisites

### 1. Variable Group Setup

Create a variable group named **"ADO Backup Restore"** with the following variables:

| Variable Name | Description | Secret |
|---------------|-------------|--------|
| `Backup.LicenseKey` | Your license key | 🔒 Yes |
| `Backup.AdoPat.RO` | Read-only PAT for backups | 🔒 Yes |
| `Backup.AdoPat.RW` | Read-write PAT for restores | 🔒 Yes |
| `Backup.Root` | Backup root directory path | ❌ No |

**To create the variable group:**
1. Go to Pipelines → Library
2. Click "+ Variable group"
3. Name it "ADO Backup Restore"
4. Add the variables listed above
5. Mark PAT and License variables as "Secret"
6. Save

### 2. Install the Extension

Install the **Azure DevOps Backup Task** extension from the marketplace (or deploy manually).

### 3. Agent Requirements

Ensure your build agent has:
- .NET 9 Runtime installed
- Sufficient disk space for backups
- Network access to Azure DevOps

## Setting Up the Pipeline

### Directory Structure

Your repository should have:
```
/
├── azure-pipelines/
│   ├── backup-pipeline.yml
│   └── restore-pipeline.yml
├── backups/                    # Created by pipeline
│   ├── logs/
│   └── metadata/
└── README.md
```

## Backup Pipeline Examples

### Example 1: Basic Incremental Backup

**File:** `azure-pipelines/backup-pipeline.yml`

```yaml
name: BackupPipeline_$(Date:yyyyMMdd)$(Rev:.r)

trigger: none

schedules:
  - cron: "0 2 * * *"  # Run daily at 2 AM
    displayName: Daily Backup
    branches:
      include:
        - main
    always: true

pool:
  vmImage: 'windows-latest'

variables:
  - group: 'ADO Backup Restore'
  - name: 'dotnetVersion'
    value: '9.0.x'

jobs:
- job: BackupJob
  displayName: 'Backup Azure DevOps'
  timeoutInMinutes: 120

  steps:
    - checkout: self
      persistCredentials: true

    - task: UseDotNet@2
      displayName: 'Install .NET 9'
      inputs:
        packageType: 'runtime'
        version: '$(dotnetVersion)'

    - task: AzureDevOpsBackupTask@0
      displayName: 'Perform Incremental Backup'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
      inputs:
        Pat: '$(Backup.AdoPat.RO)'
        BackupRoot: '$(Backup.Root)'
        Projects: ''  # Empty = all projects
        Verbose: true
        BackupMode: 'Incremental'  # Incremental backup
        SkipGit: false
        SkipBuilds: false
        SkipWorkItems: false
        SkipVariables: false
        SkipQueries: false

    # Upload logs as artifacts
    - task: PublishBuildArtifacts@1
      displayName: 'Upload Backup Logs'
      condition: always()
      inputs:
        PathtoPublish: '$(Backup.Root)/logs'
        ArtifactName: 'BackupLogs-$(Build.BuildNumber)'
        publishLocation: 'Container'
      continueOnError: true

    # Upload metadata for tracking
    - task: PublishBuildArtifacts@1
      displayName: 'Upload Backup Metadata'
      condition: succeeded()
      inputs:
        PathtoPublish: '$(Backup.Root)/metadata'
        ArtifactName: 'BackupMetadata-$(Build.BuildNumber)'
        publishLocation: 'Container'
      continueOnError: true
```

### Example 2: Selective Backup (Git Only)

```yaml
name: GitBackupPipeline_$(Date:yyyyMMdd)$(Rev:.r)

trigger: none

schedules:
  - cron: "0 3 * * *"
    displayName: Daily Git Backup
    branches:
      include:
        - main

pool:
  vmImage: 'windows-latest'

variables:
  - group: 'ADO Backup Restore'

jobs:
- job: GitBackupJob
  displayName: 'Backup Git Repositories Only'

  steps:
    - checkout: self

    - task: AzureDevOpsBackupTask@0
      displayName: 'Backup Git Repositories'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
      inputs:
        Pat: '$(Backup.AdoPat.RO)'
        BackupRoot: '$(Backup.Root)'
        Projects: 'Project1,Project2'  # Specific projects
        Verbose: true
        BackupMode: 'Full'
        SkipGit: false
        SkipBuilds: true    # Skip builds
        SkipWorkItems: true # Skip work items
        SkipVariables: true # Skip variables
        SkipQueries: true   # Skip queries
        GitRepositories: 'Repo1,Repo2'  # Specific repos

    - task: PublishBuildArtifacts@1
      displayName: 'Upload Logs'
      condition: always()
      inputs:
        PathtoPublish: '$(Backup.Root)/logs'
        ArtifactName: 'GitBackupLogs-$(Build.BuildNumber)'
```

### Example 3: Full Weekly + Incremental Daily

```yaml
name: SmartBackupPipeline_$(Date:yyyyMMdd)$(Rev:.r)

trigger: none

schedules:
  # Full backup on Sundays at 1 AM
  - cron: "0 1 * * 0"
    displayName: Weekly Full Backup
    branches:
      include:
        - main
  # Incremental backup weekdays at 2 AM
  - cron: "0 2 * * 1-5"
    displayName: Daily Incremental Backup
    branches:
      include:
        - main

pool:
  vmImage: 'windows-latest'

variables:
  - group: 'ADO Backup Restore'

jobs:
- job: SmartBackupJob
  displayName: 'Smart Backup Strategy'

  steps:
    - checkout: self

    - powershell: |
        $dayOfWeek = (Get-Date).DayOfWeek
        if ($dayOfWeek -eq 'Sunday') {
          Write-Host "##vso[task.setvariable variable=BackupMode]Full"
          Write-Host "Running FULL backup (Sunday)"
        } else {
          Write-Host "##vso[task.setvariable variable=BackupMode]Incremental"
          Write-Host "Running INCREMENTAL backup (Weekday)"
        }
      displayName: 'Determine Backup Mode'

    - task: AzureDevOpsBackupTask@0
      displayName: 'Perform $(BackupMode) Backup'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
      inputs:
        Pat: '$(Backup.AdoPat.RO)'
        BackupRoot: '$(Backup.Root)'
        Verbose: true
        BackupMode: '$(BackupMode)'

    - task: PublishBuildArtifacts@1
      condition: always()
      inputs:
        PathtoPublish: '$(Backup.Root)/logs'
        ArtifactName: 'Logs-$(BackupMode)-$(Build.BuildNumber)'
```

### Example 4: Self-Hosted Agent with Custom Path

```yaml
name: BackupPipeline_$(Date:yyyyMMdd)$(Rev:.r)

trigger: none

pool:
  name: 'Default'  # Self-hosted agent pool

variables:
  - group: 'ADO Backup Restore'

jobs:
- job: BackupJob
  displayName: 'Backup on Self-Hosted Agent'

  steps:
    - checkout: self

    - task: AzureDevOpsBackupTask@0
      displayName: 'Full Backup'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
      inputs:
        Pat: '$(Backup.AdoPat.RO)'
        BackupRoot: 'D:\ADOBackups'  # Custom path on self-hosted agent
        Verbose: true
        BackupMode: 'Incremental'

    - task: PublishBuildArtifacts@1
      condition: always()
      inputs:
        PathtoPublish: 'D:\ADOBackups\logs'
        ArtifactName: 'BackupLogs-$(Build.BuildNumber)'
```

## Restore Pipeline Examples

### Example 1: Basic Restore with Parameters

**File:** `azure-pipelines/restore-pipeline.yml`

```yaml
name: RestorePipeline_$(Date:yyyyMMdd)$(Rev:.r)

trigger: none

parameters:
  - name: 'projectsToRestore'
    displayName: 'Projects to Restore'
    type: string
    default: 'MyProject'
  
  - name: 'targetProject'
    displayName: 'Target Project Name'
    type: string
    default: 'RestoreTestProject'
  
  - name: 'dryRun'
    displayName: 'Dry Run (Preview Only)'
    type: boolean
    default: true
  
  - name: 'skipGit'
    displayName: 'Skip Git Repositories'
    type: boolean
    default: false
  
  - name: 'skipBuilds'
    displayName: 'Skip Build Definitions'
    type: boolean
    default: false
  
  - name: 'skipWorkItems'
    displayName: 'Skip Work Items'
    type: boolean
    default: false

pool:
  vmImage: 'windows-latest'

variables:
  - group: 'ADO Backup Restore'

jobs:
- job: RestoreJob
  displayName: 'Restore Azure DevOps'

  steps:
    - checkout: self

    - task: AzureDevOpsRestoreTask@0
      displayName: 'Perform Azure DevOps Restore'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
      inputs:
        Pat: '$(Backup.AdoPat.RW)'  # Read-Write PAT required
        BackupRoot: '$(Backup.Root)'
        Projects: '${{ parameters.projectsToRestore }}'
        TargetProject: '${{ parameters.targetProject }}'
        TargetOrganizationUrl: ''  # Empty = same org
        TargetPat: ''  # Empty = same PAT
        DryRun: ${{ parameters.dryRun }}
        Verbose: true
        SkipGit: ${{ parameters.skipGit }}
        SkipBuilds: ${{ parameters.skipBuilds }}
        SkipWorkItems: ${{ parameters.skipWorkItems }}
        SkipVariables: false
        SkipQueries: false
        BypassRules: true

    - task: PublishBuildArtifacts@1
      displayName: 'Upload Restore Logs'
      condition: always()
      inputs:
        PathtoPublish: '$(Backup.Root)/logs'
        ArtifactName: 'RestoreLogs-$(Build.BuildNumber)'
        publishLocation: 'Container'
      continueOnError: true
```

### Example 2: Cross-Organization Restore

```yaml
name: CrossOrgRestorePipeline_$(Date:yyyyMMdd)$(Rev:.r)

trigger: none

parameters:
  - name: 'sourceProject'
    displayName: 'Source Project'
    type: string
    default: 'ProductionProject'
  
  - name: 'targetOrg'
    displayName: 'Target Organization URL'
    type: string
    default: 'https://dev.azure.com/testorg'
  
  - name: 'targetProject'
    displayName: 'Target Project'
    type: string
    default: 'TestProject'

pool:
  vmImage: 'windows-latest'

variables:
  - group: 'ADO Backup Restore'
  - group: 'ADO Target Org'  # Separate variable group for target org

jobs:
- job: CrossOrgRestoreJob
  displayName: 'Cross-Organization Restore'

  steps:
    - checkout: self

    - task: AzureDevOpsRestoreTask@0
      displayName: 'Restore to Target Organization'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
      inputs:
        Pat: '$(Backup.AdoPat.RO)'  # Source org PAT (read-only OK)
        BackupRoot: '$(Backup.Root)'
        Projects: '${{ parameters.sourceProject }}'
        TargetOrganizationUrl: '${{ parameters.targetOrg }}'
        TargetPat: '$(Target.AdoPat.RW)'  # Target org PAT (read-write)
        TargetProject: '${{ parameters.targetProject }}'
        DryRun: false
        Verbose: true
        BypassRules: true

    - task: PublishBuildArtifacts@1
      condition: always()
      inputs:
        PathtoPublish: '$(Backup.Root)/logs'
        ArtifactName: 'CrossOrgRestoreLogs-$(Build.BuildNumber)'
```

### Example 3: Selective Restore with Resource Filters

```yaml
name: SelectiveRestorePipeline_$(Date:yyyyMMdd)$(Rev:.r)

trigger: none

parameters:
  - name: 'gitRepositories'
    displayName: 'Git Repositories to Restore (comma-separated)'
    type: string
    default: 'Repo1,Repo2'
  
  - name: 'buildDefinitions'
    displayName: 'Build Definition IDs (comma-separated)'
    type: string
    default: '1,2,5'
  
  - name: 'workitemIds'
    displayName: 'Work Item IDs (comma-separated)'
    type: string
    default: '100,101,102'

pool:
  vmImage: 'windows-latest'

variables:
  - group: 'ADO Backup Restore'

jobs:
- job: SelectiveRestoreJob
  displayName: 'Selective Resource Restore'

  steps:
    - checkout: self

    - task: AzureDevOpsRestoreTask@0
      displayName: 'Restore Selected Resources'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
      inputs:
        Pat: '$(Backup.AdoPat.RW)'
        BackupRoot: '$(Backup.Root)'
        Projects: 'MyProject'
        Verbose: true
        DryRun: false
        GitRepositories: '${{ parameters.gitRepositories }}'
        BuildDefinitions: '${{ parameters.buildDefinitions }}'
        WorkItemIds: '${{ parameters.workitemIds }}'
        SkipVariables: true
        SkipQueries: true

    - task: PublishBuildArtifacts@1
      condition: always()
      inputs:
        PathtoPublish: '$(Backup.Root)/logs'
        ArtifactName: 'SelectiveRestoreLogs-$(Build.BuildNumber)'
```

## Best Practices

### 1. Security

✅ **DO:**
- Store PAT tokens as secret variables
- Use separate PATs for backup (read-only) and restore (read-write)
- Rotate PAT tokens regularly
- Use variable groups for centralized secret management
- Restrict pipeline permissions to specific service accounts

❌ **DON'T:**
- Hard-code PAT tokens in YAML files
- Commit secrets to source control
- Use overly permissive PAT scopes
- Share PAT tokens across environments

### 2. Storage Management

✅ **DO:**
- Use self-hosted agents with dedicated backup storage
- Implement backup retention policies
- Monitor disk space usage
- Compress old backups
- Store backups on separate infrastructure

❌ **DON'T:**
- Store backups on the same infrastructure as source
- Keep unlimited backup history
- Use Microsoft-hosted agents for large backups (limited disk space)

### 3. Scheduling

✅ **DO:**
- Schedule backups during low-usage periods
- Use incremental mode for daily backups
- Perform full backups weekly/monthly
- Stagger backup times for multiple organizations
- Set appropriate timeout values

❌ **DON'T:**
- Run backups during business hours
- Run multiple full backups simultaneously
- Use overly aggressive schedules (causes rate limiting)

### 4. Monitoring

✅ **DO:**
- Enable verbose logging
- Publish logs as artifacts
- Set up failure notifications
- Monitor backup duration trends
- Validate backups regularly with test restores

❌ **DON'T:**
- Ignore backup failures
- Assume backups are working without validation
- Skip log review

### 5. Testing

✅ **DO:**
- Test restore procedures regularly
- Use dry-run mode for validation
- Maintain a test organization for restore testing
- Document restore procedures
- Practice disaster recovery scenarios

❌ **DON'T:**
- Wait for a disaster to test restores
- Skip dry-run validation
- Assume backups are restorable without testing

## Troubleshooting

### Issue: Pipeline Timeout

**Symptoms:** Pipeline exceeds timeout limit during backup.

**Solutions:**
1. Increase `timeoutInMinutes` in job definition
2. Use incremental mode instead of full backup
3. Increase `MaxParallelism` setting
4. Backup projects individually
5. Use self-hosted agent with better resources

### Issue: Authentication Failed

**Symptoms:** "401 Unauthorized" or "403 Forbidden" errors.

**Solutions:**
1. Verify PAT token hasn't expired
2. Check PAT has required scopes
3. Ensure variable group is linked to pipeline
4. Verify PAT is marked as secret
5. Check organization allows PAT authentication

### Issue: Rate Limiting

**Symptoms:** "429 Too Many Requests" errors.

**Solutions:**
1. Reduce `MaxParallelism` setting
2. Space out backup schedules
3. Use incremental mode
4. Contact Azure DevOps support for rate limit increase

### Issue: Out of Disk Space

**Symptoms:** Pipeline fails with disk space errors.

**Solutions:**
1. Use self-hosted agent with more disk space
2. Implement backup cleanup/retention policy
3. Backup specific projects instead of all
4. Compress old backups

### Issue: Logs Not Published

**Symptoms:** Artifacts not appearing in pipeline.

**Solutions:**
1. Verify `PathtoPublish` exists
2. Check `condition: always()` is set
3. Ensure agent has write permissions
4. Verify artifact name doesn't contain special characters

## Advanced Scenarios

### Scenario 1: Multi-Stage Backup and Validation

```yaml
stages:
  - stage: Backup
    jobs:
      - job: BackupJob
        steps:
          - task: AzureDevOpsBackupTask@0
            env:
              ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
            inputs:
              Pat: '$(Backup.AdoPat.RO)'
              # Additional backup configuration
  
  - stage: Validate
    dependsOn: Backup
    jobs:
      - job: ValidateJob
        steps:
          - powershell: |
              # Validate backup integrity
              $backupPath = "$(Backup.Root)"
              if (!(Test-Path "$backupPath/metadata")) {
                Write-Error "Backup metadata not found"
                exit 1
              }
              Write-Host "Backup validation passed"
```

### Scenario 2: Conditional Restore

```yaml
- powershell: |
    # Only restore if backup is recent
    $metadataFile = "$(Backup.Root)/metadata/builds/metadata.json"
    $metadata = Get-Content $metadataFile | ConvertFrom-Json
    $lastBackup = [DateTime]::Parse($metadata.LastBackupDate)
    $hoursSinceBackup = ((Get-Date) - $lastBackup).TotalHours
    
    if ($hoursSinceBackup -gt 24) {
      Write-Error "Backup is too old (>24 hours)"
      exit 1
    }
  displayName: 'Check Backup Freshness'

- task: AzureDevOpsRestoreTask@0
  # Restore configuration
```

## Example Repository Structure

```
azure-devops-backup-automation/
├── azure-pipelines/
│   ├── backup-full.yml
│   ├── backup-incremental.yml
│   ├── restore-test.yml
│   └── restore-production.yml
├── scripts/
│   ├── validate-backup.ps1
│   ├── cleanup-old-backups.ps1
│   └── notify-backup-status.ps1
├── docs/
│   ├── runbook.md
│   └── disaster-recovery-plan.md
└── README.md
```

## Next Steps

Now that you have pipeline tasks set up:

1. **[Review Best Practices](./best-practices.md)** - Optimize your backup strategy
2. **[Explore Use Cases](./use-cases.md)** - Common scenarios and solutions
3. **[Troubleshooting Guide](./troubleshooting.md)** - Common issues and solutions

### CLI Alternative

If you need manual operations or scripting outside of pipelines:
- **[Getting Started with CLI](./getting-started.md)** - CLI installation and usage
- **[Command Reference](./command-reference.md)** - Complete CLI command documentation

## See Also

- [Azure Pipelines Documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/)
- [Azure DevOps Personal Access Tokens](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate)
