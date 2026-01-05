# Azure DevOps Backup & Restore Solution

A comprehensive solution for backing up and restoring Azure DevOps resources across organizations and projects, available as Azure Pipeline tasks or a command-line utility.

## Overview

The Azure DevOps Backup & Restore Utility enables you to:
- **Backup** Azure DevOps resources including Git repositories, pull requests, build definitions, service connections, work items, pipeline variables, and queries
- **Restore** resources to the same or different organizations and projects
- **Migrate** projects and resources between Azure DevOps organizations
- **Clone** projects within the same organization
- **Incremental backups** to capture only new or changed data since the last backup

### Primary Workflow: Azure Pipeline Tasks

The **recommended approach** is to use the Azure DevOps Pipeline Tasks for automated backup and restore operations:
- **AzureDevOpsBackupTask** - For backup operations
- **AzureDevOpsRestoreTask** - For restore operations

These tasks integrate seamlessly into your CI/CD pipelines, providing scheduled automated backups and controlled restore operations.

### Alternative: Command-Line Interface

For manual operations, scripting, or environments without pipeline access, the CLI utility (`adobackup.exe`) provides full functionality through command-line interface. See [Command Reference](./command-reference.md) for complete CLI documentation.

## Key Features

💾 **Comprehensive Backup Coverage**
- Git repositories (full clone with all branches and history)
- Pull requests (metadata, comments, reviews, status)
- Build definitions and build history
- Service connections (metadata, credentials excluded)
- Work items with full history and attachments
- Pipeline variables (secrets excluded)
- Shared queries and query folders

🔄 **Flexible Restore Options**
- Restore to same or different organization
- Restore to same or different project
- Selective restore (choose specific resources)
- Dry-run mode to preview changes

⚡ **Advanced Capabilities**
- Incremental backup mode for continuous backup strategies
- Parallel processing for improved performance
- Comprehensive logging and error handling
- Metadata tracking for backup history

🏢 **Enterprise Ready**
- Support for large-scale operations
- Rate limiting and retry logic
- Configurable via command-line or config file

## Quick Start

### Prerequisites

- Azure DevOps organization and project
- Valid Azure DevOps Personal Access Token (PAT) with appropriate permissions
- License key (for production use)
- For pipeline tasks: Azure Pipelines enabled in your project
- For CLI: .NET 9 Runtime installed

### Option 1: Using Pipeline Tasks (Recommended)

**1. Install the Extension**

Install the **Azure DevOps Backup & Restore** extension from the Azure DevOps Marketplace.

**2. Create a Variable Group**

Create a variable group named **"ADO Backup Restore"** with:
- `Backup.LicenseKey` (secret)
- `Backup.AdoPat.RO` (secret, read-only PAT)
- `Backup.AdoPat.RW` (secret, read-write PAT)
- `Backup.Root` (backup directory path)

**3. Create a Backup Pipeline**

```yaml
trigger: none

schedules:
  - cron: "0 2 * * *"  # Daily at 2 AM
    displayName: Daily Incremental Backup
    branches:
      include:
        - main

pool:
  vmImage: 'windows-latest'

variables:
  - group: 'ADO Backup Restore'

jobs:
- job: BackupJob
  displayName: 'Backup Azure DevOps'
  
  steps:
    - task: AzureDevOpsBackupTask@0
      displayName: 'Perform Incremental Backup'
      env:
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
      inputs:
        Pat: '$(Backup.AdoPat.RO)'
        BackupRoot: '$(Backup.Root)'
        Projects: '*'  # All projects
        Verbose: true
        BackupMode: 'Incremental'
        BackupAll: true
```

See [Pipeline Integration Guide](./pipeline-integration.md) for complete setup instructions.

### Option 2: Using CLI

**1. Install**

Download and extract the latest release, then activate your license:
```bash
adobackup.exe license-activate -k "your-license-key"
```

