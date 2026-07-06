# Getting Started with Azure DevOps Backup & Restore

This guide walks you through setting up your first automated backup using the **Azure DevOps Backup & Restore** pipeline tasks.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [License Setup](#license-setup)
3. [Install the Extension](#install-the-extension)
4. [Create Variable Group](#create-variable-group)
5. [Your First Backup Pipeline](#your-first-backup-pipeline)
6. [Your First Restore Pipeline](#your-first-restore-pipeline)
7. [Next Steps](#next-steps)

## Prerequisites

- An Azure DevOps organization
- Azure Pipelines enabled in your project
- A Personal Access Token (PAT) with appropriate permissions (see below)
- A license key (see [License Setup](#license-setup))

### PAT Permissions

**For Backup (read-only operations):**
- Code: Read
- Build: Read
- Work Items: Read
- Variable Groups: Read
- Project and Team: Read
- Service Connections: Read

**For Restore (read/write operations):**
- Code: Read & Write
- Build: Read & Write
- Work Items: Read & Write
- Variable Groups: Read & Write
- Project and Team: Read & Write
- Service Connections: Read & Write

**To create a PAT:**
1. Go to Azure DevOps
2. Click on User Settings (top right) → Personal Access Tokens
3. Click **New Token**
4. Set expiration and select the required scopes
5. Copy and save the token securely

## License Setup

The license is issued per **Azure DevOps organization**.

- **Trial license:** A trial license is available — visit [easyadobackup.com](https://easyadobackup.com/) to request one.
- **Production license:** Purchase a production license at [easyadobackup.com](https://easyadobackup.com/).

Once you receive your license key, store it as a **secret variable** in your variable group (see [Create Variable Group](#create-variable-group) below).

The license key must be passed as the `ADOBACKUP_LICENSE_KEY` environment variable to **both** the backup task and the restore task:

```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)  # Pass secret directly as environment variable
  inputs:
    # ... task inputs
```

> **Important:** Pass `ADOBACKUP_LICENSE_KEY` as an `env` variable (not just as a task input) to each backup and restore task. This is the required way to pass the license secret securely.

## Install the Extension

Install the **Azure DevOps Backup & Restore** extension from the Visual Studio Marketplace into your Azure DevOps organization.

This provides two pipeline tasks:
- **AzureDevOpsBackupTask** — for backup operations
- **AzureDevOpsRestoreTask** — for restore operations

## Create Variable Group

1. In your Azure DevOps project, go to **Pipelines** → **Library**
2. Click **+ Variable group**
3. Name it **"ADO Backup Restore"**
4. Add the following variables:

| Variable Name | Description | Secret |
|---------------|-------------|--------|
| `Backup.LicenseKey` | Your license key (issued per Azure DevOps organization) | 🔒 Yes |
| `Backup.AdoPat.RO` | Read-only PAT for backup operations | 🔒 Yes |
| `Backup.AdoPat.RW` | Read-write PAT for restore operations | 🔒 Yes |
| `Backup.Root` | Path where backups will be stored (e.g., `C:\ADOBackups`) | ❌ No |

5. Mark the PAT and license variables as **Secret**
6. Click **Save**

## Your First Backup Pipeline

Create a new pipeline file (e.g., `azure-pipelines/backup.yml`):

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
  name: 'Default'  # Use your self-hosted agent pool

variables:
  - group: 'ADO Backup Restore'

jobs:
- job: BackupJob
  displayName: 'Backup Azure DevOps'
  timeoutInMinutes: 720  # 12 hours

  steps:
    - checkout: self
      persistCredentials: true

    - task: AzureDevOpsBackupTask@0
      displayName: 'Perform Incremental Backup'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)  # Pass secret directly as environment variable
      inputs:
        Pat: '$(Backup.AdoPat.RO)'
        BackupRoot: '$(Backup.Root)'
        Projects: '*'  # All projects, or specify comma-separated project names
        Verbose: true
        BackupMode: 'Incremental'
        BackupAll: true

    - task: PublishBuildArtifacts@1
      displayName: 'Upload Backup Logs'
      condition: always()
      inputs:
        PathtoPublish: '$(Backup.Root)/logs'
        ArtifactName: 'BackupLogs-$(Build.BuildNumber)'
        publishLocation: 'Container'
      continueOnError: true
```

**Schedule guidance:**
- Daily at 2 AM: `"0 2 * * *"`
- Sundays at 1 AM: `"0 1 * * 0"`
- Weekdays at 3 AM: `"0 3 * * 1-5"`

## Your First Restore Pipeline

Create a restore pipeline file (e.g., `azure-pipelines/restore.yml`):

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
    default: '<same>'

  - name: 'dryRun'
    displayName: 'Dry Run (Preview Changes)'
    type: boolean
    default: true

pool:
  name: 'Default'  # Use your self-hosted agent pool

variables:
  - group: 'ADO Backup Restore'

jobs:
- job: RestoreJob
  displayName: 'Restore Azure DevOps'

  steps:
    - checkout: self
      persistCredentials: true

    - task: AzureDevOpsRestoreTask@0
      displayName: 'Perform Azure DevOps Restore'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)  # Pass secret directly as environment variable
      inputs:
        Pat: '$(Backup.AdoPat.RW)'
        BackupRoot: '$(Backup.Root)'
        Projects: '${{ parameters.projectsToRestore }}'
        TargetProject: '${{ parameters.targetProject }}'
        DryRun: ${{ parameters.dryRun }}
        Verbose: true
        RestoreAll: true

    - task: PublishBuildArtifacts@1
      displayName: 'Upload Restore Logs'
      condition: always()
      inputs:
        PathtoPublish: '$(Backup.Root)/logs'
        ArtifactName: 'RestoreLogs-$(Build.BuildNumber)'
        publishLocation: 'Container'
      continueOnError: true
```

> **Tip:** Always start with `DryRun: true` to preview what will be restored before making any changes.

## ⚠️ Limitations & Known Issues

Understanding limitations helps you plan your backup and restore strategy. See the full [Limitations Guide](./LIMITATIONS.md) for details.

### Key limitations

| Component | Limitation |
|-----------|-----------|
| **Service Connections** | Secrets/credentials are NOT backed up — must be re-entered after restore |
| **Variable Groups** | Secret values are NOT backed up |
| **Build History** | Maximum 100 builds per definition per backup run |
| **Git Repositories** | Non-fast-forward pushes fail if remote history has diverged |

## Next Steps

- **[Pipeline Integration Guide](./pipeline-integration.md)** — Complete setup guide with advanced examples
- **[Best Practices](./best-practices.md)** — Optimize your backup strategy
- **[Use Cases](./use-cases.md)** — Common scenarios and solutions
- **[Troubleshooting](./troubleshooting.md)** — Common issues and solutions
- **[FAQ](./faq.md)** — Frequently asked questions

## Getting Help

- Review the [FAQ](./faq.md)
- Check the [Troubleshooting Guide](./troubleshooting.md)
- Visit [easyadobackup.com](https://easyadobackup.com/) for support
