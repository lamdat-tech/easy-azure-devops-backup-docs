# Azure DevOps Backup & Restore Solution

A comprehensive solution for backing up and restoring Azure DevOps resources across organizations and projects, available as Azure Pipeline tasks or a command-line utility.

## Overview

The Azure DevOps Backup & Restore Utility enables you to:
- **Backup** Azure DevOps resources including Git repositories, pull requests, build definitions, release definitions, service connections, work items, pipeline variables, and queries
- **Restore** resources to the same or different organizations and projects
- **Migrate** projects and resources between Azure DevOps organizations
- **Clone** projects within the same organization
- **Incremental backups** to capture only new or changed data since the last backup

### Azure Pipeline Tasks

Use the Azure DevOps Pipeline Tasks for automated backup and restore operations:
- **AzureDevOpsBackupTask** - For backup operations
- **AzureDevOpsRestoreTask** - For restore operations

These tasks integrate seamlessly into your CI/CD pipelines, providing scheduled automated backups and controlled restore operations.

## Key Features

💾 **Comprehensive Backup Coverage**
- Git repositories (full clone with all branches and history)
- Pull requests (metadata, comments, reviews, status)
- Build definitions and build history
- Release definitions (complete release pipeline configurations)
- Service connections (metadata, credentials excluded)
- Task groups (definitions and metadata)
- Boards (board configurations, columns, rows, card styles)
- Dashboards (dashboard widgets and layouts)
- Wikis (wiki pages with full content and hierarchy)
- Work items with full history and attachments
- Pipeline variables (secrets excluded)
- Shared queries and query folders
- Areas & Iterations (classification nodes, area paths and iteration paths)
- Test plans, test suites, test cases, and test run history
- Pipeline environments (definitions, resources, approvals, and checks)

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
- **Warning behavior control** - Choose how to handle partial failures:
  - **Warning** (default) - Task succeeds with issues, partial failures shown as warnings
  - **Error** - Treat warnings as errors, task fails on any partial failures
  - **Ignore** - Ignore warnings completely, task succeeds even with partial failures

🏢 **Enterprise Ready**
- Support for large-scale operations
- Rate limiting and retry logic
- Configurable via command-line or config file

## Quick Start

### Prerequisites

- Azure DevOps organization and project
- Azure Pipelines enabled in your project
- Valid Azure DevOps Personal Access Token (PAT) with appropriate permissions
- License key — the license is issued per Azure DevOps organization; a **trial license** is available at [easyadobackup.com](https://easyadobackup.com/)

### Using Pipeline Tasks

**1. Install the Extension**

Install the **Azure DevOps Backup & Restore** extension from the Azure DevOps Marketplace.

**2. Create a Variable Group**

Create a variable group named **"ADO Backup Restore"** with:
- `Backup.LicenseKey` (secret — your license key, issued per organization)
- `Backup.AdoPat.RO` (secret — read-only PAT for backups)
- `Backup.AdoPat.RW` (secret — read-write PAT for restores)
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
        ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)  # Pass secret directly as environment variable
      inputs:
        Pat: '$(Backup.AdoPat.RO)'
        BackupRoot: '$(Backup.Root)'
        Projects: '*'  # All projects
        Verbose: true
        BackupMode: 'Incremental'
        BackupAll: true
