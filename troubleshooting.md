# Troubleshooting Guide

Solutions to common issues and error messages when using the Azure DevOps Backup & Restore utility.

## Table of Contents

1. [General Troubleshooting Steps](#general-troubleshooting-steps)
2. [Authentication Issues](#authentication-issues)
3. [Backup Issues](#backup-issues)
4. [Restore Issues](#restore-issues)
5. [Performance Issues](#performance-issues)
6. [Network Issues](#network-issues)
7. [Storage Issues](#storage-issues)
8. [License Issues](#license-issues)
9. [Pipeline Integration Issues](#pipeline-integration-issues)
10. [Error Messages Reference](#error-messages-reference)

---

## General Troubleshooting Steps

When encountering any issue, follow these steps:

### 1. Enable Verbose Logging

Always run with `-v` flag for detailed output:

```bash
adobackup.exe backup-all -v
```

### 2. Check the Logs

Logs are stored in `{BackupRoot}/logs/` directory. Review the most recent log file for error details.

```powershell
# View latest log file
Get-Content (Get-ChildItem "D:\ADOBackups\logs" | Sort-Object LastWriteTime -Descending | Select-Object -First 1).FullName
```

### 3. Verify Prerequisites

- [ ] .NET 9 runtime installed
- [ ] Valid PAT token with correct scopes
- [ ] Sufficient disk space
- [ ] Network connectivity to Azure DevOps
- [ ] Valid license key

### 4. Test Connectivity

```powershell
# Test connectivity to Azure DevOps
Test-NetConnection dev.azure.com -Port 443
```

### 5. Validate Configuration

```bash
# Test with minimal command
adobackup.exe --version

# Validate license
adobackup.exe license-validate -k "your-license-key"
```

---

## Authentication Issues

### Issue: "401 Unauthorized"

**Error Message:**
```
Failed to authenticate with Azure DevOps: 401 Unauthorized
```

**Causes:**
1. PAT token expired
2. PAT token invalid
3. PAT token doesn't have required scopes
4. Organization requires different authentication

**Solutions:**

**1. Verify PAT token:**
```bash
# Test PAT with simple API call
$pat = "your-pat-token"
$base64Pat = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$pat"))
Invoke-RestMethod -Uri "https://dev.azure.com/yourorg/_apis/projects?api-version=7.0" -Headers @{Authorization = "Basic $base64Pat"}
```

**2. Regenerate PAT token:**
- Go to Azure DevOps ? User Settings ? Personal Access Tokens
- Create new token with all required scopes
- Update command or configuration file

**3. Verify required scopes:**

For backup: Code (Read), Build (Read), Work Items (Read), Variable Groups (Read), Project and Team (Read)
For restore: Code (Read & Write), Build (Read & Write), Work Items (Read & Write), Variable Groups (Read & Write), Project and Team (Read & Write)

**4. Check organization settings:**
- Ensure PAT tokens are allowed in organization
- Verify user has access to organization

### Issue: "403 Forbidden"

**Error Message:**
```
Access denied: 403 Forbidden
```

**Causes:**
1. Insufficient PAT permissions
2. User doesn't have access to resource
3. Organization policies block access

**Solutions:**

**1. Grant additional permissions:**
```bash
# Verify PAT scopes in Azure DevOps
# Recreate PAT with full read/write scopes
```

**2. Check user permissions:**
- Verify user is member of organization
- Check project-level permissions
- Ensure not blocked by conditional access policies

**3. Check organization policies:**
- Go to Organization Settings ? Policies
- Verify "Third-party application access via OAuth" is enabled
- Check if IP restrictions are in place

### Issue: "Token expired"

**Error Message:**
```
Personal Access Token has expired
```

**Solution:**

Recreate PAT token with appropriate expiration date:

```bash
# Update configuration with new token
adobackup.exe backup-all --Pat "new-pat-token" -v
```

**Best Practice:** Set calendar reminders before PAT expiration to renew proactively.

---

## Backup Issues

### Issue: Backup Fails Immediately

**Error Message:**
```
Failed to initialize backup: Invalid configuration
```

**Causes:**
1. Invalid command-line arguments
2. Missing required parameters
3. Invalid backup root path
4. License validation failure

**Solutions:**

**1. Verify command syntax:**
```bash
# Get help for backup command
adobackup.exe backup-all --help

# Check all required parameters are provided
adobackup.exe backup-all \
  --OrganizationUrl "https://dev.azure.com/yourorg" \
  --Pat "your-pat" \
  --BackupRoot "D:\ADOBackups" \
  -v
```

**2. Validate backup root directory:**
```powershell
# Ensure directory exists and is writable
New-Item -ItemType Directory -Path "D:\ADOBackups" -Force
```

**3. Check license:**
```bash
adobackup.exe license-validate -k "your-license-key"
```

### Issue: Partial Backup Completion

**Error Message:**
```
Backup completed with errors: 5 resources failed
```

**Causes:**
1. Network interruptions
2. Rate limiting
3. Individual resource access issues
4. Disk space exhaustion mid-backup

**Solutions:**

**1. Re-run backup in incremental mode:**
```bash
# Incremental mode will skip successfully backed up resources
adobackup.exe backup-all -i -v
```

**2. Check verbose logs:**
```bash
# Identify which resources failed
adobackup.exe backup-all -v 2>&1 | Select-String "Failed"
```

**3. Backup failed resources individually:**
```bash
# If Git repos failed
adobackup.exe git-backup -p "ProjectName" -v

# If work items failed
adobackup.exe workitems-backup -p "ProjectName" -v
```

### Issue: Build Backup Limited to 100 Builds

**Error Message:**
```
maxBuilds cannot exceed 100 (requested: 150). This is an Azure DevOps API limitation.
```

**Cause:**
Azure DevOps API limits build queries to 100 builds per request.

**Solutions:**

**1. Use incremental backup mode:**
```bash
# First run: backup last 100 builds
adobackup.exe build-backup --max-builds 100 -v

# Subsequent runs: backup new builds since last run
adobackup.exe build-backup -i -v
```

**2. Use time-based filtering:**
```bash
# Backup builds from specific time periods
adobackup.exe build-backup --days 30 -v
```

**3. Run multiple backups over time:**
```bash
# Today: last 100 builds
adobackup.exe build-backup --max-builds 100 -v

# Tomorrow: incremental (new builds)
adobackup.exe build-backup -i -v
```

**Note:** This is an Azure DevOps API limitation, not a utility limitation.

### Issue: Git Clone Failures

**Error Message:**
```
Failed to clone repository: Repository not found
```

**Causes:**
1. Repository doesn't exist
2. Repository is empty
3. Insufficient permissions
4. Network timeout

**Solutions:**

**1. Verify repository exists:**
```bash
# List all repositories in project
curl -u :your-pat "https://dev.azure.com/yourorg/YourProject/_apis/git/repositories?api-version=7.0"
```

**2. Skip empty repositories:**
Empty repositories will fail to clone. This is expected behavior - they will be skipped automatically on retry.

**3. Increase network timeout:**
```bash
# For large repositories, increase timeout
git config --global http.postBuffer 524288000
git config --global http.timeout 600
```

**4. Clone individually for troubleshooting:**
```bash
adobackup.exe git-backup -p "ProjectName" -r "SpecificRepo" -v
```

### Issue: Work Items Not Backing Up

**Error Message:**
```
Failed to backup work items: No work items found in range
```

**Causes:**
1. No work items exist in specified range
2. Work items deleted
3. Query permissions issue

**Solutions:**

**1. Verify work items exist:**
```bash
# Check work item count in Azure DevOps
# Go to Boards ? Work Items

# Or use API
curl -u :your-pat "https://dev.azure.com/yourorg/YourProject/_apis/wit/workitems?ids=1,2,3&api-version=7.0"
```

**2. Adjust ID range:**
```bash
# Use auto-detection (don't specify range)
adobackup.exe workitems-backup -p "ProjectName" -v

# Or specify correct range
adobackup.exe workitems-backup --min-id 1 --max-id 10000 -v
```

**3. Check permissions:**
Ensure PAT has "Work Items (Read)" scope.

---

## Restore Issues

### Issue: "Target project not found"

**Error Message:**
```
Failed to restore: Target project 'ProjectName' not found
```

**Cause:**
Target project doesn't exist in target organization.

**Solution:**

Create the target project first:

```bash
# In Azure DevOps
1. Go to target organization
2. Click "New Project"
3. Enter project name (must match --target-project parameter)
4. Create project

# Then run restore
adobackup.exe restore-all -p "SourceProject" --target-project "NewProject" -v
```

### Issue: Resource Conflicts During Restore

**Error Message:**
```
Failed to restore: Resource already exists with same name
```

**Causes:**
1. Repository with same name exists
2. Build definition with same name exists
3. Variable group with same name exists

**Solutions:**

**1. Use dry-run to preview conflicts:**
```bash
adobackup.exe restore-all --dry-run -v
```

**2. Delete conflicting resources manually:**
```bash
# In Azure DevOps, delete existing resources before restore
```

**3. Restore to different project:**
```bash
adobackup.exe restore-all -p "Source" --target-project "NewProject" -v
```

### Issue: Work Item Restore Validation Errors

**Error Message:**
```
Failed to create work item: Required field 'FieldName' is missing
```

**Causes:**
1. Different work item type definitions
2. Different process templates
3. Custom field requirements

**Solutions:**

**1. Enable bypass rules:**
```bash
# Bypass work item validation rules (default)
adobackup.exe workitems-restore --bypass-rules true -v
```

**2. Align process templates:**
- Ensure source and target use compatible process templates
- Customize target project to match source

**3. Restore selectively:**
Skip problematic work items and restore manually:
```bash
# Restore specific work items
adobackup.exe workitems-restore -i "1,2,5,10" -v
```

### Issue: Restore is Very Slow

**Causes:**
1. Network latency
2. Large data volume
3. Low parallelism
4. Target organization performance

**Solutions:**

**1. Increase parallelism:**
```bash
adobackup.exe restore-all --MaxParallelism 8 -v
```

**2. Restore selectively:**
```bash
# Restore only critical resources first
adobackup.exe restore-all --skip-git -v  # Skip large Git repos

# Then restore Git separately
adobackup.exe git-restore -v
```

**3. Monitor progress:**
```bash
# Use verbose mode to see progress
adobackup.exe restore-all -v
```

---

## Performance Issues

### Issue: Backup Takes Too Long

**Expected vs Actual:**
- Expected: 30 minutes
- Actual: 4 hours

**Causes:**
1. Large organization size
2. Slow network
3. Low parallelism
4. Rate limiting
5. Slow storage

**Solutions:**

**1. Use incremental backups:**
```bash
# After first full backup, use incremental
adobackup.exe backup-all -i -v
```

**2. Increase parallelism:**
```bash
# Carefully increase (watch for rate limiting)
adobackup.exe backup-all --MaxParallelism 8 -v
```

**3. Backup specific resources:**
```bash
# Backup only critical resources
adobackup.exe backup-all -p "Project1,Project2" --skip-git -v
```

**4. Use faster storage:**
- Move backup directory to SSD
- Use local storage instead of network share

**5. Run on Azure VM:**
- Deploy in same Azure region as DevOps
- Reduces network latency

### Issue: High Memory Usage

**Symptoms:**
- Process uses 4+ GB RAM
- System becomes unresponsive
- Out of memory errors

**Causes:**
1. Large repositories being cloned
2. High parallelism
3. Many work items with attachments

**Solutions:**

**1. Reduce parallelism:**
```bash
adobackup.exe backup-all --MaxParallelism 2 -v
```

**2. Backup in batches:**
```bash
# Backup projects one at a time
adobackup.exe backup-all -p "Project1" -v
adobackup.exe backup-all -p "Project2" -v
```

**3. Increase available memory:**
- Close other applications
- Use dedicated backup server
- Upgrade RAM

### Issue: Rate Limiting (429 Too Many Requests)

**Error Message:**
```
Rate limit exceeded: 429 Too Many Requests. Retrying after 60 seconds...
```

**Causes:**
1. Too many API requests too quickly
2. High parallelism setting
3. Multiple backup processes running
4. Organization tier limits

**Solutions:**

**1. Reduce parallelism:**
```bash
# Lower parallelism to reduce API load
adobackup.exe backup-all --MaxParallelism 2 -v
```

**2. Space out backups:**
```yaml
# In pipeline, schedule with gaps
schedules:
  - cron: "0 2 * * *"  # 2 AM
  - cron: "0 14 * * *" # 2 PM (12 hours later)
```

**3. Use incremental mode:**
```bash
# Reduces number of API calls
adobackup.exe backup-all -i -v
```

**4. Wait and retry:**
The utility automatically retries with exponential backoff. Wait for completion.

**5. Contact Azure DevOps support:**
Request rate limit increase for your organization tier.

---

## Network Issues

### Issue: Connection Timeout

**Error Message:**
```
Connection timeout: Failed to connect to dev.azure.com
```

**Causes:**
1. Network connectivity issue
2. Firewall blocking connection
3. Proxy configuration
4. Azure DevOps service outage

**Solutions:**

**1. Test connectivity:**
```powershell
Test-NetConnection dev.azure.com -Port 443
```

**2. Check firewall rules:**
```powershell
# Allow outbound HTTPS
New-NetFirewallRule -DisplayName "Azure DevOps" -Direction Outbound -RemoteAddress dev.azure.com -RemotePort 443 -Protocol TCP -Action Allow
```

**3. Configure proxy (if applicable):**
```bash
# Set proxy environment variables
$env:HTTPS_PROXY = "http://proxy:8080"
adobackup.exe backup-all -v
```

**4. Check Azure DevOps status:**
Visit https://status.dev.azure.com/ to check for outages.

### Issue: Intermittent Network Failures

**Symptoms:**
- Backup succeeds sometimes, fails other times
- Random connection drops
- Timeout errors

**Solutions:**

**1. Enable automatic retry (built-in):**
The utility automatically retries failed operations with exponential backoff.

**2. Run during stable network hours:**
```yaml
# Schedule during off-peak hours
schedules:
  - cron: "0 2 * * *"  # 2 AM when network is stable
```

**3. Use wired connection:**
Switch from WiFi to wired Ethernet for stability.

**4. Monitor network:**
```powershell
# Log network statistics during backup
while ($true) { 
    Test-NetConnection dev.azure.com -InformationLevel Detailed | 
    Out-File -Append "network-log.txt"
    Start-Sleep 60
}
```

---

## Storage Issues

### Issue: Out of Disk Space

**Error Message:**
```
Insufficient disk space: Failed to write to D:\ADOBackups
```

**Causes:**
1. Backup directory filled disk
2. Disk quota exceeded
3. No cleanup of old backups

**Solutions:**

**1. Free up space:**
```powershell
# Delete old backups
Remove-Item "D:\ADOBackups\2024*" -Recurse -Force

# Compress old backups
Compress-Archive -Path "D:\ADOBackups\2024-12" -DestinationPath "D:\ADOBackups\2024-12.zip"
Remove-Item "D:\ADOBackups\2024-12" -Recurse -Force
```

**2. Change backup location:**
```bash
# Use different drive with more space
adobackup.exe backup-all --BackupRoot "E:\ADOBackups" -v
```

**3. Implement retention policy:**
```powershell
# Auto-cleanup old backups (keep last 7 days)
Get-ChildItem "D:\ADOBackups" -Directory | 
    Where-Object { $_.CreationTime -lt (Get-Date).AddDays(-7) } |
    Remove-Item -Recurse -Force
```

**4. Monitor disk space:**
```powershell
# Alert when disk space is low
$drive = Get-PSDrive D
$freeSpaceGB = [math]::Round($drive.Free / 1GB, 2)
if ($freeSpaceGB -lt 50) {
    Write-Warning "Low disk space: $freeSpaceGB GB remaining"
}
```

### Issue: Permission Denied Writing to Backup Directory

**Error Message:**
```
Access denied: Failed to write to D:\ADOBackups\ProjectA
```

**Causes:**
1. Insufficient file system permissions
2. Directory locked by another process
3. Antivirus blocking access

**Solutions:**

**1. Grant permissions:**
```powershell
# Grant full control to current user
$acl = Get-Acl "D:\ADOBackups"
$permission = "$env:USERDOMAIN\$env:USERNAME", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow"
$accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule $permission
$acl.SetAccessRule($accessRule)
Set-Acl "D:\ADOBackups" $acl
```

**2. Run as administrator:**
```bash
# Right-click CMD/PowerShell ? "Run as Administrator"
adobackup.exe backup-all --BackupRoot "D:\ADOBackups" -v
```

**3. Check for locks:**
```powershell
# Find processes with open handles
Get-Process | Where-Object { $_.Handles -gt 1000 } | Select-Object Name, Handles
```

**4. Add antivirus exception:**
Add `D:\ADOBackups` to antivirus exclusion list.

---

## License Issues

### Issue: "License validation failed"

**Error Message:**
```
License validation failed: Invalid license key
```

**Causes:**
1. Incorrect license key
2. License key expired
3. License key for different tenant
4. License file corrupted

**Solutions:**

**1. Verify license key:**
```bash
# Check license key is correct (no extra spaces/characters)
adobackup.exe license-validate -k "your-license-key"
```

**2. Re-activate license:**
```bash
# Delete old license file
Remove-Item "license.lic" -Force

# Activate with fresh license
adobackup.exe license-activate -k "your-license-key"
```

**3. Check license file location:**
```bash
# Specify license file location explicitly
adobackup.exe license-activate -k "your-license-key" -f "C:\Licenses\ado-backup.lic"
```

**4. Contact support:**
If issue persists, contact Lamdat support with:
- License key (partial, last 4 characters)
- Error message
- Organization URL

### Issue: License Works on One Server but Not Another

**Cause:**
License may be tied to specific server/tenant.

**Solutions:**

**1. Check license restrictions:**
```bash
adobackup.exe license-validate -k "your-license-key" -v
```

**2. Request multi-server license:**
Contact Lamdat for licensing options supporting multiple servers.

**3. Copy license file:**
```bash
# Copy activated license to other servers
Copy-Item "license.lic" "\\server2\app\license.lic"
```

---

## Pipeline Integration Issues

### Issue: Pipeline Can't Find adobackup.exe

**Error Message:**
```
'adobackup.exe' is not recognized as an internal or external command
```

**Causes:**
1. Utility not installed on build agent
2. Incorrect path
3. Extension not installed

**Solutions:**

**1. Install extension:**
Ensure "Azure DevOps Backup Task" extension is installed in organization.

**2. Check agent has .NET 9:**
```yaml
steps:
  - task: UseDotNet@2
    displayName: 'Install .NET 9'
    inputs:
      packageType: 'runtime'
      version: '9.0.x'
```

**3. Use full path:**
```yaml
- script: |
    "C:\Tools\adobackup\adobackup.exe" backup-all -v
  displayName: 'Run Backup'
```

### Issue: Pipeline Timeout

**Error Message:**
```
Job timeout: The job has exceeded the maximum execution time
```

**Causes:**
1. Default timeout too short (60 minutes)
2. Backup takes longer than expected
3. Rate limiting causing delays

**Solutions:**

**1. Increase timeout:**
```yaml
jobs:
- job: BackupJob
  timeoutInMinutes: 240  # 4 hours
  steps:
    - task: AzureDevOpsBackupTask@0
```

**2. Use incremental mode:**
```yaml
inputs:
  BackupMode: 'Incremental'
```

**3. Split into multiple jobs:**
```yaml
jobs:
- job: BackupGit
  timeoutInMinutes: 120
  steps:
    - task: AzureDevOpsBackupTask@0
      inputs:
        SkipBuilds: true
        SkipWorkItems: true

- job: BackupBuilds
  timeoutInMinutes: 120
  steps:
    - task: AzureDevOpsBackupTask@0
      inputs:
        SkipGit: true
        SkipWorkItems: true
```

### Issue: Secret Variables Not Working

**Error Message:**
```
Failed to authenticate: PAT token is empty
```

**Causes:**
1. Variable group not linked to pipeline
2. Variable not marked as secret
3. Incorrect variable reference

**Solutions:**

**1. Link variable group:**
```yaml
variables:
  - group: 'ADO Backup Restore'  # Must be exact name
```

**2. Verify variable exists:**
- Go to Pipelines ? Library ? Variable groups
- Check "ADO Backup Restore" group exists
- Verify `Backup.AdoPat.RO` variable exists and is marked secret

**3. Use correct syntax:**
```yaml
inputs:
  Pat: '$(Backup.AdoPat.RO)'  # Use $() syntax, not ${}
```

---

## Error Messages Reference

### Authentication Errors

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "401 Unauthorized" | PAT expired/invalid | Regenerate PAT |
| "403 Forbidden" | Insufficient permissions | Add required scopes to PAT |
| "Token expired" | PAT expiration date passed | Create new PAT |

### Backup Errors

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "Failed to clone repository" | Git clone failure | Check permissions, network |
| "No work items found" | Invalid ID range | Adjust --min-id/--max-id |
| "maxBuilds cannot exceed 100" | API limitation | Use incremental mode or --days |
| "Insufficient disk space" | Out of disk space | Free up space or change location |

### Restore Errors

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "Target project not found" | Project doesn't exist | Create project first |
| "Resource already exists" | Name conflict | Delete existing or use different project |
| "Required field missing" | Work item validation | Use --bypass-rules true |

### Performance Errors

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "429 Too Many Requests" | Rate limiting | Reduce --MaxParallelism |
| "Connection timeout" | Network issue | Check network, increase timeout |
| "Out of memory" | Insufficient RAM | Reduce parallelism, add RAM |

### License Errors

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "Invalid license key" | Wrong key | Verify license key |
| "License expired" | License expired | Renew license |
| "License validation failed" | Various | Re-activate license |

---

## Getting More Help

If your issue isn't covered here:

1. **Check logs:** Review `{BackupRoot}/logs/` with `-v` flag
2. **Search FAQ:** See [FAQ](./faq.md) for common questions
3. **Review documentation:** Check [Command Reference](./command-reference.md)
4. **Contact support:** Email support@lamdat.com with:
   - Detailed error description
   - Log files
   - Command used
   - Environment details (OS, .NET version, organization size)

---

**Last Updated:** January 2025
