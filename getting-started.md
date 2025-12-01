# Getting Started with Azure DevOps Backup & Restore

This guide will walk you through setting up and using the Azure DevOps Backup & Restore utility for the first time.

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

**For Restore (Read/Write operations):**
- Code: Read & Write
- Build: Read & Write
- Work Items: Read & Write
- Variable Groups: Read & Write
- Project and Team: Read & Write

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

?? **Security Note:** Never commit PAT tokens to source control. Use environment variables or secure vaults for CI/CD pipelines.

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
  ??? TestProject\
  ?   ??? git\
  ?   ??? builds\
  ?   ??? workitems\
  ?   ??? variables\
  ?   ??? queries\
  ??? metadata\
  ??? logs\
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

?? **Warning:** This will overwrite existing resources.

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
  --skip-builds ^
  --skip-workitems ^
  --skip-variables ^
  --skip-queries ^
  -v
```

**Restore only build definitions:**
```bash
adobackup.exe restore-all ^
  --BackupRoot "C:\ADOBackups" ^
  -p "TestProject" ^
  --skip-git ^
  --skip-workitems ^
  --skip-variables ^
  --skip-queries ^
  -v
```

## Next Steps

Now that you've completed your first backup and restore:

1. **[Read the Command Reference](./command-reference.md)** - Learn about all available commands and options
2. **[Review Best Practices](./best-practices.md)** - Optimize your backup strategy
3. **[Set up Pipeline Integration](./pipeline-integration.md)** - Automate backups with Azure Pipelines
4. **[Explore Use Cases](./use-cases.md)** - Learn common scenarios and solutions

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

**Next:** [Command Reference ?](./command-reference.md)