```

See [Getting Started Guide](./getting-started.md) for step-by-step setup, or the [Pipeline Integration Guide](./pipeline-integration.md) for complete examples.

## Documentation

### Getting Started
- **[Getting Started Guide](./getting-started.md)** - **Start here!** Step-by-step setup for pipeline tasks
- **[Pipeline Integration Guide](./pipeline-integration.md)** - Complete pipeline examples and advanced configuration

### Reference
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

The solution consists of several components:

- **Pipeline Tasks** - Azure DevOps pipeline task extensions for automated operations
  - **AzureDevOpsBackupTask** - Backup task integration
  - **AzureDevOpsRestoreTask** - Restore task integration
- **Core Library** - Business logic and Azure DevOps integration
- **Services** - Backup/restore services for each resource type
- **Azure DevOps Client** - API wrapper with rate limiting and retry logic
- **License System** - Secure license validation and activation

## Supported Azure DevOps Resources

| Resource Type | Backup | Restore | Incremental | Notes |
|--------------|--------|---------|-------------|-------|
| Git Repositories | ✅ | ✅ | ✅ | Full clone with all branches |
| Pull Requests | ✅ | ✅ | ✅ | PR metadata, comments, reviews, status with intelligent change detection |
| Build Definitions | ✅ | ✅  |  | Includes configuration |
| Build History | ✅ |   |   | Up to 100 builds per definition - backup reference |
| Release Definitions | ✅ | ✅ | | Complete release pipeline configurations (secret variables excluded) |
| Service Connections | ✅ | ✅ | | Connection metadata (credentials excluded) |
| Task Groups | ✅ | ✅ | | Task group definitions and metadata |
| Boards | ✅ | ✅ | | Board configurations, columns, rows |
| Dashboards | ✅ | ✅ | | Dashboard widgets and layouts |
| Wikis | ✅ | ✅ | | Wiki pages with full content and hierarchy |
| Work Items | ✅ | ✅ | ✅ | All fields, attachments, and revision history (`--max-revisions` to limit stored revisions) |
| Pipeline Variables | ✅ | ✅ |  | Secrets excluded |
| Queries | ✅ | ✅ | | Shared queries and folders |
| Areas & Iterations | ✅ | ✅ | | Classification nodes (area paths and iteration paths); additive restore (never deletes); selective restore (areas-only or iterations-only); cross-project and cross-org supported |
| Test Plans | ✅ | ✅ | | Test plans, test suites (static, requirement-based, query-based), test case associations, and test run history (configurable days, default: 90); test runs optional during restore (`--include-test-runs`, same-project only); requirement/query suites auto-converted to static if dependencies missing; plans/suites support cross-project restore |
| Pipeline Environments | ✅ | ✅ | | Environment definitions, resource registrations, approval configurations, and check configurations; create-or-update restore; optional skip of resources/approvals/checks |

## Permissions Required

Your Azure DevOps PAT token needs the following permissions:

**For Backup:**
- Code (Read)
- Build (Read)
- Release (Read)
- Work Items (Read)
- Variable Groups (Read)
- Service Connections (Read)
- Task Groups (Read)
- Environments (Read)
- Wiki (Read)
- Analytics (Read) - for Boards and Dashboards
- Project and Team (Read)

**For Restore:**
- Code (Read, Write)
- Build (Read, Write)
- Release (Read, Write, Execute)
- Work Items (Read, Write)
- Variable Groups (Read, Write)
- Service Connections (Read, Write)
- Task Groups (Read, Write)
- Environments (Read, Write, Manage)
- Wiki (Read, Write)
- Analytics (Read) - for Boards and Dashboards
- Project and Team (Read, Write)

## System Requirements

- **Azure DevOps:** Organization with Azure Pipelines enabled
- **Build Agent:** Self-hosted agent recommended for large backups (sufficient disk space and network access to Azure DevOps)
- **Disk Space:** Varies based on repository and backup size
- **Memory:** Minimum 4GB RAM (8GB+ recommended for large organizations)

## License

The license is issued per **Azure DevOps organization**. A **trial license** is available — visit [easyadobackup.com](https://easyadobackup.com/) to request one.

After receiving your license key, store it as a secret variable (`Backup.LicenseKey`) and pass it as an environment variable to each backup and restore task:

```yaml
env:
  ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)  # Pass secret directly as environment variable
```

## Support

For support, bug reports, or feature requests, visit [easyadobackup.com](https://easyadobackup.com/).

 ---

**Copyright � 2025 Lamdat. All rights reserved.**
