# Azure DevOps Backup & Restore Utility

A comprehensive command-line tool for backing up and restoring Azure DevOps resources across organizations and projects.

## Overview

The Azure DevOps Backup & Restore Utility (`adobackup`) is a powerful tool that enables you to:
- **Backup** Azure DevOps resources including Git repositories, build definitions, work items, pipeline variables, and queries
- **Restore** resources to the same or different organizations and projects
- **Migrate** projects and resources between Azure DevOps organizations
- **Clone** projects within the same organization
- **Incremental backups** to capture only new or changed data since the last backup

## Key Features

? **Comprehensive Backup Coverage**
- Git repositories (full clone with all branches and history)
- Build definitions and build history
- Work items with full history and attachments
- Pipeline variables (including secret variables)
- Shared queries and query folders

? **Flexible Restore Options**
- Restore to same or different organization
- Restore to same or different project
- Selective restore (choose specific resources)
- Dry-run mode to preview changes
- Work item rules bypass for different configurations

? **Advanced Capabilities**
- Incremental backup mode for continuous backup strategies
- Parallel processing for improved performance
- Comprehensive logging and error handling
- Metadata tracking for backup history
- Cross-organization migration support

? **Enterprise Ready**
- License-based activation system
- Support for large-scale operations
- Rate limiting and retry logic
- Configurable via command-line or config file

## Quick Start

### Prerequisites

- .NET 9 Runtime
- Valid Azure DevOps Personal Access Token (PAT) with appropriate permissions
- License key (for production use)

### Installation

1. Download the latest release from the releases page
2. Extract to your desired location
3. Activate your license:
   ```bash
   adobackup.exe license-activate -k "your-license-key"
   ```

### Basic Usage

**Full Backup:**
```bash
adobackup.exe backup-all --OrganizationUrl "https://dev.azure.com/yourorg" --Pat "your-pat-token" --BackupRoot "C:\Backups"
```

**Incremental Backup:**
```bash
adobackup.exe backup-all -i --OrganizationUrl "https://dev.azure.com/yourorg" --Pat "your-pat-token" --BackupRoot "C:\Backups"
```

**Full Restore:**
```bash
adobackup.exe restore-all --OrganizationUrl "https://dev.azure.com/yourorg" --Pat "your-pat-token" --BackupRoot "C:\Backups"
```

**Restore to Different Project:**
```bash
adobackup.exe restore-all -p "SourceProject" --target-project "TargetProject" --OrganizationUrl "https://dev.azure.com/yourorg" --Pat "your-pat-token" --BackupRoot "C:\Backups"
```

## Documentation

- **[User Guide](./user-guide.md)** - Comprehensive guide for all features
- **[Command Reference](./command-reference.md)** - Detailed documentation for all commands
- **[Getting Started](./getting-started.md)** - Step-by-step tutorials
- **[Pipeline Integration](./pipeline-integration.md)** - Using with Azure Pipelines
- **[Best Practices](./best-practices.md)** - Recommended usage patterns
- **[Troubleshooting](./troubleshooting.md)** - Common issues and solutions
- **[FAQ](./faq.md)** - Frequently asked questions

## Use Cases

### 1. Disaster Recovery
Create regular backups of your Azure DevOps organization to protect against data loss.

### 2. Project Migration
Move projects between Azure DevOps organizations or tenants.

### 3. Project Cloning
Create copies of projects for testing, development, or training environments.

### 4. Selective Backup
Backup specific resources (e.g., only Git repositories or only work items).

### 5. Continuous Backup
Use incremental mode in automated pipelines for continuous data protection.

## Architecture

The utility consists of several components:

- **CLI** - Command-line interface (`adobackup.exe`)
- **Core Library** - Business logic and Azure DevOps integration
- **Services** - Backup/restore services for each resource type
- **Azure DevOps Client** - API wrapper with rate limiting and retry logic
- **License System** - Secure license validation and activation

## Supported Azure DevOps Resources

| Resource Type | Backup | Restore | Incremental | Notes |
|--------------|--------|---------|-------------|-------|
| Git Repositories | ? | ? | ? | Full clone with all branches |
| Build Definitions | ? | ? | ? | Includes configuration |
| Build History | ? | ? | ? | Up to 100 builds per definition |
| Work Items | ? | ? | ? | Full history and attachments |
| Pipeline Variables | ? | ? | ? | Including secret variables |
| Queries | ? | ? | ? | Shared queries and folders |

## Permissions Required

Your Azure DevOps PAT token needs the following permissions:

**For Backup:**
- Code (Read)
- Build (Read)
- Work Items (Read)
- Variable Groups (Read)
- Project and Team (Read)

**For Restore:**
- Code (Read, Write)
- Build (Read, Write)
- Work Items (Read, Write)
- Variable Groups (Read, Write)
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
- Internal Documentation: [docs/internal/](./internal/)

## Version History

- **v1.0.0** - Initial release
  - Full backup and restore capabilities
  - Support for all major Azure DevOps resources
  - Incremental backup mode
  - Cross-organization migration

---

**Copyright © 2025 Lamdat. All rights reserved.**
