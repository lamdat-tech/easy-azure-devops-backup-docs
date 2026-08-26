# Resource Dependencies and Restore Order

Understanding the dependencies between Azure DevOps resources is critical for successful backup and restore operations. This guide explains the proper order of operations and why dependencies matter.

## Table of Contents

1. [Overview](#overview)
2. [Dependency Graph](#dependency-graph)
3. [Backup Order](#backup-order)
4. [Restore Order](#restore-order)
5. [Resource Dependencies Explained](#resource-dependencies-explained)
6. [Best Practices](#best-practices)
7. [Troubleshooting Dependencies](#troubleshooting-dependencies)

## Overview

Azure DevOps resources have implicit dependencies on each other. Restoring resources in the wrong order can cause failures or broken references. The `adobackup` utility handles these dependencies automatically, but understanding them helps you troubleshoot issues and make informed decisions about selective restores.

### Key Principle

**Always restore dependencies before dependent resources.**

Example: Build definitions reference Git repositories, so repositories must be restored before build definitions.

## Dependency Graph

```
┌─────────────────────────────────────────────────────────────┐
│                      RESTORE ORDER                           │
│                    (Top to Bottom)                           │
└─────────────────────────────────────────────────────────────┘

1. Git Repositories
   ├─ No dependencies
       (Always safe to restore first)

2. Pull Requests
   ├─ Depends on: Git Repositories (PRs belong to repositories)
       (Restore after Git repositories)

3. Service Connections
   ├─ No resource dependencies
       (Can restore anytime, typically after Git)

4. Pipeline Variables / Variable Groups
   ├─ No resource dependencies
       (Can restore anytime, typically after Service Connections)

5. Build Definitions
   ├─ Depends on: Git Repositories (for repository references)
   ├─ Depends on: Service Connections (for deployment/resource connections)
   └─ Depends on: Variable Groups (optional, for pipeline variables)

6. Areas & Iterations
   ├─ No resource dependencies
       (Must be restored before Work Items and Shared Queries so that
        area/iteration path fields on work items resolve correctly)

7. Work Items
   ├─ Depends on: Areas & Iterations (REQUIRED for cross-project/cross-org restores; recommended for same-project)
   ├─ Depends on: Git Repositories (optional, for commit links)
   └─ Depends on: Build Definitions (optional, for build links)

8. Shared Queries
   ├─ Depends on: Work Items (queries search for work items)
   ├─ Depends on: Areas (for area path filters)
   └─ Depends on: Iterations (for iteration path filters)
```

## Backup Order

The backup order is more flexible than restore order because you're just capturing data, not creating references. However, `adobackup` follows this recommended order:

### Recommended Backup Sequence

```bash
# Order doesn't affect backup functionality, but this is the logical flow:

1. Git Repositories       # Foundation data
2. Pull Requests          # Git-related data
3. Service Connections    # Deployment/resource credentials
4. Pipeline Variables     # Configuration data
5. Build Definitions      # References Git, Service Connections, and Variables
6. Build History          # Historical data
7. Work Items            # Can reference any resource
8. Shared Queries         # Queries against work items
```

**Note:** Backup order doesn't create dependencies - you can backup any resource independently. When you run `adobackup.exe backup-all`, resources are backed up in parallel where possible (per repository, per pull request, per connection, per definition, per build, per group, in batches for work items, per query).

## Restore Order

The restore order is **critical** because creating dependent resources before their dependencies will fail or create broken references.

### Mandatory Restore Sequence

```bash
# MUST restore in this order for successful operation:

1. ✓ Git Repositories FIRST
   └─ Creates repositories that build definitions and pull requests will reference

2. ✓ Pull Requests SECOND
   └─ Restores pull requests that belong to Git repositories

3. ✓ Service Connections THIRD
   └─ Creates service connections that build definitions may reference

4. ✓ Pipeline Variables FOURTH  
   └─ Creates variable groups that pipelines may reference

5. ✓ Build Definitions FIFTH
   └─ Now can reference existing Git repos, service connections, and variables

6. ✓ Areas & Iterations SIXTH ⚠️ **Required before Work Items**
   └─ Classification nodes (area paths, iteration paths) must exist before work items
      reference them — especially important for cross-project and cross-organization restores

7. ✓ Work Items SEVENTH
   └─ Can link to commits (Git) and builds; area/iteration paths must already exist

8. ✓ Shared Queries LAST
   └─ Queries search for work items (must exist first); area/iteration filters must already exist
```

**The utility enforces this order automatically** when you run `adobackup.exe restore-all` - you cannot change it.

### Selective Restore Order

If you use selective restore with include flags or individual restore commands, **you must respect dependencies**:

#### ✅ SAFE Selective Restores

```bash
# Restore only Git (no dependencies)
adobackup.exe restore-all --include-git

# Restore Git + Pull Requests (PRs depend on Git)
adobackup.exe restore-all --include-git --include-pullrequests

# Restore Git + Service Connections + Variables (no dependency between them)
adobackup.exe restore-all --include-git --include-serviceconnections --include-variables

# Restore Areas & Iterations + Work Items (areas must come first)
adobackup.exe restore-all --include-areas --include-workitems

# Restore everything except queries (queries depend on work items)
adobackup.exe restore-all --include-git --include-pullrequests --include-serviceconnections --include-variables --include-builds --include-areas --include-workitems
```

#### ❌ UNSAFE Selective Restores

```bash
# BAD: Restore builds without Git repos
# Will fail if build definitions reference repositories that don't exist
adobackup.exe restore-all --include-builds

# BAD: Restore queries without work items
# Queries will be created but won't find any work items to query
adobackup.exe restore-all --include-queries

# BAD: Restore work items without areas/iterations (cross-project or cross-org)
# Area/iteration path fields fall back to project root; paths are not preserved
adobackup.exe restore-all --include-workitems --target-project "TargetProject"
```

## Resource Dependencies Explained

### 1. Git Repositories

**Dependencies:** None

**Depended On By:**
- Pull Requests (PRs belong to repositories)
- Build Definitions (YAML pipelines reference repos)
- Work Items (commits can be linked to work items)

**Why Restore First:**
- Build definitions fail validation if referenced repository doesn't exist
- YAML pipeline files are stored in repositories
- Build definitions include repository IDs that must be valid

**Example Issue if Not First:**
```
Error: Repository 'RepoName' not found in project 'ProjectName'
Cause: Restored build definition before repository
Solution: Restore Git repositories first
```

### 2. Pull Requests

**Dependencies:**
- Git Repositories (PRs belong to repositories)

**Depended On By:** None

**Why Restore After Git:**
- Pull requests are associated with specific repositories
- Repository must exist before PRs can be created
- PR metadata includes repository IDs that must be valid

**Important Behaviors:**
- Pull requests are restored as metadata only (code changes are in Git history)
- Comments, reviews, and status are preserved
- Work item links in PRs are preserved if work items exist
- Pull request IDs are preserved when restoring to same project
- Can filter by status during backup: All, Active, Completed, Abandoned
- Can restore specific PR IDs or all PRs from backup

**Example Issue if Not After Git:**
```
Error: Repository 'RepoName' not found when restoring pull request
Cause: Restored pull requests before Git repository
Solution: Restore Git repositories before pull requests
```

### 3. Service Connections

**Dependencies:** None (service connections are standalone credentials)

**Depended On By:**
- Build Definitions (pipelines use service connections for deployments)

**Why Restore Before Builds:**
- Build definitions may reference service connections by ID
- Missing service connection references cause warnings (not failures)
- Credentials/secrets must be re-entered manually after restore

**Important Behaviors:**
- Service connections are updated if they exist (not skipped)
- **Secrets/credentials are NOT backed up** - must be manually re-configured
- Connection metadata is preserved (type, project reference, etc.)
- Authorization for pipelines must be reconfigured manually

**Example Issue:**
```
Warning: Service connection 'AzureConnection' referenced in pipeline but not found
Cause: Restored builds before service connections
Impact: Pipeline runs but may fail during deployment steps
Solution: Restore service connections before build definitions, then re-enter credentials
```

**Important Limitation:**
- **Secret values/credentials are NOT backed up** - must be manually re-entered
- After restore, go to Project Settings → Service Connections → Edit each connection
- Re-enter passwords, keys, certificates, or other secrets

### 4. Pipeline Variables / Variable Groups

**Dependencies:** None (variables are standalone configuration)

**Depended On By:**
- Build Definitions (pipelines can reference variable groups)

**Why Restore Before Builds:**
- Build definitions may reference variable groups by ID
- Missing variable group references cause warnings (not failures)
- Secret values must be re-entered manually

**Example Issue:**
```
Warning: Variable group 'VarGroupName' referenced in pipeline but not found
Cause: Restored builds before variable groups
Impact: Pipeline runs but may fail due to missing variables
Solution: Restore variable groups before build definitions
```

**Important Limitation:**
- **Secret values are NOT backed up** - must be manually re-entered
- Variable groups are updated if they exist (not skipped)

### 5. Build Definitions

**Dependencies:**
- Git Repositories (for repository references)
- Service Connections (for deployment/resource connections)
- Variable Groups (optional, for pipeline variables)

**Depended On By:**
- Work Items (builds can be linked to work items)
- Build History (historical build runs)

**Why Restore After Git/Service Connections/Variables:**
- Build definitions contain repository IDs
- YAML pipelines are stored in repositories (must exist)
- Build definitions reference service connections for deployments
- Variable group references are validated

**Cross-Project Repository References:**
Build definitions can reference repositories in other projects:
```yaml
resources:
  repositories:
    - repository: SharedRepo
      type: git
      name: OtherProject/SharedRepo  # Cross-project reference
```

For cross-project scenarios:
1. Restore all Git repositories first (all projects)
2. Then restore build definitions

### 6. Areas & Iterations

**Dependencies:** None

**Depended On By:**
- Work Items (`System.AreaPath` and `System.IterationPath` fields reference classification nodes)
- Shared Queries (WIQL filters by area path and iteration path)

**Why Restore Before Work Items:**
- Work item fields `System.AreaPath` and `System.IterationPath` must resolve to existing nodes
- For cross-project and cross-organization restores, paths are rewritten to the target project; those target nodes must already exist or the restore will fall back to the root path or emit warnings
- Shared queries that filter by area/iteration return no results until the nodes exist

**Important Behaviors:**
- Restore is **additive only** — existing nodes are never deleted
- **Idempotent** — safe to run multiple times without creating duplicates
- `--areas-only` restores only area paths; `--iterations-only` restores only iteration paths
- Cross-project and cross-organization restore is fully supported

**Example Commands:**
```bash
# Restore both areas and iterations (recommended before work items)
adobackup.exe areas-restore -p "SourceProject" --target-project "TargetProject" -v

# Or use restore-all with selective include
adobackup.exe restore-all --include-areas --include-workitems -p "SourceProject" -v
```

---

### 7. Work Items

**Dependencies:**
- **Areas & Iterations** (⚠️ **Required** for cross-project/cross-org; strongly recommended for same-project so that area/iteration path fields resolve correctly)
- Git Repositories (optional, for commit and development links)
- Build Definitions (optional, for build run links)

**Depended On By:**
- Shared Queries (queries search work items)

**Why Restore After Areas/Iterations and Git/Builds:**
- `System.AreaPath` and `System.IterationPath` fields must resolve to existing classification nodes; without them the restore warns and falls back to the project root
- Work items can link to Git commits (requires repository to exist)
- Work items can link to builds (requires build definition to exist)
- Development links (commits, pull requests, builds) are preserved when possible; fallback hyperlinks are created for closed PRs and build runs that cannot be migrated natively

**Cross-Organization Restore — Two-Phase Process:**
- **Phase 1** — Creates all work items in the target with new IDs and builds a source-to-target ID mapping persisted to `.metadata/id-mapping.json`. Already-mapped items are skipped on re-runs.
- **Phase 2** — Relinks all relations (parent-child, related, predecessor-successor) and development links using the persisted mapping. Cross-org work-item URLs are rewritten; commit links are kept as native artifact links where possible; closed PR and build-run links are converted to fallback hyperlinks pointing to the source.
- **Idempotent re-runs** — The mapping file is written incrementally after each Phase 1 creation, so a partial restore can be safely re-run without creating duplicates.

**Important Behaviors:**
- Existing work items are **updated** (not skipped)
- Deleted work items are **recreated with new IDs**
- No comments are added to work item history about the restore
- `--bypass-rules` skips field validation (recommended for cross-process-template restores)

**Cross-Project/Cross-Org Requirement:**
- Specify exactly one source project with `-p` when using `--target-project` or `--target-org`
- Restore Areas & Iterations **first** so classification paths exist in the target before work items are created

### 8. Shared Queries

**Dependencies:**
- Work Items (queries search for work items)
- Areas & Iterations (queries filter by area paths and iteration paths — nodes must exist)

**Depended On By:** None

**Why Restore Last:**
- Queries are WIQL (Work Item Query Language) that search work items
- Query will be created but return no results if work items don't exist
- Area and iteration references are NOT updated in cross-project restore

**Important Behaviors:**
- Existing queries are **updated** (not skipped) - query definition is overwritten
- Queries keep **original project/area references** when restored to different project
- Area paths and iteration paths are **NOT** updated automatically
- Must manually update queries after cross-project restore

**Example Issue:**
```
Query created successfully but returns no results
Cause: Work items haven't been restored yet
Solution: Restore work items before queries
```

## Best Practices

### ✅ DO: Follow the Recommended Order

```bash
# Full restore - automatic correct order
adobackup.exe restore-all -v

# Selective restore - respect dependencies
# Step 1: Git first
adobackup.exe git-restore -p "ProjectA" -v

# Step 2: Then builds
adobackup.exe build-restore -p "ProjectA" -v
```

### ✅ DO: Use Dry-Run to Verify Dependencies

```bash
# Preview restore to check for dependency issues
adobackup.exe restore-all --dry-run -v

# Look for warnings about missing dependencies in output
```

### ✅ DO: Restore Complete Dependency Chains

```bash
# If restoring builds, also restore Git repos, service connections, and variables
adobackup.exe restore-all --include-git --include-serviceconnections --include-variables --include-builds -v
# ✓ Includes: Git + Service Connections + Variables + Builds (complete chain)
```

### ❌ DON'T: Restore Without Dependencies

```bash
# BAD: Builds without repos
adobackup.exe restore-all --include-builds -v
# ❌ Build definitions will reference non-existent repositories

# BAD: Queries without work items  
adobackup.exe restore-all --include-queries -v
# ❌ Queries will be created but find nothing
```

### ❌ DON'T: Assume Broken References Fix Themselves

```bash
# If you restore in wrong order, re-running won't automatically fix broken references
# You must delete and re-restore in correct order
```

## Troubleshooting Dependencies

### Issue 1: Build Definition Restore Fails - Repository Not Found

**Error:**
```
Failed to restore build definition: Repository 'MyRepo' not found in project 'MyProject'
```

**Cause:** Build definition references a repository that doesn't exist yet.

**Solution:**
```bash
# Step 1: Verify repository exists in backup
adobackup.exe git-restore -p "MyProject" --dry-run -v

# Step 2: Restore Git repositories first
adobackup.exe git-restore -p "MyProject" -v

# Step 3: Then restore build definitions
adobackup.exe build-restore -p "MyProject" -v
```

### Issue 2: Build Definition References Variable Group Not Found

**Warning:**
```
Warning: Variable group 'MyVarGroup' referenced but not found
```

**Cause:** Build definition references a variable group that doesn't exist.

**Impact:** Pipeline may run but fail if it uses variables from the missing group.

**Solution:**
```bash
# Step 1: Restore variable groups
adobackup.exe variables-restore -p "MyProject" -v

# Step 2: Manually re-enter secret values (not backed up)
# Go to Azure DevOps ? Pipelines ? Library ? MyVarGroup
# Re-enter all secret variable values

# Step 3: Restore or update build definitions
adobackup.exe build-restore -p "MyProject" -v
```

### Issue 3: Query Returns No Results After Restore

**Symptom:** Query created successfully but returns zero results.

**Cause:** Work items haven't been restored yet, or query references wrong project/areas.

**Solution:**

**For Same-Project Restore:**
```bash
# Restore work items
adobackup.exe workitems-restore -p "MyProject" -v

# Queries will now find work items
```

**For Cross-Project Restore:**
```bash
# Restore work items to target project
adobackup.exe workitems-restore -p "SourceProject" --target-project "TargetProject" -v

# Manually update queries to reference TargetProject areas/iterations
# 1. Go to Azure DevOps ? Boards ? Queries
# 2. Edit each query
# 3. Update "Area Path" and "Iteration Path" to reference TargetProject
```

### Issue 4: Work Item Area/Iteration Path Warnings or Falls Back to Root

**Symptom:** Work items restore successfully but `System.AreaPath` or `System.IterationPath` values are replaced with the project root or warnings appear about missing classification nodes.

**Cause:** Area or iteration nodes were not present in the target project when work items were restored.

**Solution:**
```bash
# Step 1: Restore areas and iterations first
adobackup.exe areas-restore -p "SourceProject" --target-project "TargetProject" -v

# Step 2: Then restore work items
adobackup.exe workitems-restore -p "SourceProject" --target-project "TargetProject" -v
```

**For cross-organization restore:**
```bash
# Step 1: Restore areas and iterations to the target organization
adobackup.exe areas-restore \
  -p "SourceProject" \
  --target-org "https://dev.azure.com/targetorg" \
  --target-pat "$(Target.AdoPat.RW)" \
  --target-project "TargetProject" \
  -v

# Step 2: Restore work items
adobackup.exe workitems-restore \
  -p "SourceProject" \
  --target-org "https://dev.azure.com/targetorg" \
  --target-pat "$(Target.AdoPat.RW)" \
  --target-project "TargetProject" \
  -v
```

---

### Issue 5: Work Item Links Are Broken After Restore

**Symptom:** Work items restored successfully but links to commits/builds are broken.

**Causes:**
- Git repositories or builds not restored yet
- Repository/build IDs changed during restore
- Cross-project or cross-org restore (work items get new IDs)

**Solution:**

**For Same-Project Restore:**
```bash
# Restore in correct order
adobackup.exe restore-all -v  # Automatic correct order includes areas before work items

# Links should be preserved with original IDs
```

**For Cross-Organization Restore:**
- **Git commit links** are preserved as native artifact links where the target repository exists
- **Closed pull request links** are converted to fallback hyperlinks pointing to the source PR — the source PR URL is preserved so reviewers can still navigate to the original discussion
- **Build run links** are converted to fallback hyperlinks pointing to the source build run — useful for audit trail
- **Work item relations** (parent-child, related, predecessor-successor) are automatically relinked using the source-to-target ID mapping persisted in `.metadata/id-mapping.json`

```bash
# Idempotent re-run — already-mapped items are skipped, no duplicates created
adobackup.exe workitems-restore \
  -p "SourceProject" \
  --target-org "https://dev.azure.com/targetorg" \
  --target-pat "your-target-pat" \
  --target-project "TargetProject" \
  -v
```

### Issue 6: Cross-Project Build Definition Fails

**Error:**
```
Build failed: Repository 'OtherProject/SharedRepo' not found
```

**Cause:** Build definition references repository in a different project that hasn't been restored.

**Solution:**
```bash
# Step 1: Restore Git repos from ALL referenced projects
adobackup.exe git-restore -p "MyProject,OtherProject" -v

# Step 2: Restore build definitions
adobackup.exe build-restore -p "MyProject" -v

# Build definition will now find OtherProject/SharedRepo
```

## Dependency Matrix

Quick reference table for resource dependencies:

| Resource | Depends On | Depended On By | Restore Priority |
|----------|-----------|----------------|------------------|
| **Git Repositories** | None | Pull Requests, Builds, Work Items | **1 - FIRST** |
| **Pull Requests** | Git Repositories | None | **2 - SECOND** |
| **Service Connections** | None | Builds (optional) | **3 - THIRD** |
| **Variable Groups** | None | Builds (optional) | **4 - FOURTH** |
| **Build Definitions** | Git, Service Connections, Variables | Work Items | **5 - FIFTH** |
| **Areas & Iterations** | None | Work Items ⚠️, Queries | **6 - SIXTH** |
| **Work Items** | Areas & Iterations ⚠️, Git (optional), Builds (optional) | Queries | **7 - SEVENTH** |
| **Shared Queries** | Work Items, Areas & Iterations | None | **8 - LAST** |

## Advanced Scenarios

### Scenario 1: Restore Only Builds and Their Dependencies

```bash
# Restore builds with their dependencies
adobackup.exe restore-all \
  --include-git \
  --include-serviceconnections \
  --include-variables \
  --include-builds \
  -v

# This restores:
# 1. Git Repositories (dependency of builds)
# 2. Service Connections (dependency of builds)
# 3. Variable Groups (dependency of builds)  
# 4. Build Definitions (requested)
```

### Scenario 2: Restore to Empty Project

```bash
# Create target project in Azure DevOps first
# Then restore everything in order:

adobackup.exe restore-all \
  -p "SourceProject" \
  --target-project "EmptyProject" \
  -v

# Automatic order:
# 1. Git → 2. Pull Requests → 3. Service Connections → 4. Variables → 5. Builds → 6. Areas & Iterations → 7. Work Items → 8. Queries
```

### Scenario 3: Incremental Restore (Add Resources)

```bash
# Already restored Git and Builds, now adding Work Items
# Dependencies already satisfied, safe to restore work items:

adobackup.exe workitems-restore \
  -p "ProjectA" \
  -v

# Then add queries
adobackup.exe queries-restore \
  -p "ProjectA" \
  -v
```

### Scenario 4: Fix Broken Dependencies

```bash
# If you restored in wrong order and have broken references:

# Option 1: Delete and re-restore in correct order
# 1. Delete builds from Azure DevOps
# 2. Restore Git first: adobackup.exe git-restore -p "Project" -v
# 3. Restore builds: adobackup.exe build-restore -p "Project" -v

# Option 2: Use --target-project to restore to clean project
adobackup.exe restore-all \
  -p "SourceProject" \
  --target-project "CleanProject" \
  -v
```

## Summary

### Key Takeaways

1. **Always restore Git repositories first** - they are the foundation
2. **Restore pull requests after Git** - PRs belong to repositories
3. **Restore service connections before builds** - builds may reference them (secrets must be re-entered)
4. **Restore variables before builds** - builds may reference them
5. **Restore work items before queries** - queries search work items
6. **Use `restore-all` for automatic ordering** - it handles dependencies
7. **Use `--dry-run` to preview** - catches dependency issues early
8. **Respect cross-project dependencies** - restore all referenced projects
9. **Manual fixes required** for cross-project area/iteration references in queries
10. **Queries are updated if they exist** - existing queries are overwritten with backup data

### Quick Reference Commands

```bash
# ✅ CORRECT: Full restore (automatic ordering)
adobackup.exe restore-all -v

# ✅ CORRECT: Selective with dependencies
adobackup.exe restore-all --include-git --include-serviceconnections --include-variables --include-builds -v

# ❌ WRONG: Builds without Git
adobackup.exe restore-all --include-builds -v

# ✅ CORRECT: Step-by-step respecting order
adobackup.exe git-restore -p "Project" -v
adobackup.exe pullrequests-restore -p "Project" -v
adobackup.exe serviceconnections-restore -p "Project" -v
adobackup.exe variables-restore -p "Project" -v
adobackup.exe build-restore -p "Project" -v
adobackup.exe workitems-restore -p "Project" -v
adobackup.exe queries-restore -p "Project" -v
```

---

**See Also:**
- [Pipeline Integration Guide](./pipeline-integration.md) - Task configuration and examples
- [Best Practices](./best-practices.md) - Backup and restore strategies
- [Troubleshooting Guide](./troubleshooting.md) - Common issues and solutions
- [Use Cases](./use-cases.md) - Real-world scenarios

---

**Last Updated:** January 2025
