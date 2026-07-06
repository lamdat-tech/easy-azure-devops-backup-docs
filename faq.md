# Frequently Asked Questions (FAQ)

Common questions and answers about the Azure DevOps Backup & Restore utility.

## General Questions

### What is Azure DevOps Backup & Restore?

Azure DevOps Backup & Restore is a solution that enables comprehensive backup and restore operations for Azure DevOps resources including Git repositories, build definitions, work items, pipeline variables, and queries. It is delivered as Azure DevOps pipeline tasks (**AzureDevOpsBackupTask** and **AzureDevOpsRestoreTask**) that integrate directly into your CI/CD pipelines.

### What Azure DevOps resources can be backed up?

The following resources can be backed up:
- **Git Repositories** - Full clone with all branches and history
- **Build Definitions** - Pipeline configurations
- **Build History** - Up to 100 builds per definition per run
- **Work Items** - Complete history, attachments, and relationships
- **Pipeline Variables** - Variable groups (including secret values)
- **Shared Queries** - Query definitions and folder structures
- **Pull Requests** - PR metadata, comments, and relationships
- **Task Groups** - Reusable task group definitions
- **Boards** - Board configurations and settings
- **Dashboards** - Dashboard widgets and layouts
- **Wikis** - Wiki pages and content
- **Release Definitions** - Release pipeline configurations
- **Service Connections** - Service endpoint configurations
- **Areas & Iterations** - Classification node hierarchies
- **Test Plans** - Test plans, suites, runs, and results

### Do I need a license?

Yes, a valid license key is required. The license is issued per **Azure DevOps organization**. A **trial license** is available — visit [easyadobackup.com](https://easyadobackup.com/) to request one.

Once you receive your license key, store it as a secret variable (`Backup.LicenseKey`) and pass it as the `ADOBACKUP_LICENSE_KEY` environment variable to each backup and restore task.

### What platforms are supported?

The pipeline tasks run on any Azure Pipelines build agent (Microsoft-hosted or self-hosted) that supports Windows. A self-hosted Windows agent is recommended for large backups due to disk space requirements.

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

Yes! Use the include flag inputs on `AzureDevOpsBackupTask`:

```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    BackupAll: false
    IncludeGit: true       # Only Git repositories
    IncludeWorkItems: true # Only work items
```

### Can I backup specific projects only?

Yes, use the `Projects` input:

```yaml
inputs:
  Projects: 'ProjectA,ProjectB'
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

Yes! Use Azure Pipelines scheduled triggers. See the [Pipeline Integration Guide](./pipeline-integration.md) for complete examples.

### Why does build backup always include history?

Build history is valuable for audit trails and compliance. The backup task always backs up build history (up to 100 builds per definition per run — Azure DevOps API limitation). You can use the `Days` input to limit the time range.

---

## Restore Questions

### Can I restore to a different organization?

Yes! Use the `TargetOrganizationUrl` and `TargetPat` inputs on `AzureDevOpsRestoreTask`:

```yaml
- task: AzureDevOpsRestoreTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)
  inputs:
    Pat: '$(Backup.AdoPat.RO)'
    BackupRoot: '$(Backup.Root)'
    TargetOrganizationUrl: 'https://dev.azure.com/targetorg'
    TargetPat: '$(Target.AdoPat.RW)'
```

**Note for Work Items:** Cross-organization work items restore uses a two-phase process — Phase 1 creates items with new IDs while building a source-to-target ID mapping file (`.metadata/id-mapping.json`), Phase 2 relinks all relations using the new target IDs. Specify exactly one source project with `Projects` when using `TargetOrganizationUrl`.

### Can I restore to a different project?

Yes! Use the `TargetProject` input:

```yaml
inputs:
  Projects: 'SourceProject'
  TargetProject: 'TargetProject'
```

**Note:** The target project must exist before running the restore.

### Does restore overwrite existing resources?

Yes, restore will overwrite existing resources with the same name/ID. Always use `DryRun: true` to preview changes first:

```yaml
inputs:
  DryRun: true
  Verbose: true
