# Frequently Asked Questions (FAQ)

Common questions and answers about the Azure DevOps Backup & Restore utility.

## General Questions

### What is adobackup?

adobackup (Azure DevOps Backup & Restore) is a command-line utility that enables comprehensive backup and restore operations for Azure DevOps resources including Git repositories, build definitions, work items, pipeline variables, and queries.

### What Azure DevOps resources can be backed up?

The following resources can be backed up:
- **Git Repositories** - Full clone with all branches and history
- **Build Definitions** - Pipeline configurations
- **Build History** - Up to 100 builds per definition per run
- **Work Items** - Complete history, attachments, and relationships
- **Pipeline Variables** - Variable groups (including secret values)
- **Shared Queries** - Query definitions and folder structures

### Do I need a license?

Yes, a valid license key is required for production use. Contact Lamdat for licensing information.

### What platforms are supported?

- Windows 10 and later
- Windows Server 2016 and later
- Linux (with .NET 9 runtime)
- macOS (with .NET 9 runtime)

### What .NET version is required?

.NET 9.0 runtime or later is required.

---

## Backup Questions

### How long does a backup take?

Backup duration depends on several factors:
- **Organization size** - Number of projects, repositories, work items
- **Network speed** - Connection to Azure DevOps
- **Parallelism** - Number of concurrent operations (default: 4)
- **Backup mode** - Full vs incremental

**Typical timings:**
- Small org (1-5 projects): 5-30 minutes
- Medium org (5-20 projects): 30-120 minutes
- Large org (20+ projects): 2-8 hours

Use incremental mode (`-i`) to significantly reduce backup time after the initial full backup.

### What is the difference between full and incremental backup?

**Full Backup:**
- Backs up all resources regardless of previous backups
- Longer duration
- Higher storage usage
- Recommended for initial backup and weekly/monthly runs

**Incremental Backup:**
- Only backs up new/changed data since last backup
- Faster execution
- Lower storage usage
- Requires metadata from previous backup
- Recommended for daily backups

### How much disk space do I need?

Estimate based on your organization:
- Git repos: ~1.2x current repo size
- Build history: ~1-5 MB per build
- Work items: ~10-50 KB per item
- Other resources: typically < 1 GB total

**Recommendation:** Plan for 2x your estimated size.

### Can I backup only specific resources?

Yes! Use include flags to specify resources:

```bash
# Backup only Git repositories
adobackup.exe backup-all --include-git

# Backup only work items
adobackup.exe backup-all --include-workitems
```

You can also use specific commands like `git-backup`, `build-backup`, etc.

### Can I backup specific projects only?

Yes, use the `-p` parameter:

```bash
adobackup.exe backup-all -p "ProjectA,ProjectB"
```

### Does backup impact my Azure DevOps performance?

Minimal impact. The utility:
- Uses read-only API calls for backup
- Implements rate limiting to respect Azure DevOps limits
- Can be scheduled during off-peak hours
- Parallelism can be tuned to reduce load

### What are the Azure DevOps API limits?

Azure DevOps has rate limits to protect service performance:
- Varies by organization tier (Basic, Basic + Test Plans, Visual Studio subscribers)
- Default: 200 requests per user per hour per organization
- The utility automatically handles rate limiting with retries

If you consistently hit limits, reduce `MaxParallelism` setting.

### Can I schedule automatic backups?

Yes! Integrate with:
- **Azure Pipelines** (recommended) - See [Pipeline Integration Guide](./pipeline-integration.md)
- **Windows Task Scheduler**
- **Cron (Linux/macOS)**
- **Jenkins, GitHub Actions**, or other CI/CD tools

### Why does build backup always include history?

Build history is valuable for audit trails and compliance. The utility always backs up build history (up to 100 builds per definition per run - Azure DevOps API limitation). You can use `--days` parameter to limit the time range.

---

## Restore Questions

### Can I restore to a different organization?

Yes! Use the `--target-org` and `--target-pat` parameters:

```bash
adobackup.exe restore-all \
  --target-org "https://dev.azure.com/targetorg" \
  --target-pat "target-pat-token"
```

### Can I restore to a different project?

Yes! Use the `--target-project` parameter:

```bash
adobackup.exe restore-all -p "SourceProject" --target-project "TargetProject"
```

**Note:** The target project must exist before running the restore.

### Does restore overwrite existing resources?

Yes, restore will overwrite existing resources with the same name/ID. Always use `--dry-run` to preview changes first:

```bash
adobackup.exe restore-all --dry-run -v
```

### What is dry-run mode?

Dry-run mode (`--dry-run`) previews what would be restored without making any actual changes. This is useful for:
- Validating restore before execution
- Checking for conflicts
- Estimating restore time
- Testing restore procedures

### Can I restore only specific resources?

