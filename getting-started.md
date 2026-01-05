# Getting Started with Azure DevOps Backup & Restore CLI

This guide covers the command-line interface (CLI) for Azure DevOps Backup & Restore.

> **💡 Recommended Approach:** For most users, we recommend using the **Azure Pipeline Tasks** instead of the CLI for automated, scheduled backups. See the [Pipeline Integration Guide](./pipeline-integration.md) to get started with pipeline tasks.
>
> **Use this CLI guide if you need:**
> - Manual backup/restore operations
> - Scripting outside of Azure Pipelines
> - On-demand operations without pipeline setup

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [Your First Backup](#your-first-backup)
5. [Your First Restore](#your-first-restore)
6. [Next Steps](#next-steps)

## Prerequisites

### 1. .NET 9 Runtime

Download and install the .NET 9 Runtime from [Microsoft .NET Downloads](https://dotnet.microsoft.com/download/dotnet/9.0).

Verify installation:
```bash
dotnet --version
```

### 2. Azure DevOps Personal Access Token (PAT)

Create a PAT token with the following scopes:

**For Backup (Read-only operations):**
- Code: Read
- Build: Read
- Work Items: Read
- Variable Groups: Read
- Project and Team: Read
- Service Connections: Read (for service connections backup)

**For Restore (Read/Write operations):**
- Code: Read & Write
- Build: Read & Write
- Work Items: Read & Write
- Variable Groups: Read & Write
- Project and Team: Read & Write
- Service Connections: Read & Write (for service connections restore)

**To create a PAT:**
1. Go to Azure DevOps
2. Click on User Settings (top right) ? Personal Access Tokens
3. Click "New Token"
4. Set expiration and select scopes
5. Copy and save the token securely

### 3. License Key

Contact Lamdat to obtain a license key for production use.

## Installation

### Option 1: Download Release Package

1. Download the latest release from the releases page
2. Extract to your desired location (e.g., `C:\Tools\adobackup`)
3. Add the installation directory to your PATH (optional)

### Option 2: Build from Source

```bash
git clone https://dev.azure.com/Lamdat/Lamdat/_git/ado-backup-restore
cd ado-backup-restore
dotnet build ADOBackupRestore.CLI/Lamdat.ADOBackupRestore.CLI.csproj -c Release
```

The built executable will be in `ADOBackupRestore.CLI/bin/Release/net9.0/`

## Configuration

### Activate License

Before using the utility, activate your license:

```bash
adobackup.exe license-activate -k "your-license-key-here"
```

This creates a `license.lic` file in the application directory.

### Validate License

Verify your license is active:

```bash
adobackup.exe license-validate -k "your-license-key-here"
```

### Configuration Methods

You can configure the utility using:

1. **Command-line arguments** (recommended for CI/CD)
2. **Configuration file** (recommended for local use)
3. **Environment variables**

#### Configuration File (appsettings.json)

Create an `appsettings.json` file in the application directory:

```json
{
  "Settings": {
    "OrganizationUrl": "https://dev.azure.com/yourorg",
    "Pat": "your-pat-token",
    "BackupRoot": "C:\\ADOBackups",
    "MaxParallelism": 4
  }
}
```

⚠️ **Security Note:** Never commit PAT tokens to source control. Use environment variables or secure vaults for CI/CD pipelines.

## Your First Backup

### Step 1: Test Connection

Verify connectivity to Azure DevOps:

```bash
adobackup.exe backup-all --help
```

### Step 2: Perform a Test Backup

Start with a single project:

```bash
adobackup.exe backup-all ^
  --OrganizationUrl "https://dev.azure.com/yourorg" ^
  --Pat "your-pat-token" ^
  --BackupRoot "C:\ADOBackups" ^
  -p "TestProject" ^
  -v
```

**Command breakdown:**
- `backup-all` - Backup all resource types
- `--OrganizationUrl` - Your Azure DevOps organization URL
- `--Pat` - Your Personal Access Token
- `--BackupRoot` - Directory where backups will be saved
- `-p "TestProject"` - Specific project to backup (optional)
- `-v` - Verbose output for detailed logging

### Step 3: Verify Backup

Check the backup directory:

```bash
C:\ADOBackups\
  └── TestProject\
      ├── git\
      ├── pullrequests\
      ├── builds\
      ├── serviceconnections\
      ├── workitems\
      ├── variables\
      └── queries\
  ├── metadata\
  └── logs\
```

### Step 4: Full Organization Backup

Once you've verified a test backup works, perform a full backup:

```bash
adobackup.exe backup-all ^
  --OrganizationUrl "https://dev.azure.com/yourorg" ^
  --Pat "your-pat-token" ^
  --BackupRoot "C:\ADOBackups" ^
  -v
```

This will backup all projects and all resource types.

### Step 5: Incremental Backup

For subsequent backups, use incremental mode to only backup new/changed data:

```bash
adobackup.exe backup-all ^
  --OrganizationUrl "https://dev.azure.com/yourorg" ^
  --Pat "your-pat-token" ^
  --BackupRoot "C:\ADOBackups" ^
  -i ^
  -v
```

The `-i` flag enables incremental mode, which:
- Backs up work items created/modified since last backup
- Backs up builds from the last backup date
- Always backs up latest definitions

## Your First Restore

### Step 1: Dry Run

Always start with a dry run to preview changes:

```bash
adobackup.exe restore-all ^
  --OrganizationUrl "https://dev.azure.com/yourorg" ^
  --Pat "your-pat-token" ^
  --BackupRoot "C:\ADOBackups" ^
  -p "TestProject" ^
  --dry-run ^
  -v
```

The `--dry-run` flag shows what would be restored without making any changes.

### Step 2: Restore to Same Project

Restore the test project back to itself:

```bash
adobackup.exe restore-all ^
  --OrganizationUrl "https://dev.azure.com/yourorg" ^
  --Pat "your-pat-token" ^
  --BackupRoot "C:\ADOBackups" ^
  -p "TestProject" ^
  -v
```

⚠️ **Warning:** This will overwrite existing resources.

### Step 3: Restore to Different Project

Clone the project to a new project:

```bash
adobackup.exe restore-all ^
  --OrganizationUrl "https://dev.azure.com/yourorg" ^
  --Pat "your-pat-token" ^
  --BackupRoot "C:\ADOBackups" ^
  -p "TestProject" ^
  --target-project "TestProject-Copy" ^
  -v
```

The `--target-project` must exist in the target organization before running the restore.

### Step 4: Restore to Different Organization

Migrate a project to a different Azure DevOps organization:

```bash
adobackup.exe restore-all ^
  --OrganizationUrl "https://dev.azure.com/yourorg" ^
  --Pat "your-source-pat" ^
  --BackupRoot "C:\ADOBackups" ^
  -p "TestProject" ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "your-target-pat" ^
  --target-project "MigratedProject" ^
  -v
```

### Step 5: Selective Restore

Restore only specific resource types:

**Restore only Git repositories:**
```bash
adobackup.exe restore-all ^
  --BackupRoot "C:\ADOBackups" ^
  -p "TestProject" ^
  --include-git ^
  -v
```

**Restore only build definitions:**
```bash
adobackup.exe restore-all ^
  --BackupRoot "C:\ADOBackups" ^
  -p "TestProject" ^
  --include-builds ^
  -v
```

## ⚠️ Limitations & Known Issues

Understanding the limitations helps you plan your backup and restore strategy effectively. Most limitations are due to Azure DevOps API constraints or Git protocol requirements.

### API & Platform Limitations

| Component | Limitation | Impact | Workaround/Notes |
|-----------|-----------|--------|------------------|
| **Work Items** | Restores to same project only | Work items restored with same IDs | Deleted work items are recreated with new IDs if needed |
| **Work Items** | Restore updates existing work items without adding comments | No audit trail of restore operation | Work item history shows field changes but not restore event |
| **Work Items** | Incremental backup uses daily granularity, not exact timestamp | May backup some items twice if run multiple times per day | Incremental mode tracks last backup date (YYYY-MM-DD), not time |
| **Work Items** | Deleted work items are recreated with new IDs during restore | Original IDs cannot be preserved | Links and relationships are recreated with new IDs |
| **Queries** | Query definitions restored/updated as-is | Existing queries are updated with backup data | Verify area paths and iteration paths exist in project before restore |
| **Pull Requests** | Restored as metadata only | Code changes are in Git history | Comments, reviews, and status are preserved |
| **Pull Requests** | PR IDs preserved only when restoring to same project | Cross-project restore may create new IDs | Work item links in PRs are preserved if work items exist |
| **Service Connections** | Secrets/credentials are NOT backed up | Credentials will be lost | Must manually re-enter secrets after restore |
| **Service Connections** | Pipeline authorizations must be reconfigured | Pipelines won't have access until reauthorized | Manually authorize pipelines to use connections after restore |
| **Git Repositories** | Non-fast-forward pushes fail if remote history diverged | Restore fails if repository changed after backup | See Git-specific limitations below |
| **Azure DevOps API** | Rate limiting varies by organization tier | Backup may be throttled on high-volume operations | Reduce `MaxParallelism`, space out backups, or contact Microsoft for limit increase |
| **Build History** | Maximum 100 builds per definition per backup run | Can't backup more than 100 builds in single run | Use `--days` parameter with incremental mode to capture builds over time, build history is kept locally for reference and cannot be restored due to Azure DevOps limitations |
| **Build Definitions** | Existing definitions are **updated** during restore | Definition configuration is overwritten | Review changes before restore; use `--dry-run` to preview |
| **Variable Groups** | Passwords/secrets are NOT backed up | Secret values will be lost | Document secret variables separately; re-enter after restore |
| **Variable Groups** | Existing variable groups are **updated** during restore | Variable group values are overwritten (except secrets) | Secrets must be manually re-entered after restore |

### Git-Specific Limitations

| Scenario | Issue | Error Message | Solution |
|----------|-------|---------------|----------|
| **Diverged History** | Remote repository has commits not in backup | `NonFastForwardException: cannot push because a reference that you are trying to update on the remote contains commits that are not present locally` | **Option 1:** Force push (loses remote commits)<br>**Option 2:** Manually merge changes<br>**Option 3:** Delete remote repo and restore fresh |
| **Branch Protection** | Branch policies prevent force push | Push fails due to branch policies | Temporarily disable branch policies during restore |
| **Large Repositories** | Git clone/push timeout on repos >50GB | Operation timeout or memory errors | Increase timeout, use `--MaxParallelism 1` for large repos |
| **LFS Objects** | Git LFS files require separate authentication | LFS files may not restore correctly | Ensure LFS credentials are configured |

### Security Limitations

| Feature | Limitation | Recommendation |
|---------|-----------|----------------|
| **Secret Variables** | Pipeline variable secrets are NOT backed up | Document secrets separately; use Azure Key Vault |
| **PAT Tokens** | PAT tokens used in pipelines are NOT backed up | Re-configure service connections after restore |
| **SSH Keys** | Git SSH keys are NOT backed up | Re-upload SSH keys to target organization |
| **Service Connections** | Service connections/endpoints are NOT backed up | Manually recreate service connections if needed |

### Restore Behavior & Important Notes

| Component | Behavior | Important Notes |
|-----------|----------|-----------------|
| **Work Items** | Updates existing work item if found by ID | No comments added to work item history about restore operation |
| **Work Items** | Creates new work item if original was deleted | New ID assigned; relationships recreated |
| **Build Definitions** | **Updates** existing definition if found by ID | Definition is updated with backup data |
| **Build Definitions** | **Creates** new definition if doesn't exist | New definition created with original configuration |
| **Variable Groups** | **Updates** existing group if found by name | Group is updated with backup data (except secrets) |
| **Variable Groups** | **Creates** new group if doesn't exist | New variable group created |
| **Variable Groups** | Secret values are **NOT** restored | Passwords/secrets are not backed up; must be re-entered manually |
| **Git Repositories** | Uses force push by default | **Warning:** Overwrites remote history; ensure backup is correct version |
| **Queries** | Restores folder structure and hierarchy | Original GUIDs preserved where possible |



## Next Steps

Now that you've completed your first CLI backup and restore:

### Recommended: Automate with Pipeline Tasks

**[Set up Pipeline Integration](./pipeline-integration.md)** - Automate backups with Azure Pipeline tasks (recommended for production)

### CLI Resources

1. **[Command Reference](./command-reference.md)** - Complete CLI command documentation
2. **[Review Best Practices](./best-practices.md)** - Optimize your backup strategy
3. **[Explore Use Cases](./use-cases.md)** - Common scenarios and solutions

## Common Issues

### Issue: "Failed to authenticate"

**Solution:** Verify your PAT token has the required permissions and hasn't expired.

### Issue: "Access denied" during restore

**Solution:** Ensure your PAT token has write permissions for restore operations.

### Issue: Backup is very slow

**Solution:** Increase parallelism with `--MaxParallelism 8` (default is 4).

### Issue: Out of disk space

**Solution:** 
- Use incremental mode (`-i`)
- Clean up old backups
- Backup specific projects/resources instead of everything

For more troubleshooting, see the [Troubleshooting Guide](./troubleshooting.md).

## Getting Help

- Review the [FAQ](./faq.md)
- Check the [Troubleshooting Guide](./troubleshooting.md)
- Contact support: support@lamdat.com

---

**Next:** [Command Reference →](./command-reference.md)