```

### What is dry-run mode?

`DryRun: true` previews what would be restored without making any actual changes. This is useful for:
- Validating restore before execution
- Checking for conflicts
- Estimating restore time
- Testing restore procedures

### Can I restore only specific resources?

Yes, use the include flag inputs:

```yaml
inputs:
  RestoreAll: false
  IncludeGit: true
  GitRepositories: 'Repo1,Repo2'    # specific repos
  IncludeBuilds: true
  BuildDefinitions: '1,2,5'         # specific definitions
  IncludeWorkItems: true
  WorkItemIds: '100,101,102'        # specific work items
```

### What is bypass rules in work items restore?

`BypassRules: true` skips work item validation rules during restore. This is useful when:
- Source and target have different work item configurations
- Required fields differ between organizations
- Custom rules block restoration

**Default:** `true` (bypass enabled)

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

1. **Dry-run mode** — set `DryRun: true` in the task inputs (no changes are made)

2. **Restore to test organization** — use `TargetOrganizationUrl` and `TargetPat` inputs

3. **Restore to different project** — use `TargetProject` input with a test project name

---

## Security Questions

### How are credentials stored?

Credentials are NOT stored by the task. Provide them as secret variables in a pipeline variable group and reference them in the task inputs. For enhanced security, link your variable group to Azure Key Vault so secrets are retrieved at runtime.

**Best Practice:** Store all secrets in Azure Key Vault and link them to your variable group.

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

Yes! This is a common use case. Use two pipelines:

1. **Backup pipeline** — runs against the source organization
2. **Restore pipeline** — uses `TargetOrganizationUrl` and `TargetPat` inputs pointing to the target organization

See [Pipeline Integration Guide](./pipeline-integration.md#example-2-cross-organization-restore) for a complete example.

### Can I clone a project?

Yes! Set `TargetProject` to a different project name in the restore task:

```yaml
inputs:
  Projects: 'SourceProject'
  TargetProject: 'ClonedProject'
  RestoreAll: true
```

The target project must exist before running the restore.

### Does restore preserve work item IDs?

When restoring to the same organization and project, work item IDs are preserved. When restoring cross-organization or cross-project, work item IDs cannot be preserved. The restore task uses a two-phase approach:

- **Phase 1:** Creates all work items in the target with new IDs assigned by Azure DevOps, and builds a source-to-target ID mapping file (`.metadata/id-mapping.json`).
- **Phase 2:** Relinks all relations, parent-child links, and references using the new target IDs from the mapping.

This ensures all relationships are preserved even though IDs change. The mapping file is persisted so re-runs are idempotent — already-mapped items are detected and duplicate relations are skipped.

### Can I merge backups from multiple organizations?

No, the backup task does not support merging. Each backup is organization-specific. For multi-org scenarios:
- Backup each organization separately
- Restore each organization separately
- Use different backup directories

---

## Licensing Questions

### How do I set up my license?

1. Store your license key as a **secret variable** named `Backup.LicenseKey` in your variable group **"ADO Backup Restore"**
2. Pass it as `ADOBACKUP_LICENSE_KEY` environment variable to **both** the backup and restore tasks:

```yaml
- task: AzureDevOpsBackupTask@0
  env:
    ADOBACKUP_LICENSE_KEY: $(Backup.LicenseKey)  # Required
  inputs:
    # ...
