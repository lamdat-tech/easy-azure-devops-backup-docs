# Quick Start - ADO Backup & Restore VSIX Extension

Get up and running with the ADO Backup & Restore extension in 5 minutes!

## 5-Minute Setup

### Step 1: Build the VSIX Package (2 minutes)

```bash
cd ADOBackupRestore.VSIX

# Install dependencies
npm install

# Create VSIX package
npm run package
```

Expected output:
```
ado-backup-restore-extension-1.0.0.vsix
```

### Step 2: Prepare the CLI (1 minute)

```bash
# Build CLI
cd ../ADOBackupRestore.CLI
dotnet publish -c Release -r win-x64 --self-contained

# Copy to accessible location (or include in VSIX)
# Create: ADOBackupRestore.VSIX/tools/
# Copy: adobackup.exe
```

### Step 3: Deploy to Azure DevOps (1 minute)

**Option A: To Private Collection** (Easiest for testing)
1. Go to Organization Settings ? Extensions ? Manage Extensions
2. Click "Upload new extension"
3. Select the `.vsix` file
4. Click "Upload"

**Option B: To Marketplace** (For sharing)
1. Create publisher account at marketplace.visualstudio.com
2. Update `vss-extension.json` with your publisher ID
3. Run: `tfx extension publish --manifest-globs vss-extension.json`

### Step 4: Create Service Connection (1 minute)

1. Go to Project Settings ? Service Connections
2. New Service Connection ? Azure DevOps
3. Name: `ADO_ServiceConnection`
4. Create PAT token and add to connection

### Step 5: Use in Pipeline (0 minutes - you're done!)

Add to your pipeline YAML:

```yaml
steps:
  - task: BuildBackupTask@1
    displayName: 'Backup Builds'
    inputs:
      connectedServiceName: 'ADO_ServiceConnection'
 Projects: ''
      IncludeHistory: true
      Verbose: true
```

## Common Tasks

### Backup All Builds

```yaml
- task: BuildBackupTask@1
  displayName: 'Backup All Build Definitions'
  inputs:
    connectedServiceName: 'ADO_ServiceConnection'
    IncludeHistory: true
    MaxBuilds: '100'
 Verbose: true
```

### Backup Specific Projects

```yaml
- task: BuildBackupTask@1
  displayName: 'Backup Project Builds'
  inputs:
    connectedServiceName: 'ADO_ServiceConnection'
    Projects: 'MyProject1,MyProject2'
    Verbose: true
```

### Restore from Backup

```yaml
- task: BuildRestoreTask@1
  displayName: 'Restore Builds'
  inputs:
    connectedServiceName: 'ADO_ServiceConnection'
    BackupDate: '2024-01-15-14-30-00'
    Projects: 'SourceProject'
  TargetProject: 'TargetProject'
    DryRun: true  # Preview first
    Verbose: true
```

### Backup Git Repositories

```yaml
- task: GitBackupTask@1
  displayName: 'Backup Repositories'
  inputs:
    connectedServiceName: 'ADO_ServiceConnection'
    Verbose: true
```

### Backup Work Items

```yaml
- task: WorkItemsBackupTask@1
  displayName: 'Backup Work Items'
  inputs:
    connectedServiceName: 'ADO_ServiceConnection'
    Projects: 'MyProject'
    Verbose: true
```

### Backup Pipeline Variables

```yaml
- task: PipelineVariablesBackupTask@1
  displayName: 'Backup Variables'
  inputs:
    connectedServiceName: 'ADO_ServiceConnection'
    Projects: 'MyProject'
    Verbose: true
```

## Task Reference

| Task | Command | Purpose |
|------|---------|---------|
| BuildBackupTask | build-backup | Backup build definitions + history |
| BuildRestoreTask | build-restore | Restore builds from backup |
| GitBackupTask | git-backup | Backup git repositories |
| GitRestoreTask | git-restore | Restore repositories from backup |
| WorkItemsBackupTask | workitems-backup | Backup work items |
| WorkItemsRestoreTask | workitems-restore | Restore work items from backup |
| PipelineVariablesBackupTask | pipeline-variables-backup | Backup pipeline variables |
| PipelineVariablesRestoreTask | pipeline-variables-restore | Restore variables from backup |

## Input Parameters Explained

### Backup Tasks

- **Projects**: Comma-separated list (empty = all projects)
- **Definitions/Repositories**: Specific items to backup (empty = all)
- **MinId/MaxId**: Range for work items
- **IncludeHistory**: Include build history (true/false)
- **MaxBuilds**: Max builds per definition (default: 100)
- **Days**: Look back N days (empty = all)
- **Verbose**: Detailed logging (true/false)

### Restore Tasks

- **BackupDate**: Format: `YYYY-MM-DD` or `YYYY-MM-DD-HH-MM-SS` (required)
- **Projects**: Source projects (empty = use backup config)
- **TargetProject**: Destination (empty = original projects)
- **DryRun**: Preview without changes (true/false)
- **Verbose**: Detailed logging (true/false)