Yes, use resource filters:

```bash
# Restore specific repositories
adobackup.exe restore-all --repositories "Repo1,Repo2"

# Restore specific build definitions
adobackup.exe restore-all --definitions "1,2,5"

# Restore specific work items
adobackup.exe restore-all --workitem-ids "100,101,102"
```

### What is bypass rules in work items restore?

`--bypass-rules` skips work item validation rules during restore. This is useful when:
- Source and target have different work item configurations
- Required fields differ between organizations
- Custom rules block restoration

**Default:** `true` (bypass enabled)

```bash
# Enable bypass rules (default)
adobackup.exe workitems-restore --bypass-rules true

# Disable bypass rules (strict validation)
adobackup.exe workitems-restore --bypass-rules false
```

### Can I restore from a specific backup date?

Yes, use the `-d` parameter with YYYYMMDD format:

```bash
adobackup.exe restore-all -d "20250128"
```

Without this parameter, the most recent backup is used.

### How long does a restore take?

Similar to backup duration, it depends on:
- Amount of data to restore
- Network speed
- Target organization responsiveness
- Number of parallel operations

**Typical timings:**
- Small project: 5-20 minutes
- Medium project: 20-60 minutes
- Large project: 1-4 hours

### Can I test a restore without affecting production?

Yes! Use one of these approaches:

1. **Dry-run mode:**
   ```bash
   adobackup.exe restore-all --dry-run -v
   ```

2. **Restore to test organization:**
   ```bash
   adobackup.exe restore-all \
     --target-org "https://dev.azure.com/testorg" \
     --target-pat "test-pat"
   ```

3. **Restore to different project:**
   ```bash
   adobackup.exe restore-all \
     -p "Production" \
     --target-project "Test-Restore"
   ```

---

## Security Questions

### How are credentials stored?

Credentials are NOT stored by the utility. You must provide them:
- As command-line parameters
- In configuration file (not recommended for PAT)
- Via environment variables
- Retrieved from secure vault (Azure Key Vault, etc.)

**Best Practice:** Use Azure Key Vault or similar for credential management.

### Are PAT tokens secure?

PAT tokens are passed directly to Azure DevOps API and not stored. However:
- Never commit PAT tokens to source control
- Use secret variables in pipelines
- Rotate tokens regularly
- Use minimum required scopes
- Set expiration dates

### Are backups encrypted?

The utility does not encrypt backups by default. To secure backups:
- Use BitLocker (Windows) or similar disk encryption
- Encrypt backup directories with VeraCrypt
- Store backups on encrypted network storage
- Use Azure Storage with encryption

