# Command Reference

Complete reference documentation for all commands and options available in the Azure DevOps Backup & Restore utility.

## Table of Contents

- [Global Options](#global-options)
- [Backup Commands](#backup-commands)
  - [backup-all](#backup-all)
  - [git-backup](#git-backup)
  - [build-backup](#build-backup)
  - [workitems-backup](#workitems-backup)
  - [variables-backup](#variables-backup)
  - [queries-backup](#queries-backup)
- [Restore Commands](#restore-commands)
  - [restore-all](#restore-all)
  - [git-restore](#git-restore)
  - [build-restore](#build-restore)
  - [workitems-restore](#workitems-restore)
  - [variables-restore](#variables-restore)
  - [queries-restore](#queries-restore)
- [License Commands](#license-commands)
  - [license-validate](#license-validate)
  - [license-activate](#license-activate)

---

## Global Options

These options can be used with any command:

| Option | Short | Type | Required | Description |
|--------|-------|------|----------|-------------|
| `--OrganizationUrl` | | string | Yes* | Azure DevOps organization URL (e.g., https://dev.azure.com/yourorg) |
| `--Pat` | | string | Yes* | Personal Access Token for authentication |
| `--BackupRoot` | | string | Yes* | Root directory for backups |
| `--MaxParallelism` | | int | No | Maximum parallel operations (default: 4) |
| `--verbose` | `-v` | flag | No | Enable verbose output |

\* Can be configured in appsettings.json instead of command-line

---

## Backup Commands

### backup-all

Backup all resource types from Azure DevOps organization.

**Syntax:**
```bash
adobackup.exe backup-all [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names to backup |
| `--incremental` | `-i` | flag | false | Incremental backup mode (only new/changed data) |
| `--all` | `-a` | flag | true | Backup all components (default behavior) |
| `--include-git` | | flag | false | Include Git repositories in backup |
| `--include-builds` | | flag | false | Include build definitions and history in backup |
| `--include-workitems` | | flag | false | Include work items in backup |
| `--include-variables` | | flag | false | Include pipeline variables in backup |
| `--include-queries` | | flag | false | Include queries in backup |
| `--warnings-as-errors` | | flag | false | Treat warnings (partial failures) as errors - task fails on partial failures |
| `--ignore-warnings` | | flag | false | Ignore warnings (partial failures) - task succeeds even with partial failures |
| `--repositories` | `-r` | string | all | Comma-separated list of repository names (for Git) |
| `--definitions` | `-d` | string | all | Comma-separated list of build definition names |
| `--max-builds` | | int | 100 | Maximum builds per definition (max: 100) |
| `--days` | | int | all | Backup builds from last N days |
| `--min-id` | | int | 1 | Minimum work item ID |
| `--max-id` | | int | all | Maximum work item ID |
| `--queries-folder-path` | `-f` | string | all | Specific query folder path |

**Component Selection Behavior:**
- By default, `--all` is `true`, backing up all components
- When any `--include-*` flag is used, only specified components are backed up
- Use `--include-*` flags for selective backups

**Examples:**

**Basic full backup (all components):**
```bash
adobackup.exe backup-all
```

**Explicitly backup all components:**
```bash
adobackup.exe backup-all --all -v
```

**Incremental backup:**
```bash
adobackup.exe backup-all -i -v
```

**Backup specific projects:**
```bash
adobackup.exe backup-all -p "ProjectA,ProjectB" -v
```

**Backup only Git repositories:**
```bash
adobackup.exe backup-all --include-git -v
```

**Backup Git and Builds only:**
```bash
adobackup.exe backup-all --include-git --include-builds -v
```

**Backup Work Items and Queries only:**
```bash
adobackup.exe backup-all --include-workitems --include-queries -v
```

**Backup with custom settings:**
```bash
adobackup.exe backup-all ^
  -p "MyProject" ^
  -i ^
  --include-git ^
  --include-builds ^
  --repositories "Repo1,Repo2" ^
  --max-builds 50 ^
  --days 30 ^
  -v
```

**Incremental backup (continuous backup strategy):**
```bash
adobackup.exe backup-all -i --MaxParallelism 8 -v
```

**Treat warnings as errors (fail on partial failures):**
```bash
adobackup.exe backup-all --warnings-as-errors -v
```

**Ignore warnings (succeed even with partial failures):**
```bash
adobackup.exe backup-all --ignore-warnings -v
```

**Incremental backup with warnings as errors:**
```bash
adobackup.exe backup-all -i --warnings-as-errors -v
```

---

### git-backup

Backup Git repositories only.

**Syntax:**
```bash
adobackup.exe git-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--repositories` | `-r` | string | all | Comma-separated list of repository names |

**Examples:**

**Backup all repositories:**
```bash
adobackup.exe git-backup -v
```

**Backup specific repositories from specific projects:**
```bash
adobackup.exe git-backup -p "ProjectA" -r "Repo1,Repo2" -v
```

**Backup with increased parallelism:**
```bash
adobackup.exe git-backup --MaxParallelism 8 -v
```

---

### build-backup

Backup build definitions and build history.

**Syntax:**
```bash
adobackup.exe build-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--definitions` | `-d` | string | all | Comma-separated list of build definition names |
| `--incremental` | `-i` | flag | false | Incremental mode (builds since last backup) |
| `--max-builds` | | int | 100 | Maximum builds per definition (limit: 100) |
| `--days` | | int | all | Backup builds from last N days |

**Important Notes:**
- Build history is **always** backed up (there is no option to disable it)
- Maximum 100 builds per definition per run (Azure DevOps API limitation)
- Metadata is **always** saved to support incremental backups
- Use `--incremental` or `--days` to capture more builds over time

**Examples:**

**Backup all builds:**
```bash
adobackup.exe build-backup -v
```

**Incremental backup (automatic since last run):**
```bash
adobackup.exe build-backup -i -v
```

**Backup specific build definitions:**
```bash
adobackup.exe build-backup -p "MyProject" -d "CI-Build,Release-Build" -v
```

**Backup builds from last 30 days:**
```bash
adobackup.exe build-backup --days 30 -v
```

**Backup with custom max builds:**
```bash
adobackup.exe build-backup --max-builds 50 -v
```

**Incremental with time filter:**
```bash
adobackup.exe build-backup -i --days 30 -v
```

---

### workitems-backup

Backup work items with full history and attachments.

**Syntax:**
```bash
adobackup.exe workitems-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--incremental` | `-i` | flag | false | Incremental mode (new work items only) |
| `--min-id` | | int | 1 | Minimum work item ID |
| `--max-id` | | int | all | Maximum work item ID |

**Examples:**

**Backup all work items:**
```bash
adobackup.exe workitems-backup -v
```

**Incremental backup (automatic):**
```bash
adobackup.exe workitems-backup -i -v
```

**Backup specific project:**
```bash
adobackup.exe workitems-backup -p "ProjectA" -v
```

**Manual incremental (new items only):**
```bash
adobackup.exe workitems-backup --min-id 50000 -v
```

**Backup specific ID range:**
```bash
adobackup.exe workitems-backup --min-id 10000 --max-id 20000 -v
```

---

### variables-backup

Backup pipeline variables and variable groups.

**Syntax:**
```bash
adobackup.exe variables-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |

**Examples:**

**Backup all variables:**
```bash
adobackup.exe variables-backup -v
```

**Backup specific projects:**
```bash
adobackup.exe variables-backup -p "ProjectA,ProjectB" -v
```

---

### queries-backup

Backup shared queries and query folders.

**Syntax:**
```bash
adobackup.exe queries-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--queries-folder-path` | `-f` | string | all | Specific query folder path |

**Examples:**

**Backup all queries:**
```bash
adobackup.exe queries-backup -v
```

**Backup specific project queries:**
```bash
adobackup.exe queries-backup -p "ProjectA" -v
```

**Backup specific folder:**
```bash
adobackup.exe queries-backup -f "Shared Queries/Team A" -v
```

**Backup single query:**
```bash
adobackup.exe queries-backup -p "ProjectA" -f "Epics Queries/All Active Epics" -v
```

---

## Restore Commands

### restore-all

Restore all resource types from backup.

**Syntax:**
```bash
adobackup.exe restore-all [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names from backup |
| `--backup-date` | `-d` | string | latest | Backup date to restore from (YYYYMMDD format) |
| `--target-org` | | string | source | Target organization URL (for migration) |
| `--target-pat` | | string | source | Target organization PAT token |
| `--target-project` | `-t` | string | source | Target project name (for cloning/migration) |
| `--dry-run` | | flag | false | Preview changes without applying them |
| `--all` | `-a` | flag | true | Restore all components (default behavior) |
| `--include-git` | | flag | false | Include Git repositories in restore |
| `--include-builds` | | flag | false | Include build definitions in restore |
| `--include-workitems` | | flag | false | Include work items in restore |
| `--include-variables` | | flag | false | Include pipeline variables in restore |
| `--include-queries` | | flag | false | Include queries in restore |
| `--warnings-as-errors` | | flag | false | Treat warnings (partial failures) as errors - task fails on partial failures |
| `--ignore-warnings` | | flag | false | Ignore warnings (partial failures) - task succeeds even with partial failures |
| `--repositories` | `-r` | string | all | Comma-separated list of repository names |
| `--definitions` | | string | all | Comma-separated list of build definition IDs |
| `--workitem-ids` | `-i` | string | all | Comma-separated list of work item IDs |
| `--bypass-rules` | | bool | true | Bypass work item validation rules |
| `--queries-folder-path` | `-f` | string | all | Specific query folder path |

**Component Selection Behavior:**
- By default, `--all` is `true`, restoring all components
- When any `--include-*` flag is used, only specified components are restored
- Use `--include-*` flags for selective restores

**Examples:**

**Basic restore (to same organization/project):**
```bash
adobackup.exe restore-all -v
```

**Explicitly restore all components:**
```bash
adobackup.exe restore-all --all -v
```

**Restore from specific backup date:**
```bash
adobackup.exe restore-all -d "20250128" -v
```

**Dry run (preview only):**
```bash
adobackup.exe restore-all --dry-run -v
```

**Restore to different organization:**
```bash
adobackup.exe restore-all ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

**Clone project (restore to different project name):**
```bash
adobackup.exe restore-all ^
  -p "SourceProject" ^
  --target-project "ClonedProject" ^
  -v
```

**Restore only Git repositories:**
```bash
adobackup.exe restore-all --include-git -v
```

**Restore Git and Builds only:**
```bash
adobackup.exe restore-all --include-git --include-builds -v
```

**Restore Work Items and Queries only:**
```bash
adobackup.exe restore-all --include-workitems --include-queries -v
```

**Restore specific resources:**
```bash
adobackup.exe restore-all ^
  --include-git ^
  --include-workitems ^
  --repositories "Repo1,Repo2" ^
  -i "1001,1002,1003" ^
  -v
```

**Restore with warnings as errors:**
```bash
adobackup.exe restore-all --warnings-as-errors -v
```

**Restore with warnings ignored:**
```bash
adobackup.exe restore-all --ignore-warnings -v
```

---

### git-restore

Restore Git repositories only.

**Syntax:**
```bash
adobackup.exe git-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--repositories` | `-r` | string | all | Comma-separated list of repository names |
| `--target-org` | | string | source | Target organization URL |
| `--target-pat` | | string | source | Target PAT token |
| `--target-project` | `-t` | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Examples:**

**Restore all repositories:**
```bash
adobackup.exe git-restore -v
```

**Restore specific repositories:**
```bash
adobackup.exe git-restore -p "ProjectA" -r "Repo1,Repo2" -v
```

**Restore to different organization:**
```bash
adobackup.exe git-restore ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat" ^
  -v
```

---

### build-restore

Restore build definitions only.

**Syntax:**
```bash
adobackup.exe build-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--definitions` | | string | all | Comma-separated list of build definition IDs |
| `--target-org` | | string | source | Target organization URL |
| `--target-pat` | | string | source | Target PAT token |
| `--target-project` | `-t` | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Examples:**

**Restore all build definitions:**
```bash
adobackup.exe build-restore -v
```

**Restore specific definitions:**
```bash
adobackup.exe build-restore --definitions "1,2,5" -v
```

**Restore to different project:**
```bash
adobackup.exe build-restore ^
  -p "SourceProject" ^
  --target-project "TargetProject" ^
  -v
```

---

### workitems-restore

Restore work items only.

**Syntax:**
```bash
adobackup.exe workitems-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--workitem-ids` | `-i` | string | all | Comma-separated list of work item IDs |
| `--bypass-rules` | | bool | true | Bypass work item validation rules |
| `--target-org` | | string | source | Target organization URL |
| `--target-pat` | | string | source | Target PAT token |
| `--target-project` | `-t` | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Examples:**

**Restore all work items:**
```bash
adobackup.exe workitems-restore -v
```

**Restore specific work items:**
```bash
adobackup.exe workitems-restore -i "1,2,3,100" -v
```

**Restore with bypass rules disabled:**
```bash
adobackup.exe workitems-restore --bypass-rules false -v
```

**Restore from specific date:**
```bash
adobackup.exe workitems-restore -d "20250128" -v
```

---

### variables-restore

Restore pipeline variables only.

**Syntax:**
```bash
adobackup.exe variables-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--target-org` | | string | source | Target organization URL |
| `--target-pat` | | string | source | Target PAT token |
| `--target-project` | `-t` | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Examples:**

**Restore all variables:**
```bash
adobackup.exe variables-restore -v
```

**Restore to different project:**
```bash
adobackup.exe variables-restore ^
  -p "SourceProject" ^
  --target-project "TargetProject" ^
  -v
```

---

### queries-restore

Restore queries only.

**Syntax:**
```bash
adobackup.exe queries-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--queries-folder-path` | `-f` | string | all | Specific query folder path |
| `--target-project` | `-t` | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Examples:**

**Restore all queries:**
```bash
adobackup.exe queries-restore -v
```

**Restore specific folder:**
```bash
adobackup.exe queries-restore -f "Shared Queries/Team A" -v
```

**Restore to different project:**
```bash
adobackup.exe queries-restore ^
  -p "SourceProject" ^
  -t "TargetProject" ^
  -v
```

**Restore single query:**
```bash
adobackup.exe queries-restore ^
  -p "ProjectA" ^
  -f "Epics Queries/All Active Epics" ^
  -v
```

---

## License Commands

### license-validate

Validate a license key.

**Syntax:**
```bash
adobackup.exe license-validate -k <license-key> [options]
```

**Options:**

| Option | Short | Type | Required | Description |
|--------|-------|------|----------|-------------|
| `--key` | `-k` | string | Yes | License key to validate |
| `--tenant` | `-t` | string | No | Azure DevOps tenant URL to validate against |

**Examples:**

**Validate license key:**
```bash
adobackup.exe license-validate -k "your-license-key"
```

**Validate against specific tenant:**
```bash
adobackup.exe license-validate -k "your-license-key" -t "https://dev.azure.com/yourtenant"
```

---

### license-activate

Activate the utility with a license key.

**Syntax:**
```bash
adobackup.exe license-activate -k <license-key> [options]
```

**Options:**

| Option | Short | Type | Required | Description |
|--------|-------|------|----------|-------------|
| `--key` | `-k` | string | Yes | License key to activate |
| `--file` | `-f` | string | No | Path to save license file (default: ./license.lic) |

**Examples:**

**Activate with license key:**
```bash
adobackup.exe license-activate -k "your-license-key"
```

**Activate and save to specific location:**
```bash
adobackup.exe license-activate -k "your-license-key" -f "C:\path\to\license.lic"
```

---

## Common Patterns

### Pattern 1: Full Backup + Incremental Updates

**Initial full backup:**
```bash
adobackup.exe backup-all -v
```

**Daily incremental backups:**
```bash
adobackup.exe backup-all -i -v
```

### Pattern 2: Disaster Recovery Test

**Backup production:**
```bash
adobackup.exe backup-all --BackupRoot "C:\ProdBackup" -v
```

**Restore to test organization:**
```bash
adobackup.exe restore-all ^
  --BackupRoot "C:\ProdBackup" ^
  --target-org "https://dev.azure.com/testorg" ^
  --target-pat "test-pat" ^
  --dry-run ^
  -v
```

### Pattern 3: Project Migration

**Backup source:**
```bash
adobackup.exe backup-all -p "OldProject" -v
```

**Restore to target:**
```bash
adobackup.exe restore-all ^
  -p "OldProject" ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat" ^
  --target-project "NewProject" ^
  -v
```

### Pattern 4: Selective Component Backup/Restore

**Backup only Git and Builds:**
```bash
adobackup.exe backup-all --include-git --include-builds -v
```

**Restore only Work Items and Queries:**
```bash
adobackup.exe restore-all --include-workitems --include-queries -v
```

**Backup specific components with custom settings:**
```bash
adobackup.exe backup-all ^
  --include-builds ^
  --include-workitems ^
  --max-builds 50 ^
  --days 30 ^
  --min-id 10000 ^
  -v
```

---

## Exit Codes

The utility uses different exit codes to indicate the result of operations. The `--warnings-as-errors` and `--ignore-warnings` flags affect how exit codes are interpreted:

| Code | Description | With `--warnings-as-errors` | With `--ignore-warnings` |
|------|-------------|----------------------------|-------------------------|
| 0 | Success - All operations completed without issues | Same | Same |
| 1 | Critical failure - Unable to execute command | Same | Same |
| 2 | Partial failure - Some resources failed | Treated as failure (exit 2) | Treated as success (exit 0) |
| 3 | Fatal error - Unexpected error occurred | Same | Same |
| 255 | Invalid arguments or configuration | Same | Same |

**Warning Behavior Details:**

- **Default behavior:** Exit code 2 indicates partial failures (some items succeeded, some failed). In Azure Pipelines, this is shown as "Succeeded with issues".
  
- **With `--warnings-as-errors`:** Exit code 2 is treated as a failure, causing the task/pipeline to fail. Use this for strict quality control where any failure is unacceptable.
  
- **With `--ignore-warnings`:** Exit code 2 is converted to exit code 0 (success). Use this when partial failures are acceptable for your workflow.

**Example scenarios:**
- Backing up 100 work items where 2 fail: Exit code 2 (partial failure)
- With `--warnings-as-errors`: Pipeline fails
- With `--ignore-warnings`: Pipeline succeeds
- Default: Pipeline shows "Succeeded with issues"

---

**See Also:**
- [Getting Started Guide](./getting-started.md)
- [Best Practices](./best-practices.md)
- [Pipeline Integration](./pipeline-integration.md)