```

### What happens if my license expires or is invalid?

The task will fail with a license error. Visit [easyadobackup.com](https://easyadobackup.com/) to request a trial or renew your production license.


---

## Integration Questions

### Can I integrate with Azure Pipelines?

Yes! See the [Pipeline Integration Guide](./pipeline-integration.md) for detailed examples.

### Can I integrate with monitoring systems?

## Test Plans Questions

### What test plan data is backed up?

The test plans backup includes:
- **Test Plans** - Top-level containers with name, description, state, iteration, and area path
- **Test Suites** - Hierarchical structure (static, requirement-based, query-based)
- **Test Cases** - Test case associations within suites
- **Test Runs** - Test run history (configurable date range, default: last 90 days)
- **Test Results** - Detailed outcomes, durations, and result metadata

### How much test run history is backed up?

By default, the last **90 days** of test run history is backed up. You can customize this:

Use the `TestRunsDays` or `TestRunsAll` inputs on `AzureDevOpsBackupTask`. By default, the last 90 days are backed up.

**Note:** Backing up all test runs may take a long time for organizations with extensive test history.

### What happens to requirement-based and query-based suites during restore?

The restore process handles suite type conversions automatically:

- **Requirement-Based Suites**: If the requirement work item is missing in the target project, the suite is converted to a **static suite** (with a logged warning)
- **Query-Based Suites**: If the query is missing in the target project, the suite is converted to a **static suite** (with a logged warning)
- **Static Suites**: Restored as-is with full test case associations

This ensures test plans are restorable even when dependencies are missing in the target.

### What if test case work items don't exist in the target project?

During restore, test case work items are validated before association:

- **Existing test cases**: Associated successfully with the test suite
- **Missing test cases**: Skipped with a warning logged
- **Summary report**: Shows `successCount/skippedCount` for each suite

**Example:**
```
✅ Test Suite "Regression Tests": 45/50 test cases restored (5 skipped - test cases 1234, 1235, 1236, 1237, 1238 not found)
```

**Recommendation:** Restore work items before restoring test plans to ensure all test case associations succeed.

### Can I restore test plans without test runs?

Yes! By default, only **test plans and suites** are restored. Test runs are optional:

Use `IncludeTestPlans: true` on the restore task. Test runs can be included via `TestRunsInclude: true`.

### Can I restore test plans to a different project?

Yes! Use the `TargetProject` input on `AzureDevOpsRestoreTask`:

```yaml
inputs:
  Projects: 'SourceProject'
  TargetProject: 'TargetProject'
  IncludeTestPlans: true
```

Test plan and suite IDs are automatically mapped from source to target and saved to `.metadata/test-plan-id-mapping.json` and `test-suite-id-mapping.json`.

### How is the test suite hierarchy preserved during backup and restore?

Test suites are backed up and restored hierarchically using a **depth-first recursive** approach:

1. **Backup**: Root suite is backed up first, then child suites recursively
2. **Restore**: Root suite is restored first, then child suites in the same order
3. **ID Mapping**: Suite IDs are tracked from source → target for test run restoration

This ensures the complete suite tree structure is preserved exactly as it was in the source.

---


Yes! The pipeline tasks:
- Return pipeline success/failure status
- Create log files in `{BackupRoot}/logs/`
- Can be published as artifacts for downstream processing
- Log output can be forwarded to Splunk, ELK, or other monitoring systems via pipeline scripts

---

## Support Questions

### How do I report a bug?

Visit [easyadobackup.com](https://easyadobackup.com/) with:
- Detailed error description
- Log files (from `{BackupRoot}/logs/`)
- Task inputs used (with sensitive data redacted)
- Organization size and Azure DevOps tier

### How do I request a feature?

Visit [easyadobackup.com](https://easyadobackup.com/) with:
- Detailed feature description
- Use case and benefits
- Priority level

### Where can I find more documentation?

- [Getting Started Guide](./getting-started.md)
- [Pipeline Integration Guide](./pipeline-integration.md)
- [Best Practices](./best-practices.md)
- [Troubleshooting Guide](./troubleshooting.md)

### Is there a community forum?

Contact Lamdat for information about community resources and support channels.

---

## Still Have Questions?

If your question isn't answered here:
1. Check the [Troubleshooting Guide](./troubleshooting.md)
2. Review the [Pipeline Integration Guide](./pipeline-integration.md)
3. Visit [easyadobackup.com](https://easyadobackup.com/) for support

---

**Last Updated:** January 2025
