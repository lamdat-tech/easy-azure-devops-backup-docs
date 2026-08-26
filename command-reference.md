# Command Reference

Complete reference documentation for all commands and options available in the Azure DevOps Backup & Restore utility.

## Table of Contents

- [Global Options](#global-options)
- [Backup Commands](#backup-commands)
  - [backup-all](#backup-all)
  - [git-backup](#git-backup)
  - [build-backup](#build-backup)
  - [release-backup](#release-backup)
  - [workitems-backup](#workitems-backup)
  - [variables-backup](#variables-backup)
  - [queries-backup](#queries-backup)
  - [pullrequests-backup](#pullrequests-backup)
  - [serviceconnections-backup](#serviceconnections-backup)
  - [taskgroup-backup](#taskgroup-backup)
  - [boards-backup](#boards-backup)
  - [dashboards-backup](#dashboards-backup)
  - [wiki-backup](#wiki-backup)
  - [areas-backup](#areas-backup)
  - [environments-backup](#environments-backup)
  - [testplans-backup](#testplans-backup)
- [Restore Commands](#restore-commands)
  - [restore-all](#restore-all)
  - [git-restore](#git-restore)
  - [build-restore](#build-restore)
  - [release-restore](#release-restore)
  - [workitems-restore](#workitems-restore)
  - [variables-restore](#variables-restore)
  - [queries-restore](#queries-restore)
  - [pullrequests-restore](#pullrequests-restore)
  - [serviceconnections-restore](#serviceconnections-restore)
  - [taskgroup-restore](#taskgroup-restore)
  - [boards-restore](#boards-restore)
  - [dashboards-restore](#dashboards-restore)
  - [wiki-restore](#wiki-restore)
  - [areas-restore](#areas-restore)
  - [environments-restore](#environments-restore)
  - [testplans-restore](#testplans-restore)
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
| `--MaxParallelism` | | int | No | Maximum parallel operations (default: 4). Controls how many resources are processed concurrently
| `--verbose` | `-v` | flag | No | Enable verbose output |

\* Can be configured in appsettings.json instead of command-line

**MaxParallelism Configuration Priority:**
1. **Command-line argument** `--MaxParallelism <value>` 

**Recommended MaxParallelism Values:**

| Environment | Hardware | Recommended | Reason |
|-------------|----------|-------------|--------|
| **Server** | 16+ cores, 1 Gbps | **6-8** | High performance |
| **Azure Pipeline** | Hosted agents | **4-6** | Good balance for cloud agents |
| **Debug** | Any | **1** | Sequential, easier debugging |

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
| `--include-releases` | | flag | false | Include release definitions in backup |
| `--include-workitems` | | flag | false | Include work items in backup |
| `--include-variables` | | flag | false | Include pipeline variables in backup |
| `--include-queries` | | flag | false | Include queries in backup |
| `--include-serviceconnections` | | flag | false | Include service connections in backup |
| `--include-pullrequests` | | flag | false | Include pull requests in backup |
| `--include-taskgroups` | | flag | false | Include task groups in backup |
| `--include-boards` | | flag | false | Include boards in backup |
| `--include-dashboards` | | flag | false | Include dashboards in backup |
| `--include-wikis` | | flag | false | Include wikis in backup |
| `--include-areas` | | flag | false | Include areas and iterations in backup |
| `--include-testplans` | | flag | false | Include test plans, suites, and runs in backup |
| `--test-runs-days` | | int | 90 | Number of days of test run history to backup (default: 90 days, 0 = all) |
| `--include-all-test-runs` | | flag | false | Backup all test runs regardless of date (overrides --test-runs-days) |
| `--include-environments` | | flag | false | Include pipeline environments in backup |
| `--warnings-as-errors` | | flag | false | Treat warnings (partial failures) as errors - task fails on partial failures |
| `--ignore-warnings` | | flag | false | Ignore warnings (partial failures) - task succeeds even with partial failures |
| `--repositories` | | string | all | Comma-separated list of repository names (for Git and Pull Requests) |
| `--definitions` | | string | all | Comma-separated list of build definition names |
| `--max-builds` | | int | 100 | Maximum builds per definition (max: 100) |
| `--days` | | int | all | Backup builds from last N days |
| `--min-id` | | int | none | Minimum work item ID |
| `--max-id` | | int | all | Maximum work item ID |
| `--queries-folder-path` | | string | all | Specific query folder path |
| `--pr-status` | | string | All | Pull request status filter: All, Active, Completed, or Abandoned |
| `--active-pr-refresh-hours` | | int | 24 | Hours interval to refresh active PRs in incremental mode (to capture comments, work items, reviewers) |
| `--service-connections` | | string | all | Comma-separated list of service connection names |

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

> **Note:** Branch policies (all policy configurations per repository) and the repository default branch are automatically backed up to `branch-policies.json` alongside each git clone. No additional flags are needed.

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
|--------|-------|------|---------|-------------|
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

### release-backup

Backup release definitions (release pipelines).

**Syntax:**
```bash
adobackup.exe release-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | **required** | Comma-separated list of project names |
| `--release-name` | `-n` | string | none | Specific release definition name to backup (optional, backs up all if not specified) |

**Important Notes:**
- **Secret variables cannot be retrieved** via Azure DevOps API (platform limitation)
  - Backed up as `null` with `isSecret: true` flag
  - Must be manually reconfigured after restore
- Release definitions use the **vsrm subdomain** (`https://vsrm.dev.azure.com/`)
- Includes all environments, artifacts, variables, approvals, and deployment conditions
- Does not include release execution history (only definitions)

**Examples:**

**Backup all release definitions:**
```bash
adobackup.exe release-backup -v
```

**Backup specific project:**
```bash
adobackup.exe release-backup -p "MyProject" -v
```

**Backup specific release definitions:**
```bash
adobackup.exe release-backup -p "MyProject" --release-name "Release-Prod" -v
```

**Backup all releases from multiple projects:**
```bash
adobackup.exe release-backup -p "Project1,Project2" -v
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
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--incremental` | `-i` | flag | false | Incremental mode (new work items only) |
| `--min-id` | | int | none | Minimum work item ID |
| `--max-id` | | int | all | Maximum work item ID |
| `--max-revisions` | | int | all | Maximum number of most-recent revisions to store per work item |

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

**Backup with revision limit (keep 50 most-recent revisions per item):**
```bash
adobackup.exe workitems-backup --max-revisions 50 -v
```

**Incremental backup with revision limit:**
```bash
adobackup.exe workitems-backup -i --max-revisions 100 -v
```

**Backup specific ID range with revision limit:**
```bash
adobackup.exe workitems-backup --min-id 10000 --max-id 20000 --max-revisions 25 -v
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
|--------|-------|------|---------|-------------|
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

### pullrequests-backup

Backup pull requests with metadata, comments, reviews, and status.

**Syntax:**
```bash
adobackup.exe pullrequests-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--repositories` | `-r` | string | all | Comma-separated list of repository names |
| `--status` | `-s` | string | All | PR status filter: All, Active, Completed, or Abandoned |
| `--incremental` | `-i` | flag | false | Incremental mode (only new/modified PRs) |

**Examples:**

**Backup all pull requests:**
```bash
adobackup.exe pullrequests-backup -v
```

**Incremental backup (only new/modified PRs):**
```bash
adobackup.exe pullrequests-backup -i -v
```

**Backup active PRs only:**
```bash
adobackup.exe pullrequests-backup --status Active -v
```

**Backup specific repositories:**
```bash
adobackup.exe pullrequests-backup -p "ProjectA" -r "Repo1,Repo2" -v
```

**Backup completed PRs from specific project:**
```bash
adobackup.exe pullrequests-backup -p "ProjectA" --status Completed -v
```

---

### taskgroup-backup

Backup task groups from Azure DevOps.

**Syntax:**
```bash
adobackup.exe taskgroup-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--groups` | `-g` | string | all | Comma-separated list of task group names |

**Examples:**

**Backup all task groups:**
```bash
adobackup.exe taskgroup-backup -v
```

**Backup specific projects:**
```bash
adobackup.exe taskgroup-backup -p "ProjectA,ProjectB" -v
```

---

### boards-backup

Backup boards configuration from Azure DevOps.

**Syntax:**
```bash
adobackup.exe boards-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--team` | `-t` | string | all | Team name |
| `--board` | `-b` | string | all | Board name |

**Examples:**

**Backup all boards:**
```bash
adobackup.exe boards-backup -v
```

**Backup all boards for a specific team:**
```bash
adobackup.exe boards-backup --team "TeamA" -v
```

**Backup specific board:**
```bash
adobackup.exe boards-backup --team "TeamA" --board "Sprint Board" -v
```

---

### dashboards-backup

Backup dashboards from Azure DevOps.

**Syntax:**
```bash
adobackup.exe dashboards-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--team` | `-t` | string | all | Team name |
| `--dashboard` | `-d` | string | all | Dashboard name |

**Examples:**

**Backup all dashboards:**
```bash
adobackup.exe dashboards-backup -v
```

**Backup all dashboards for a specific team:**
```bash
adobackup.exe dashboards-backup --team "TeamA" -v
```

**Backup specific dashboard:**
```bash
adobackup.exe dashboards-backup --team "TeamA" --dashboard "Overview" -v
```

---

### wiki-backup

Backup wikis from Azure DevOps.

**Syntax:**
```bash
adobackup.exe wiki-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | **required** | Comma-separated list of project names to backup |
| `--wiki` | `-w` | string | all | Specific wiki name to backup (optional, backs up all wikis if not specified) |

**Examples:**

**Backup all wikis from a project:**
```bash
adobackup.exe wiki-backup -p "MyProject" -v
```

**Backup specific wiki:**
```bash
adobackup.exe wiki-backup -p "MyProject" -w "MyProject.wiki" -v
```

**Backup wikis from multiple projects:**
```bash
adobackup.exe wiki-backup -p "ProjectA,ProjectB" -v
```

---

### serviceconnections-backup

Backup service connections (connection metadata only, credentials excluded).

**Syntax:**
```bash
adobackup.exe serviceconnections-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |

**Examples:**

**Backup all service connections:**
```bash
adobackup.exe serviceconnections-backup -v
```

**Backup specific projects:**
```bash
adobackup.exe serviceconnections-backup -p "ProjectA,ProjectB" -v
```

---

### areas-backup

Backup areas and iterations (classification nodes) from Azure DevOps.

**Syntax:**
```bash
adobackup.exe areas-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--areas-only` | | flag | false | Backup area paths only (skip iterations) |
| `--iterations-only` | | flag | false | Backup iteration paths only (skip areas) |

**Examples:**

**Backup all areas and iterations:**
```bash
adobackup.exe areas-backup -v
```

**Backup specific projects:**
```bash
adobackup.exe areas-backup -p "ProjectA,ProjectB" -v
```

**Backup single project:**
```bash
adobackup.exe areas-backup -p "MyProject" -v
```

---

### environments-backup

Backup pipeline environments (definitions, resources, approvals, and checks) from Azure DevOps.

**Syntax:**
```bash
adobackup.exe environments-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names |
| `--environment-names` | `-e` | string | all | Comma-separated list of environment names to backup |

**Important Notes:**
- Backs up environment definitions, resource registrations, approval configurations, and check configurations
- Resource secrets (e.g., service account credentials) are **not** backed up
- Supports cross-project and cross-organization restore

**Examples:**

**Backup all environments:**
```bash
adobackup.exe environments-backup -v
```

**Backup environments from specific projects:**
```bash
adobackup.exe environments-backup -p "ProjectA,ProjectB" -v
```

**Backup specific environments by name:**
```bash
adobackup.exe environments-backup -e "Production,Staging" -v
```

**Backup specific environments from specific project:**
```bash
adobackup.exe environments-backup -p "ProjectA" -e "Production,Staging,Development" -v
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
| `--target-project` | | string | source | Target project name (for cloning/migration) |
| `--dry-run` | | flag | false | Preview changes without applying them |
| `--all` | `-a` | flag | true | Restore all components (default behavior) |
| `--include-git` | | flag | false | Include Git repositories in restore |
| `--include-builds` | | flag | false | Include build definitions in restore |
| `--include-releases` | | flag | false | Include release definitions in restore |
| `--include-workitems` | | flag | false | Include work items in restore |
| `--include-variables` | | flag | false | Include pipeline variables in restore |
| `--include-queries` | | flag | false | Include queries in restore |
| `--include-serviceconnections` | | flag | false | Include service connections in restore |
| `--include-pullrequests` | | flag | false | Include pull requests in restore |
| `--include-taskgroups` | | flag | false | Include task groups in restore |
| `--include-boards` | | flag | false | Include boards in restore |
| `--include-dashboards` | | flag | false | Include dashboards in restore |
| `--include-wikis` | | flag | false | Include wikis in restore |
| `--include-areas` | | flag | false | Include areas and iterations in restore |
| `--include-environments` | | flag | false | Include pipeline environments in restore |
| `--include-testplans` | | flag | false | Include test plans, suites, and runs in restore |
| `--include-test-runs` | | flag | false | Also restore test runs (requires `--include-testplans`) |
| `--test-plan-ids` | | string | all | Comma-separated test plan IDs to restrict test plan/suite/run restore to (requires `--include-testplans`) |
| `--fix-references-only` | | flag | false | Skip restore phases and only rewrite inline `#N` work item ID references in already-restored work items and wiki pages |
| `--mapping-file` | | string | auto | Path to id-mapping.json. When omitted, auto-discovered from `{BackupRoot}/{SourceOrg}/.metadata/id-mapping.json` |
| `--warnings-as-errors` | | flag | false | Treat warnings (partial failures) as errors - task fails on partial failures |
| `--ignore-warnings` | | flag | false | Ignore warnings (partial failures) - task succeeds even with partial failures |
| `--repositories` | | string | all | Comma-separated list of repository names |
| `--definitions` | | string | all | Comma-separated list of build definition IDs |
| `--release-name` | `-n` | string | none | Specific release definition name to restore (optional, restores all if not specified) |
| `--update-existing` | | flag | false | Update existing release definitions (default: skip existing) |
| `--ids` | | string | all | Comma-separated list of work item IDs to restore |
| `--max-revisions` | | int | all | Maximum number of most-recent revisions to replay per work item (default: all) |
| `--pr-ids` | | string | all | Comma-separated list of pull request IDs to restore |
| `--bypass-rules` | | bool | true | Bypass work item validation rules. When `true` (default), original `System.CreatedBy`, `System.CreatedDate`, `System.ChangedBy`, and `System.ChangedDate` are written to the target so authorship and timestamps are preserved. The identity must exist in the target org; unresolvable identities are skipped per-field. |
| `--queries-folder-path` | | string | all | Specific query folder path |
| `--task-group-names` | | string | all | Comma-separated list of task group names to restore |

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
  --ids "1001,1002,1003" ^
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

> **Note:** Branch policies and the default branch are automatically restored from `branch-policies.json` after branches are pushed. Server-generated fields (`id`, `revision`, `url`, `createdBy`) and unresolvable identity/pipeline references are stripped with warnings. If `branch-policies.json` is absent (pre-feature backup), the step is silently skipped.

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
| `--target-project` | | string | source | Target project name |
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
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--definitions` | | string | all | Comma-separated list of build definition IDs |
| `--target-org` | | string | source | Target organization URL |
| `--target-pat` | | string | source | Target PAT token |
| `--target-project` | | string | source | Target project name |
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

### release-restore

Restore release definitions only.

**Syntax:**
```bash
adobackup.exe release-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | **required** | Comma-separated list of source project names |
| `--release-name` | `-n` | string | none | Specific release definition name to restore (optional, restores all if not specified) |
| `--update-existing` | `-u` | flag | false | Update existing release definitions (default: skip existing) |
| `--target-organization-url` | | string | source | Target organization URL (for cross-org restore) |
| `--target-pat` | | string | source | Target PAT token |
| `--target-project` | | string | source | Target project name |

**Important Notes:**
- **Secret variables** are not restored (Azure DevOps API limitation)
  - Backed up as `null` with `isSecret: true` flag
  - Must be manually reconfigured in Azure DevOps after restore
- By default, skips existing release definitions (`--update-existing=false`)
- Use `--update-existing` to overwrite existing releases (use with caution)
- Cross-project restore is supported

**Examples:**

**Restore all release definitions:**
```bash
adobackup.exe release-restore -v
```

**Restore specific release definitions:**
```bash
adobackup.exe release-restore -p "MyProject" --release-name "Release-Prod" -v
```

**Restore to different project:**
```bash
adobackup.exe release-restore ^
  -p "SourceProject" ^
  --target-project "TargetProject" ^
  -v
```

**Update existing releases (overwrite):**
```bash
adobackup.exe release-restore --update-existing -v
```

**Cross-organization restore:**
```bash
adobackup.exe release-restore ^
  -p "SourceProject" ^
  --target-organization-url "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  --target-project "NewProject" ^
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
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--ids` | | string | all | Comma-separated list of work item IDs to restore |
| `--max-revisions` | | int | all | Maximum number of most-recent revisions to replay per work item |
| `--bypass-rules` | | bool | true | Bypass work item validation rules. When `true` (default), original `System.CreatedBy`, `System.CreatedDate`, `System.ChangedBy`, and `System.ChangedDate` are written to the target so authorship and timestamps are preserved. The identity must exist in the target org; unresolvable identities are skipped per-field. |
| `--fix-references-only` | | flag | false | Skip Phases 1–3 and only rewrite inline `#N` HTML references in description fields using the existing id-mapping.json. Use to retroactively fix references on an already-restored org without re-running the full restore. |
| `--mapping-file` | | string | auto | Path to id-mapping.json for `--fix-references-only`. When omitted, auto-discovered from `{BackupRoot}/{SourceOrg}/.metadata/id-mapping.json`. |
| `--target-org` | | string | source | Target organization URL |
| `--target-pat` | | string | source | Target PAT token |
| `--target-project` | | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Important Notes:**
- ⚠️ **Restore Areas & Iterations first** — Work item fields `System.AreaPath` and `System.IterationPath` must resolve to existing classification nodes in the target. For cross-project and cross-organization restores, always run `areas-restore` (or `restore-all --include-areas`) before restoring work items to ensure paths are resolved correctly.
- **Cross-organization restore** uses a four-phase approach:
  - **Phase 1** — Creates all work items in the target with new IDs and builds a source-to-target ID mapping, persisted incrementally to `.metadata/id-mapping.json`. Already-mapped items are automatically skipped on re-runs.
  - **Phase 2** — Relinks all relations (parent-child, related, predecessor-successor) using new target IDs. Development links are handled intelligently: git commit links are preserved as native artifact links where possible; closed pull request links and build run links that cannot be migrated are converted to fallback hyperlinks pointing to the source, preserving audit trails.
  - **Phase 3** — Stamps original `System.ChangedDate` on each work item via bypass-rules PATCH.
  - **Phase 4** — Rewrites inline work item references in HTML description fields (`System.Description`, `Microsoft.VSTS.Common.AcceptanceCriteria`, `Microsoft.VSTS.TCM.ReproSteps`, `Microsoft.VSTS.TCM.SystemInfo`): `href` attributes pointing to `/_workitems/edit/N`, `AB#N` mentions, and plain `#N` mentions are rewritten from source IDs to target IDs. IDs not present in the mapping are left unchanged.
- **Retroactive reference rewriting** — Use `--fix-references-only` to rewrite HTML references on an already-restored org without re-running Phases 1–3. The mapping file is auto-discovered from `{BackupRoot}/{SourceOrg}/.metadata/id-mapping.json`; override with `--mapping-file` if needed. Supports `--dry-run`.
- **Idempotent re-runs** — The mapping file is written after each Phase 1 creation; a partial restore can be safely re-run without creating duplicate work items or relations.
- **Single source project required** — When using `--target-project` or `--target-org`, specify exactly one source project with `-p`.
- **Author & timestamp preservation** — When `--bypass-rules true` (the default), original authorship and dates are preserved:
  - `System.CreatedBy` and `System.CreatedDate` are written on work item creation.
  - `System.ChangedBy` and `System.ChangedDate` are written per revision so the history timeline matches the source.
  - The identity (UPN/email) must exist in the target org. Unresolvable identities are silently skipped per-field so the rest of the item still restores.
  - Set `--bypass-rules false` to restore without authorship (all items and history will show the restore service account identity).
- **Comment attribution** — The ADO Comments API does not support setting the comment author. All restored comments are prepended with a markdown attribution block showing the original author and date:
  ```
  > 🗣️ Originally by **John Doe** (john@contoso.com) — Jan 15, 2023
  ```

**Examples:**

**Restore all work items:**
```bash
adobackup.exe workitems-restore -v
```

**Restore specific work items:**
```bash
adobackup.exe workitems-restore --ids "1,2,3,100" -v
```

**Restore with bypass rules disabled (no authorship preservation):**
```bash
adobackup.exe workitems-restore --bypass-rules false -v
```

**Restore from specific date:**
```bash
adobackup.exe workitems-restore -d "20250128" -v
```

**Restore with revision limit (replay 50 most-recent revisions per item):**
```bash
adobackup.exe workitems-restore --max-revisions 50 -v
```

**Restore specific work items with revision limit:**
```bash
adobackup.exe workitems-restore --ids "1001,1002" --max-revisions 25 -v
```

**Retroactively rewrite HTML references (auto-discovers mapping file):**
```bash
adobackup.exe workitems-restore --fix-references-only ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

**Retroactively rewrite HTML references with explicit mapping file:**
```bash
adobackup.exe workitems-restore --fix-references-only ^
  --mapping-file "C:\ADOBackups\oldorg\.metadata\id-mapping.json" ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

**Dry-run reference rewriting (preview without modifying work items):**
```bash
adobackup.exe workitems-restore --fix-references-only --dry-run ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
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
| `--target-project` | | string | source | Target project name |
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

### pullrequests-restore

Restore pull requests only.

**Syntax:**
```bash
adobackup.exe pullrequests-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--repositories` | `-r` | string | all | Comma-separated list of repository names |
| `--ids` | | string | all | Comma-separated list of pull request IDs to restore |
| `--target-org` | | string | source | Target organization URL |
| `--target-pat` | | string | source | Target PAT token |
| `--target-project` | | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Examples:**

**Restore all pull requests:**
```bash
adobackup.exe pullrequests-restore -v
```

**Restore specific repositories:**
```bash
adobackup.exe pullrequests-restore -p "ProjectA" -r "Repo1,Repo2" -v
```

**Restore to different organization:**
```bash
adobackup.exe pullrequests-restore ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat" ^
  -v
```

**Dry run (preview only):**
```bash
adobackup.exe pullrequests-restore --dry-run -v
```

**Restore specific pull requests:**
```bash
adobackup.exe pullrequests-restore --ids "1,2,5,10" -v
```

---

### serviceconnections-restore

Restore service connections only.

**Syntax:**
```bash
adobackup.exe serviceconnections-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--target-org` | | string | source | Target organization URL |
| `--target-pat` | | string | source | Target PAT token |
| `--target-project` | | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Examples:**

**Restore all service connections:**
```bash
adobackup.exe serviceconnections-restore -v
```

**Restore to different project:**
```bash
adobackup.exe serviceconnections-restore ^
  -p "SourceProject" ^
  --target-project "TargetProject" ^
  -v
```

---

### taskgroup-restore

Restore task groups only.

**Syntax:**
```bash
adobackup.exe taskgroup-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--groups` | `-g` | string | all | Comma-separated list of task group names |
| `--target-project` | | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Examples:**

**Restore all task groups:**
```bash
adobackup.exe taskgroup-restore -v
```

**Restore to different project:**
```bash
adobackup.exe taskgroup-restore ^
  -p "SourceProject" ^
  --target-project "TargetProject" ^
  -v
```

---

### boards-restore

Restore boards configuration only.

**Syntax:**
```bash
adobackup.exe boards-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--team` | `-t` | string | all | Team name |
| `--board` | `-b` | string | all | Board name |

**Examples:**

**Restore all boards:**
```bash
adobackup.exe boards-restore -v
```

**Restore all boards for a specific team:**
```bash
adobackup.exe boards-restore --team "TeamA" -v
```

**Restore boards for specific project:**
```bash
adobackup.exe boards-restore -p "MyProject" -v
```

---

### dashboards-restore

Restore dashboards only.

**Syntax:**
```bash
adobackup.exe dashboards-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--team` | `-t` | string | all | Team name |
| `--dashboard` | `-d` | string | all | Dashboard name |

**Examples:**

**Restore all dashboards:**
```bash
adobackup.exe dashboards-restore -v
```

**Restore all dashboards for a specific team:**
```bash
adobackup.exe dashboards-restore --team "TeamA" -v
```

**Restore dashboards for specific project:**
```bash
adobackup.exe dashboards-restore -p "MyProject" -v
```

---

### wiki-restore

Restore wikis from backup.

**Syntax:**
```bash
adobackup.exe wiki-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | required | Comma-separated list of source project names |
| `--backup-date` | `-d` | string | latest | Backup date (YYYYMMDD) |
| `--wiki` | `-w` | string | all | Specific wiki name to restore |
| `--target-organization-url` | | string | source | Target organization URL (for cross-org restore) |
| `--target-pat` | | string | source | Target organization PAT |
| `--target-project` | | string | source | Target project name |
| `--dry-run` | | flag | false | Preview changes without applying them |
| `--fix-references-only` | | flag | false | Skip restore and only rewrite inline `#N` work item ID references in already-restored wiki pages. Requires the ID mapping file produced by a previous `workitems-restore` run. |
| `--mapping-file` | | string | auto | Path to `id-mapping.json`. When omitted, auto-discovered from `{BackupRoot}/{SourceOrg}/.metadata/id-mapping.json`. |

**Important Notes:**
- **`--fix-references-only`** — Scans already-restored wiki pages on the target and rewrites work item ID references from source IDs to target IDs using the mapping produced by `workitems-restore`. Does not re-restore page content from backup. Idempotent — safe to run multiple times.
- Rewritten patterns: `](…/_workitems/edit/N…)` markdown links, bare `https://…/_workitems/edit/N` URLs, `AB#N` mentions, and plain `#N` mentions. IDs not in the mapping are left unchanged.
- When restoring to a different org, `--fix-references-only` requires `--target-organization-url` and `--target-pat`.

**Examples:**

**Restore all wikis:**
```bash
adobackup.exe wiki-restore -p "MyProject" -v
```

**Restore specific wiki:**
```bash
adobackup.exe wiki-restore -p "MyProject" -w "MyProject.wiki" -v
```

**Restore from specific backup date:**
```bash
adobackup.exe wiki-restore -p "MyProject" -d "20240115" -v
```

**Restore to different organization:**
```bash
adobackup.exe wiki-restore -p "MyProject" ^
  --target-organization-url "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

**Restore to different project:**
```bash
adobackup.exe wiki-restore -p "SourceProject" --target-project "TargetProject" -v
```

**Dry run:**
```bash
adobackup.exe wiki-restore -p "MyProject" --dry-run -v
```

**Rewrite work item ID references in already-restored wiki pages (auto-discovers mapping):**
```bash
adobackup.exe wiki-restore -p "MyProject" --fix-references-only ^
  --target-organization-url "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

**Dry-run reference rewrite (preview without modifying pages):**
```bash
adobackup.exe wiki-restore -p "MyProject" --fix-references-only --dry-run ^
  --target-organization-url "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

**Reference rewrite with explicit mapping file:**
```bash
adobackup.exe wiki-restore -p "MyProject" --fix-references-only ^
  --mapping-file "C:\ADOBackups\sourceorg\.metadata\id-mapping.json" ^
  --target-organization-url "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

---

### areas-restore

Restore areas and iterations (classification nodes) only.

**Syntax:**
```bash
adobackup.exe areas-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--areas-only` | | flag | false | Restore only area paths (not iteration paths) |
| `--iterations-only` | | flag | false | Restore only iteration paths (not area paths) |
| `--target-project` | `-t` | string | source | Target project name |
| `--dry-run` | | flag | false | Preview only |

**Important Notes:**
- **Additive restore** - Never deletes existing nodes, only creates/updates
- **Idempotent** - Safe to run multiple times without creating duplicates
- **Cross-project restore supported** - Use `--target-project` to restore to a different project
- **Selective restore** - Can restore areas-only or iterations-only
- **Node IDs may change** - Paths are preserved but IDs may differ

**Examples:**

**Restore all areas and iterations:**
```bash
adobackup.exe areas-restore -v
```

**Restore only area paths:**
```bash
adobackup.exe areas-restore --areas-only -v
```

**Restore only iteration paths:**
```bash
adobackup.exe areas-restore --iterations-only -v
```

**Restore to different project:**
```bash
adobackup.exe areas-restore ^
  -p "SourceProject" ^
  --target-project "TargetProject" ^
  -v
```

**Dry run (preview changes):**
```bash
adobackup.exe areas-restore --dry-run -v
```

**Restore specific project's areas only:**
```bash
adobackup.exe areas-restore ^
  -p "MyProject" ^
  --areas-only ^
  --dry-run ^
  -v
```

---

### environments-restore

Restore pipeline environments (definitions, resources, approvals, and checks).

**Syntax:**
```bash
adobackup.exe environments-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names |
| `--environment-names` | `-e` | string | all | Comma-separated list of environment names to restore |
| `--skip-resources` | | flag | false | Skip restoration of environment resources (Kubernetes, VM, etc.) |
| `--skip-approvals` | | flag | false | Skip restoration of environment approval configurations |
| `--skip-checks` | | flag | false | Skip restoration of environment check configurations |
| `--dry-run` | | flag | false | Preview only |

**Important Notes:**
- Supports **create or update** - existing environments are updated, not duplicated
- Resource registrations (Kubernetes, VMs) are restored if the target agent/cluster is reachable
- Approval and check configurations are fully restored
- Use `--skip-resources` when target infrastructure differs from source

**Examples:**

**Restore all environments:**
```bash
adobackup.exe environments-restore -v
```

**Dry-run (preview without making changes):**
```bash
adobackup.exe environments-restore --dry-run -v
```

**Restore environments to specific projects:**
```bash
adobackup.exe environments-restore -p "ProjectA,ProjectB" -v
```

**Restore specific environments:**
```bash
adobackup.exe environments-restore -e "Production,Staging" -v
```

**Restore only environment definitions (skip sub-resources):**
```bash
adobackup.exe environments-restore ^
  --skip-resources ^
  --skip-approvals ^
  --skip-checks ^
  -v
```

**Restore environments but skip resource registration:**
```bash
adobackup.exe environments-restore -p "ProjectA" --skip-resources -v
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
| `--tenant-url` | `-t` | string | No | Azure DevOps tenant URL to validate against |

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

### Pattern 5: Incremental Pull Requests Backup

**Initial full backup of pull requests:**
```bash
adobackup.exe backup-all --include-pullrequests -v
```

**Daily incremental backup (only new/modified PRs):**
```bash
adobackup.exe backup-all --include-pullrequests -i -v
```

**Incremental with aggressive refresh (every 12 hours):**
```bash
adobackup.exe backup-all ^
  --include-pullrequests ^
  -i ^
  --active-pr-refresh-hours 12 ^
  -v
```

**Backup only active PRs with incremental mode:**
```bash
adobackup.exe pullrequests-backup ^
  --status Active ^
  -i ^
  -v
```

### Pattern 6: Pull Requests Migration Workflow

**Backup from source:**
```bash
adobackup.exe pullrequests-backup -p "SourceProject" -r "Repo1" -v
```

**Restore to different organization:**
```bash
adobackup.exe pullrequests-restore ^
  -p "SourceProject" ^
  -r "Repo1" ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

### Pattern 7: Performance Optimization with MaxParallelism

**High-performance backup on server (8 concurrent operations):**
```bash
adobackup.exe backup-all --MaxParallelism 8 -v
```

**Laptop-friendly backup (2 concurrent operations):**
```bash
adobackup.exe backup-all --MaxParallelism 2 -v
```

**Large repository backup with increased parallelism:**
```bash
adobackup.exe git-backup ^
  -p "LargeProject" ^
  --MaxParallelism 6 ^
  -v
```

**Sequential backup for debugging (no parallelism):**
```bash
adobackup.exe backup-all --MaxParallelism 1 -v
```

**Restore with custom parallelism:**
```bash
adobackup.exe restore-all ^
  --MaxParallelism 4 ^
  --dry-run ^
  -v
```

**Azure Pipeline with optimal settings:**
```bash
adobackup.exe backup-all ^
  --OrganizationUrl "$(System.CollectionUri)" ^
  --Pat "$(System.AccessToken)" ^
  --BackupRoot "$(Build.ArtifactStagingDirectory)/Backups" ^
  --MaxParallelism 6 ^
  -i ^
  -v
```

### Pattern 8: Areas & Iterations Workflow

**Backup areas and iterations:**
```bash
adobackup.exe backup-all --include-areas -v
```

**Or use dedicated command:**
```bash
adobackup.exe areas-backup -p "MyProject" -v
```

**Restore both areas and iterations (default):**
```bash
adobackup.exe areas-restore -p "SourceProject" -v
```

**Restore only area paths (organizational structure):**
```bash
adobackup.exe areas-restore ^
  -p "SourceProject" ^
  --areas-only ^
  -v
```

**Restore only iteration paths (sprint schedules):**
```bash
adobackup.exe areas-restore ^
  -p "SourceProject" ^
  --iterations-only ^
  -v
```

**Cross-project restore (replicate team structure):**
```bash
adobackup.exe areas-restore ^
  -p "SourceProject" ^
  --target-project "TargetProject" ^
  --dry-run ^
  -v
```

**Cross-project restore (different project name):**
```bash
adobackup.exe areas-restore ^
  -p "SourceProject" ^
  --target-project "NewProject" ^
  -v
```

**Restore with selective component (areas + work items):**
```bash
adobackup.exe restore-all ^
  --include-areas ^
  --include-workitems ^
  -p "MyProject" ^
  -v
```

---

### Pattern 9: Cross-Organization Work Items Restore

**Backup work items from source organization:**
```bash
adobackup.exe backup-all ^
  --include-workitems ^
  -p "SourceProject" ^
  -v
```

**Preview cross-org restore (dry run):**
```bash
adobackup.exe workitems-restore ^
  -p "SourceProject" ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "your-target-pat" ^
  --target-project "TargetProject" ^
  --dry-run ^
  -v
```

**Execute cross-org work items restore:**
```bash
adobackup.exe workitems-restore ^
  -p "SourceProject" ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "your-target-pat" ^
  --target-project "TargetProject" ^
  -v
```

**Cross-org restore using restore-all (work items only):**
```bash
adobackup.exe restore-all ^
  -p "SourceProject" ^
  --include-workitems ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "your-target-pat" ^
  --target-project "TargetProject" ^
  -v
```

**Notes:**
- Phase 1 creates all work items in the target with new IDs and builds a source-to-target ID mapping saved to `.metadata/id-mapping.json`. The mapping is written **incrementally after each item creation**, so a partial restore can be safely re-run without creating duplicate work items.
- Phase 2 relinks all relations using the new target IDs. Git commit links are preserved as native artifact links where the target repository exists. Closed pull request links and build run links that cannot be migrated are converted to fallback hyperlinks pointing to the source, preserving navigation and audit trail.
- Re-runs are idempotent — already-mapped items are detected and duplicate relations are skipped.
- Specify exactly one source project with `-p` when using `--target-project` or `--target-org`.
- ⚠️ **Always restore Areas & Iterations before work items** (or use `restore-all` which handles ordering automatically).

---

### Pattern 11: Cross-Organization Full Migration Workflow

The recommended sequence for a cross-organization work item migration. Areas and iterations **must** be restored before work items so that `System.AreaPath` and `System.IterationPath` fields resolve correctly.

**Step 1 — Back up source (if not already done):**
```bash
adobackup.exe backup-all ^
  --include-areas ^
  --include-workitems ^
  -p "SourceProject" ^
  -v
```

**Step 2 — Restore areas and iterations to target first:**
```bash
adobackup.exe restore-all ^
  -p "SourceProject" ^
  --include-areas ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "your-target-pat" ^
  --target-project "TargetProject" ^
  --dry-run ^
  -v

adobackup.exe restore-all ^
  -p "SourceProject" ^
  --include-areas ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "your-target-pat" ^
  --target-project "TargetProject" ^
  -v
```

**Step 3 — Restore work items (idempotent; safe to re-run):**
```bash
adobackup.exe workitems-restore ^
  -p "SourceProject" ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "your-target-pat" ^
  --target-project "TargetProject" ^
  --dry-run ^
  -v

adobackup.exe workitems-restore ^
  -p "SourceProject" ^
  --target-org "https://dev.azure.com/targetorg" ^
  --target-pat "your-target-pat" ^
  --target-project "TargetProject" ^
  -v
```

**Development link behavior after cross-org restore:**
- **Git commit links** — preserved as native artifact links where the target repository exists
- **Closed pull request links** — converted to fallback hyperlinks; source PR URL preserved for navigation
- **Build run links** — converted to fallback hyperlinks; source build URL preserved for audit trail
- **Work item relations** — automatically relinked using `.metadata/id-mapping.json`; idempotent on re-runs

---

### Pattern 10: Retroactive HTML Reference Rewriting

Use `--fix-references-only` to rewrite stale source IDs in work item description fields after a cross-org restore has already been completed.

**Rewrite references using the auto-discovered mapping (most common):**
```bash
# id-mapping.json is picked up from {BackupRoot}/{SourceOrg}/.metadata/ automatically
adobackup.exe workitems-restore --fix-references-only ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

**Dry run first to see what would change:**
```bash
adobackup.exe workitems-restore --fix-references-only --dry-run ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

**Supply an explicit mapping file from a non-default location:**
```bash
adobackup.exe workitems-restore --fix-references-only ^
  --mapping-file "D:\Backups\sourceorg\.metadata\id-mapping.json" ^
  --target-org "https://dev.azure.com/neworg" ^
  --target-pat "new-pat-token" ^
  -v
```

**Notes:**
- Rewrites three patterns per field: `href="…/_workitems/edit/N"` attributes, `AB#N` mentions, and plain `#N` mentions
- Covered fields: `System.Description`, `Microsoft.VSTS.Common.AcceptanceCriteria`, `Microsoft.VSTS.TCM.ReproSteps`, `Microsoft.VSTS.TCM.SystemInfo`
- IDs not present in the mapping are left unchanged
- Idempotent — safe to run multiple times
- Phase 4 of a normal cross-org restore runs this automatically; `--fix-references-only` is for retroactive runs

---

### Pattern 12: Work Item Revision Control

Use `--max-revisions` to control how many revision history entries are stored and replayed per work item. Useful for large projects with deep revision histories.

**Backup with revision limit (store 50 most-recent revisions per item):**
```bash
adobackup.exe workitems-backup --max-revisions 50 -v
```

**Restore replaying limited revisions:**
```bash
adobackup.exe workitems-restore --max-revisions 50 -v
```

**End-to-end selective backup and restore with matching limits:**
```bash
adobackup.exe backup-all --include-workitems --max-revisions 100 -v
adobackup.exe restore-all --include-workitems --max-revisions 100 -v
```

**Incremental backup of large project with revision cap:**
```bash
adobackup.exe backup-all -i --include-workitems --max-revisions 50 -v
```

**Notes:**
- `--max-revisions` retains the N most-recent revisions per work item (by revision number)
- When omitted, all revisions are stored or replayed (full fidelity)
- Recommended values: 50 for large projects, 100 for standard projects
- Applies independently to backup and restore — set on each side as needed
- Revision data is stored in `revisions.json` alongside each work item's backup folder

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

---

### testplans-backup

Backup Azure DevOps test plans, test suites, and test runs.

**Syntax:**
```bash
adobackup.exe testplans-backup [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of project names to backup test plans from |
| `--test-runs-days` | | int | 90 | Number of days of test run history to backup (default: 90 days, 0 = all) |
| `--include-all-test-runs` | | flag | false | Backup all test runs regardless of date (overrides `--test-runs-days`) |

**What Gets Backed Up:**
- **Test Plans**: Top-level test plan containers with metadata (name, description, state, iteration, area path)
- **Test Suites**: Hierarchical test suite structure (static, requirement-based, and query-based suites)
- **Test Cases**: Test case associations within suites
- **Test Runs**: Test run history (configurable by date range, default: last 90 days)
- **Test Results**: Detailed test results and outcomes

**Backup Structure:**
```
test-plans/
  {project}/
    {planId}/
      plan.json
      root-suite.json
      suite-{suiteId}.json
      ...
.metadata/
  test-plans-metadata.json
test-runs/
  {project}/
    {runId}.json    # run metadata with its results embedded
```

**Examples:**

**Backup all test plans (last 90 days of runs):**
```bash
adobackup.exe testplans-backup -v
```

**Backup specific projects:**
```bash
adobackup.exe testplans-backup -p "Project1,Project2" -v
```

**Backup with last 30 days of test runs:**
```bash
adobackup.exe testplans-backup --test-runs-days 30 -v
```

**Backup all test runs (entire history):**
```bash
adobackup.exe testplans-backup --include-all-test-runs -v
```

**Notes:**
- Test plans are backed up hierarchically (root suite → child suites → test cases)
- Test runs are filtered by completion date (default: last 90 days)
- Use `--test-runs-days 0` or `--include-all-test-runs` for complete history (may take a long time for extensive test history)
- Requirement-based and query-based suites are fully preserved during backup
- Restoring test runs is optional and off by default - see `testplans-restore`'s `--include-test-runs` flag below
- The run-level completed timestamp on a *restored* run reflects when the restore ran, not the
  original completion time - Azure DevOps does not accept a historical value there. Per-result dates,
  outcomes, and durations are preserved exactly.
- `RunBy` (who ran the test) cannot be restored as a structured field - Azure DevOps requires a
  resolvable identity, not just a name. The original value is preserved as a "Originally run by: X"
  annotation in the result's comment instead.

---

### testplans-restore

Restore Azure DevOps test plans, test suites, and test runs from backup.

**Syntax:**
```bash
adobackup.exe testplans-restore [options]
```

**Options:**

| Option | Short | Type | Default | Description |
|--------|-------|------|---------|-------------|
| `--projects` | `-p` | string | all | Comma-separated list of source project names from backup |
| `--target-project` | | string | source | Target project name (for cross-project restore) |
| `--include-test-runs` | | flag | false | Also restore test runs (in addition to plans and suites). Same-project restore only. |
| `--test-plan-ids` | | string | all | Comma-separated test plan IDs from the backup to restrict restore to (e.g. `101,102`). Applies to plans/suites and, combined with `--include-test-runs`, scopes which plans' test runs get restored. |
| `--dry-run` | | flag | false | Preview changes without applying them |

**What Gets Restored:**
- **Test Plans**: Test plan containers with original metadata
- **Test Suites**: Hierarchical suite structure (depth-first recursive restoration)
- **Test Cases**: Test case associations (validated against existing work items)
- **Test Runs** (optional): Test run history with results (requires `--include-test-runs`)

**ID Mapping:**
- Test plan IDs are mapped from source → target
- Test suite IDs are mapped hierarchically
- Mappings saved to `.metadata/test-plan-id-mapping.json` and `test-suite-id-mapping.json`

**Automatic Conversions:**
- **Requirement-Based Suites**: Converted to static suite if requirement work item missing in target
- **Query-Based Suites**: Converted to static suite if query missing in target
- Conversions are logged with warnings

**Work Item Validation:**
- Test case work items are validated before association
- Missing test cases are skipped (not created)
- Summary report shows `successCount/skippedCount` for each suite

**Examples:**

**Restore all test plans (plans and suites only):**
```bash
adobackup.exe testplans-restore -v
```

**Restore specific projects:**
```bash
adobackup.exe testplans-restore -p "Project1,Project2" -v
```

**Restore to different project:**
```bash
adobackup.exe testplans-restore -p "SourceProject" --target-project "TargetProject" -v
```

**Restore with test runs (same project only):**
```bash
adobackup.exe testplans-restore --include-test-runs -v
```

**Restore test runs for one plan only (avoids duplicating runs for other plans):**
```bash
adobackup.exe testplans-restore --include-test-runs --test-plan-ids 101 -v
```

**Restore just a couple of specific plans (plans/suites only, no test runs):**
```bash
adobackup.exe testplans-restore --test-plan-ids 101,102 -v
```

**Dry run (preview only):**
```bash
adobackup.exe testplans-restore --dry-run -v
```

**Cross-project restore with dry run:**
```bash
adobackup.exe testplans-restore ^
  -p "OldProject" ^
  --target-project "NewProject" ^
  --dry-run ^
  -v
```

**Notes:**
- Test plans and suites are always restored; test runs are optional (`--include-test-runs`)
- Suite hierarchy is preserved during restore
- Suite type conversions (requirement/query → static) are automatic when needed
- Test case associations are validated against target project work items
- Use `--dry-run` to preview changes before actual restore
- Test plan/suite ID mapping files enable re-running restore idempotently (existing plans/suites are
  reused, not duplicated) - **test runs are the exception**: there is no run ID mapping, so re-running
  `--include-test-runs` creates a new set of test runs each time rather than skipping already-restored ones
- Test run restore is **same-project only** in this version; requesting `--include-test-runs` together
  with `--target-project` (cross-project) logs a warning and skips test runs for that plan (plans/suites
  still restore normally)
- `--include-test-runs` applies to every plan being restored in one invocation, and since test runs aren't
  idempotent (see above), restoring the whole project's history repeatedly duplicates runs across all its
  plans - use `--test-plan-ids` to scope a restore (and any re-run) to just the plan(s) you actually need
- A restored run's completed timestamp reflects when the restore ran, not the original completion time -
  Azure DevOps does not accept a historical value there. Per-result dates/outcomes are preserved exactly
- `RunBy` is not restorable as a structured field (Azure DevOps requires a resolvable identity, not just a
  name) - the original value is preserved as an "Originally run by: X" annotation in the result's comment

---