See [Best Practices - Security](./best-practices.md#security) for details.

### What permissions does my PAT need?

**Backup (read-only):**
- Code: Read
- Build: Read
- Work Items: Read
- Variable Groups: Read
- Project and Team: Read

**Restore (read-write):**
- Code: Read & Write
- Build: Read & Write
- Work Items: Read & Write
- Variable Groups: Read & Write
- Project and Team: Read & Write

### Can I backup secret variables?

Yes, the utility can backup variable groups including secret variables (when PAT has appropriate permissions). Secret values are stored in plain text in the backup files, so ensure backups are secured with encryption.

---

## Troubleshooting Questions

### Why is my backup failing?

Common causes:
1. **PAT expired or invalid** - Verify PAT is valid and has required scopes
2. **Network issues** - Check connectivity to Azure DevOps
3. **Insufficient permissions** - Ensure PAT has all required scopes
4. **Disk space** - Verify sufficient disk space available
5. **Rate limiting** - Reduce parallelism or space out backups
6. **License issues** - Validate license key

Run with `-v` (verbose) for detailed error information.

### Why am I getting rate limited?

Azure DevOps has API rate limits. To resolve:
1. Reduce parallelism: `--MaxParallelism 2`
2. Space out backup schedules
3. Use incremental backups
4. Contact Azure DevOps support for limit increase

### Why is my backup so slow?

Possible reasons:
1. **Network speed** - Slow connection to Azure DevOps
2. **Large data volume** - Many repositories, builds, or work items
3. **Low parallelism** - Increase `--MaxParallelism` (carefully)
4. **Rate limiting** - API throttling by Azure DevOps
5. **Disk I/O** - Slow storage (use SSD)

Try running with `--MaxParallelism 8` and monitor for improvements.

### My restore is failing with "Project not found"

The target project must exist before restore. Create it in Azure DevOps first:
1. Go to target organization
2. Create new project with desired name
3. Run restore with `--target-project "ProjectName"`

### How do I recover from a failed backup?

1. **Check logs** - Review error messages with `-v`
2. **Verify metadata** - Check if metadata directory exists
3. **Re-run backup** - The utility is idempotent (safe to re-run)
4. **Use incremental** - If metadata exists, use `-i` to continue

Failed backups are safe to retry - they won't corrupt existing backups.

### Where are the logs stored?

Logs are stored in `{BackupRoot}/logs/` directory:
- One log file per execution
- Named with timestamp
- Retained until manually deleted

In Azure Pipelines, publish logs as artifacts for troubleshooting.

---

## Performance Questions

### How can I speed up backups?

1. **Use incremental mode:** `-i` flag
2. **Increase parallelism:** `--MaxParallelism 8` (monitor for rate limiting)
3. **Use selective backups:** Skip unnecessary resources
4. **Upgrade storage:** Use SSD instead of HDD
5. **Run on Azure VM:** Reduce network latency
6. **Backup specific projects:** `-p "Project1,Project2"`

### What is the optimal MaxParallelism setting?

It depends on your environment:
- **High-performance server:** 8-12
- **Standard server:** 4-6 (default: 4)
- **Limited resources:** 2-3

Start with default (4) and increase gradually while monitoring for rate limiting.

### Should I backup during business hours?

**Recommendation:** Schedule backups during off-peak hours (nights/weekends) to:
- Minimize impact on users
- Reduce rate limiting risk
- Faster execution (less API contention)

For incremental backups, impact is minimal and can run during business hours.

---

## Migration Questions

### Can I migrate between Azure DevOps organizations?

Yes! This is a common use case:

```bash
# Backup from source org
adobackup.exe backup-all \
  --OrganizationUrl "https://dev.azure.com/sourceorg" \
  --Pat "source-pat"

# Restore to target org
adobackup.exe restore-all \
  --target-org "https://dev.azure.com/targetorg" \
  --target-pat "target-pat"
```

### Can I clone a project?

Yes! Restore to a different project name:

```bash
adobackup.exe restore-all \
  -p "SourceProject" \
  --target-project "ClonedProject"
```

The target project must exist before cloning.

### Does restore preserve work item IDs?

Work item IDs are assigned by Azure DevOps and cannot be preserved when restoring to a different organization or project. The utility will create new work items with new IDs but preserve relationships and history.

### Can I merge backups from multiple organizations?

No, the utility does not support merging. Each backup is organization-specific. For multi-org scenarios:
- Backup each organization separately
- Restore each organization separately
- Use different backup directories

---

## Licensing Questions

### How do I activate my license?

```bash
adobackup.exe license-activate -k "your-license-key"
```

This creates a `license.lic` file in the application directory.

### How do I validate my license?

```bash
adobackup.exe license-validate -k "your-license-key"
```

### Where is the license file stored?

By default, the license file is stored in the application directory as `license.lic`. You can specify a custom location:

```bash
adobackup.exe license-activate -k "key" -f "C:\path\to\license.lic"
```

### What happens if my license expires?

The utility will stop working and display a license error message. Contact Lamdat to renew your license.

### Can I use one license for multiple servers?

License terms vary. Contact Lamdat for multi-server licensing options.

---

## Integration Questions

### Can I integrate with Azure Pipelines?

Yes! See the [Pipeline Integration Guide](./pipeline-integration.md) for detailed examples.

### Can I use with GitHub Actions?

Yes! The utility can be used with any CI/CD system that supports:
- Running command-line tools
- .NET 9 runtime
- Secure secret management

### Can I run in Docker?

Yes, create a Dockerfile:

```dockerfile
FROM mcr.microsoft.com/dotnet/runtime:9.0
COPY adobackup /app/
WORKDIR /app
ENTRYPOINT ["./adobackup"]
```

### Can I integrate with monitoring systems?

Yes! The utility:
- Returns exit codes (0 = success, 1 = failure)
- Logs to stdout/stderr
- Creates log files
- Can integrate with Splunk, ELK, etc.

---

## Support Questions

### How do I report a bug?

Contact Lamdat support at support@lamdat.com with:
- Detailed error description
- Log files (from `{BackupRoot}/logs/`)
- Command used
- Organization size and configuration

### How do I request a feature?

Email feature requests to support@lamdat.com with:
- Detailed feature description
- Use case and benefits
- Priority level

### Where can I find more documentation?

- [Getting Started Guide](./getting-started.md)
- [Command Reference](./command-reference.md)
- [Best Practices](./best-practices.md)
- [Pipeline Integration](./pipeline-integration.md)
- [Troubleshooting Guide](./troubleshooting.md)

### Is there a community forum?

Contact Lamdat for information about community resources and support channels.

---

## Still Have Questions?

If your question isn't answered here:
1. Check the [Troubleshooting Guide](./troubleshooting.md)
2. Review the [Command Reference](./command-reference.md)
3. Contact support: support@lamdat.com

---

**Last Updated:** January 2025