## Backup Location

Backups are stored in:
```
%BACKUP_ROOT%/
??? 2024-01-15-14-30-00/
?   ??? builds/
?   ??? repositories/
?   ??? workitems/
?   ??? variables/
??? 2024-01-16-09-15-00/
    ??? ... more backups
```

Configure `BACKUP_ROOT` in your agent environment or pipeline variables.

## Checking Backup Status

Backups include detailed logs:

```bash
# View backup metadata
type "C:\backups\2024-01-15-14-30-00\backup-manifest.json"

# View operation logs
type "C:\backups\2024-01-15-14-30-00\backup.log"
```

## Restore Safety

**Always use DryRun first!**

```yaml
# Step 1: Preview
- task: BuildRestoreTask@1
  inputs:
    connectedServiceName: 'ADO_ServiceConnection'
    BackupDate: '2024-01-15'
    DryRun: true  # Preview only
    Verbose: true

# Step 2: Verify dry-run output
# Step 3: When ready, remove DryRun and re-run
```

## Troubleshooting

### "CLI not found"
- Check CLI path in agent: `echo %ADOBACKUP_CLI_PATH%`
- Verify agent has execute permissions
- Restart build agent

### "Authentication failed"
- Verify service connection credentials
- Check PAT token has required scopes
- Ensure organization URL is correct

### "Permission denied"
- Check user has admin rights in organization
- Verify PAT token permissions
- Run with verbose flag for details

### Tasks not visible in pipeline
- Refresh browser
- Verify extension is installed
- Check extension is not disabled

## Advanced Usage

### Scheduled Backups

Schedule daily backups using Azure Pipelines:

```yaml
trigger: none

schedules:
  - cron: "0 2 * * *"  # 2 AM daily
    displayName: Daily Backup
    branches:
    include:
        - main

steps:
  - task: BuildBackupTask@1
    inputs:
      connectedServiceName: 'ADO_ServiceConnection'
      Verbose: true
```

### Backup Multiple Artifact Types

```yaml
steps:
  - task: BuildBackupTask@1
    displayName: 'Backup Builds'
    inputs:
      connectedServiceName: 'ADO_ServiceConnection'

  - task: GitBackupTask@1
    displayName: 'Backup Repos'
    inputs:
      connectedServiceName: 'ADO_ServiceConnection'

  - task: WorkItemsBackupTask@1
    displayName: 'Backup Work Items'
    inputs:
      connectedServiceName: 'ADO_ServiceConnection'

  - task: PipelineVariablesBackupTask@1
    displayName: 'Backup Variables'
    inputs:
  connectedServiceName: 'ADO_ServiceConnection'
```

### Conditional Restore

```yaml
steps:
  - task: BuildRestoreTask@1
    displayName: 'Restore Builds'
    condition: eq(variables['Build.Reason'], 'Manual')
    inputs:
   connectedServiceName: 'ADO_ServiceConnection'
    BackupDate: '2024-01-15'
```

## Next Steps

1. **Read**: Check `DEPLOYMENT_GUIDE.md` for production setup
2. **Integrate**: Add `CLI_INTEGRATION_GUIDE.md` for advanced configuration
3. **Explore**: Review `BUILD_GUIDE.md` for customization
4. **Support**: Visit repository for issues: https://dev.azure.com/Lamdat/Lamdat/_git/ado-backup-restore

## Need Help?

### Common Issues

**Q: How do I verify the CLI is working?**
```bash
# Test CLI directly
adobackup build-backup --projects TestProject --verbose
```

**Q: Where are backups stored?**
- Check your `appsettings.json` configuration
- Or set `BACKUP_ROOT` environment variable

**Q: Can I restore to a different organization?**
- Service connections are org-specific
- Create separate service connections for each organization
- Use appropriate TargetProject in restore task

**Q: How do I backup only specific builds?**
```yaml
inputs:
  Projects: 'MyProject'
  Definitions: 'BuildPipeline1,BuildPipeline2'
```

**Q: What if restore fails halfway through?**
- Check DryRun mode first
- See verbose logs for exact failure point
- Fix underlying issue and retry

## Success Checklist

- [ ] VSIX package created successfully
- [ ] CLI executable deployed to agents
- [ ] Extension installed in Azure DevOps
- [ ] Service connection configured
- [ ] Test backup completes successfully
- [ ] Test restore with DryRun succeeds
- [ ] Full restore completes successfully
- [ ] Backup and restore are in production pipelines

## Performance Tips

- Use specific Projects/Definitions to reduce scope
- Schedule backups during off-peak hours
- Increase `MaxDegreeOfParallelism` for faster operations
- Use DryRun before large restore operations
- Monitor build agent resource usage during operations

---

**You're ready!** Add the first task to your pipeline and run a test backup. See the `DEPLOYMENT_GUIDE.md` for complete setup instructions.