**2. Backup**
```bash
adobackup.exe backup-all --OrganizationUrl "https://dev.azure.com/yourorg" --Pat "your-pat-token" --BackupRoot "C:\Backups" -i -v
```

**3. Restore**
```bash
adobackup.exe restore-all --OrganizationUrl "https://dev.azure.com/yourorg" --Pat "your-pat-token" --BackupRoot "C:\Backups" -v
```

See [Command Reference](./command-reference.md) for all CLI commands and options.

## Documentation

### Getting Started
- **[Pipeline Integration Guide](./pipeline-integration.md)** - **Start here!** Setup Azure Pipeline tasks (recommended)
- **[Getting Started with CLI](./getting-started.md)** - CLI installation and basic usage

### Reference
- **[Command Reference](./command-reference.md)** - Complete CLI command documentation
- **[Best Practices](./best-practices.md)** - Recommended usage patterns
- **[Use Cases](./use-cases.md)** - Common scenarios and examples

### Help & Support
- **[Troubleshooting](./troubleshooting.md)** - Common issues and solutions
- **[FAQ](./faq.md)** - Frequently asked questions

## Use Cases

**Note:** This utility supports **backup and restore operations only**. It does not provide migration, synchronization, or multi-organization management features.

### 1. Disaster Recovery
Create regular backups of your Azure DevOps organization to protect against data loss.

### 2. Selective Backup
Backup specific resources (e.g., only Git repositories or only work items).

### 3. Continuous Backup
Use incremental mode in automated pipelines for continuous data protection.


## Architecture

The utility consists of several components:

- **Pipeline Tasks** - Azure DevOps pipeline task extensions for automated operations
  - **AzureDevOpsBackupTask** - Backup task integration
  - **AzureDevOpsRestoreTask** - Restore task integration
- **CLI** - Command-line interface (`adobackup.exe`) for manual operations
- **Core Library** - Business logic and Azure DevOps integration
- **Services** - Backup/restore services for each resource type
- **Azure DevOps Client** - API wrapper with rate limiting and retry logic
- **License System** - Secure license validation and activation

## Supported Azure DevOps Resources

| Resource Type | Backup | Restore | Incremental | Notes |
|--------------|--------|---------|-------------|-------|
| Git Repositories | ✅ | ✅ | ✅ | Full clone with all branches |
| Pull Requests | ✅ | ✅ | | PR metadata, comments, reviews, status |
| Build Definitions | ✅ | ✅  |  | Includes configuration |
| Build History | ✅ |   |   | Up to 100 builds per definition - backup reference |
| Service Connections | ✅ | ✅ | | Connection metadata (credentials excluded) |
| Work Items | ✅ | ✅ | ✅ | All fields and attachments |
| Pipeline Variables | ✅ | ✅ |  | Secrets excluded |
| Queries | ✅ | ✅ | | Shared queries and folders |

## Permissions Required

Your Azure DevOps PAT token needs the following permissions:

**For Backup:**
- Code (Read)
- Build (Read)
- Work Items (Read)
- Variable Groups (Read)
- Service Connections (Read)
- Project and Team (Read)

**For Restore:**
- Code (Read, Write)
- Build (Read, Write)
- Work Items (Read, Write)
- Variable Groups (Read, Write)
- Service Connections (Read, Write)
- Project and Team (Read, Write)

## System Requirements

- **Operating System:** Windows 10+, Windows Server 2016+, Linux, macOS
- **.NET Runtime:** .NET 9.0 or later
- **Disk Space:** Varies based on repository and backup size
- **Memory:** Minimum 4GB RAM (8GB+ recommended for large organizations)
- **Network:** Stable internet connection to Azure DevOps

## License

This software requires a valid license key for production use. Contact Lamdat for licensing information.

## Support

For support, bug reports, or feature requests:
- Email: support@lamdat.com
- Documentation: [docs/](./docs/)

 ---

**Copyright � 2025 Lamdat. All rights reserved.**
