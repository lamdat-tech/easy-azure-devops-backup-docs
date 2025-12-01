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
???????????????????????????????????????????????????????????????
?                      RESTORE ORDER                           ?
?                    (Top to Bottom)                           ?
???????????????????????????????????????????????????????????????

1. Git Repositories
   ??? No dependencies
       (Always safe to restore first)

2. Pipeline Variables / Variable Groups
   ??? No resource dependencies
       (Can restore anytime, typically after Git)

3. Build Definitions
   ??? Depends on: Git Repositories (for repository references)
   ??? Depends on: Variable Groups (optional, for pipeline variables)

4. Work Items
   ??? Depends on: Git Repositories (optional, for commit links)
   ??? Depends on: Build Definitions (optional, for build links)

5. Shared Queries
   ??? Depends on: Work Items (queries search for work items)
   ??? Depends on: Areas (for area path filters)
   ??? Depends on: Iterations (for iteration path filters)
```

## Backup Order

The backup order is more flexible than restore order because you're just capturing data, not creating references. However, `adobackup` follows this recommended order:

### Recommended Backup Sequence

```bash
# Order doesn't affect backup functionality, but this is the logical flow:

1. Git Repositories       # Foundation data
2. Pipeline Variables     # Configuration data
3. Build Definitions      # References Git and Variables
4. Build History          # Historical data
5. Work Items            # Can reference any resource
6. Shared Queries         # Queries against work items
```

### Backup Command Execution Order

When you run `adobackup.exe backup-all`, resources are backed up in this order:

```csharp
// From BackupAllCommand.cs internal logic:
1. Git Repositories      (parallel per repository)
2. Build Definitions     (parallel per definition)
3. Build History         (parallel per build)
4. Pipeline Variables    (parallel per group)
5. Work Items           (parallel in batches)
6. Shared Queries        (parallel per query)
```

**Note:** Backup order doesn't create dependencies - you can backup any resource independently.

## Restore Order

The restore order is **critical** because creating dependent resources before their dependencies will fail or create broken references.

### Mandatory Restore Sequence

```bash
# MUST restore in this order for successful operation:

1. ? Git Repositories FIRST
   ??? Creates repositories that build definitions will reference

2. ? Pipeline Variables SECOND  
   ??? Creates variable groups that pipelines may reference

3. ? Build Definitions THIRD
   ??? Now can reference existing Git repos and variables

4. ? Work Items FOURTH
   ??? Can link to commits (Git) and builds

5. ? Shared Queries LAST
   ??? Queries search for work items (must exist first)
```

### Restore Command Execution Order

When you run `adobackup.exe restore-all`, resources are restored in this order:

```csharp
// From RestoreAllCommand.cs execution order:
1. Git Repositories      (RestoreGitRepositoriesAsync)
2. Pipeline Variables    (RestorePipelineVariablesAsync)
3. Build Definitions     (RestoreBuildDefinitionsAsync)
4. Work Items           (RestoreWorkItemsAsync)
5. Shared Queries        (RestoreQueriesAsync)
```

**The utility enforces this order automatically** - you cannot change it when using `restore-all`.

### Selective Restore Order

If you use skip flags or individual restore commands, **you must respect dependencies**:

#### ? SAFE Selective Restores

```bash
# Restore only Git (no dependencies)
adobackup.exe restore-all --skip-builds --skip-workitems --skip-variables --skip-queries

# Restore Git + Variables (no dependency between them)
adobackup.exe restore-all --skip-builds --skip-workitems --skip-queries

# Restore everything except queries (queries depend on work items)
adobackup.exe restore-all --skip-queries
```

#### ? UNSAFE Selective Restores

```bash
# BAD: Restore builds without Git repos
# Will fail if build definitions reference repositories that don't exist
adobackup.exe restore-all --skip-git --skip-workitems --skip-variables --skip-queries

# BAD: Restore queries without work items
# Queries will be created but won't find any work items to query
adobackup.exe restore-all --skip-git --skip-builds --skip-workitems --skip-variables
```

## Resource Dependencies Explained

### 1. Git Repositories

**Dependencies:** None

**Depended On By:**
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

### 2. Pipeline Variables / Variable Groups

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

### 3. Build Definitions

**Dependencies:**
- Git Repositories (for repository references)
- Variable Groups (optional, for pipeline variables)

**Depended On By:**
- Work Items (builds can be linked to work items)
- Build History (historical build runs)

**Why Restore After Git/Variables:**
- Build definitions contain repository IDs
- YAML pipelines are stored in repositories (must exist)
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

### 4. Work Items

**Dependencies:**
- Git Repositories (optional, for commit links)
- Build Definitions (optional, for build links)

**Depended On By:**
- Shared Queries (queries search work items)

**Why Restore After Git/Builds:**
- Work items can link to Git commits (requires repository to exist)
- Work items can link to builds (requires build definition to exist)
- Links are preserved even if dependencies missing (but may be broken)

**Important Behaviors:**
- Existing work items are **updated** (not skipped)
- Deleted work items are **recreated with new IDs**
- No comments added to work item history about restore
- `--bypass-rules` skips field validation (use for different process templates)

**Cross-Project Limitations:**
- Work items can **only** be restored to the same project
- Use `--target-project` to clone to different project (creates new IDs)

### 5. Shared Queries

**Dependencies:**
- Work Items (queries search for work items)
- Areas (queries filter by area paths)
- Iterations (queries filter by iteration paths)

**Depended On By:** None

**Why Restore Last:**
- Queries are WIQL (Work Item Query Language) that search work items
- Query will be created but return no results if work items don't exist
- Area and iteration references are NOT updated in cross-project restore

**Important Limitations:**
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

### ? DO: Follow the Recommended Order

```bash
# Full restore - automatic correct order
adobackup.exe restore-all -v

# Selective restore - respect dependencies
# Step 1: Git first
adobackup.exe git-restore -p "ProjectA" -v

# Step 2: Then builds
adobackup.exe build-restore -p "ProjectA" -v
```

### ? DO: Use Dry-Run to Verify Dependencies

```bash
# Preview restore to check for dependency issues
adobackup.exe restore-all --dry-run -v

# Look for warnings about missing dependencies in output
```

### ? DO: Restore Complete Dependency Chains

```bash
# If restoring builds, also restore Git repos
adobackup.exe restore-all --skip-workitems --skip-queries -v
# ? Includes: Git + Variables + Builds (complete chain)
```

### ? DON'T: Restore Without Dependencies

```bash
# BAD: Builds without repos
adobackup.exe restore-all --skip-git -v
# ? Build definitions will reference non-existent repositories

# BAD: Queries without work items  
adobackup.exe queries-restore -v
# ? Queries will be created but find nothing
```

### ? DON'T: Assume Broken References Fix Themselves

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

### Issue 4: Work Item Links Are Broken After Restore

**Symptom:** Work items restored successfully but links to commits/builds are broken.

**Causes:**
- Git repositories or builds not restored yet
- Repository/build IDs changed during restore
- Cross-project restore (work items get new IDs)

**Solution:**

**For Same-Project Restore:**
```bash
# Restore in correct order
adobackup.exe restore-all -v  # Automatic correct order

# Links should be preserved with original IDs
```

**For Cross-Project Restore:**
```bash
# Links will be broken because work items get new IDs
# Manual remediation required:
# 1. Document important links before restore
# 2. Restore all resources
# 3. Manually recreate critical links in Azure DevOps
```

### Issue 5: Cross-Project Build Definition Fails

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
| **Git Repositories** | None | Builds, Work Items | **1 - FIRST** |
| **Variable Groups** | None | Builds (optional) | **2 - SECOND** |
| **Build Definitions** | Git, Variables | Work Items | **3 - THIRD** |
| **Work Items** | Git (optional), Builds (optional) | Queries | **4 - FOURTH** |
| **Shared Queries** | Work Items, Areas, Iterations | None | **5 - LAST** |

## Advanced Scenarios

### Scenario 1: Restore Only Builds and Their Dependencies

```bash
# Automatic dependency resolution
adobackup.exe restore-all \
  --skip-workitems \
  --skip-queries \
  -v

# This restores:
# 1. Git Repositories (dependency of builds)
# 2. Variable Groups (dependency of builds)  
# 3. Build Definitions (requested)
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
# 1. Git ? 2. Variables ? 3. Builds ? 4. Work Items ? 5. Queries
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
2. **Restore variables before builds** - builds may reference them
3. **Restore work items before queries** - queries search work items
4. **Use `restore-all` for automatic ordering** - it handles dependencies
5. **Use `--dry-run` to preview** - catches dependency issues early
6. **Respect cross-project dependencies** - restore all referenced projects
7. **Manual fixes required** for cross-project area/iteration references in queries

### Quick Reference Commands

```bash
# ? CORRECT: Full restore (automatic ordering)
adobackup.exe restore-all -v

# ? CORRECT: Selective with dependencies
adobackup.exe restore-all --skip-workitems --skip-queries -v

# ? WRONG: Builds without Git
adobackup.exe restore-all --skip-git -v

# ? CORRECT: Step-by-step respecting order
adobackup.exe git-restore -p "Project" -v
adobackup.exe variables-restore -p "Project" -v
adobackup.exe build-restore -p "Project" -v
adobackup.exe workitems-restore -p "Project" -v
adobackup.exe queries-restore -p "Project" -v
```

---

**See Also:**
- [Command Reference](./command-reference.md) - Detailed command documentation
- [Best Practices](./best-practices.md) - Backup and restore strategies
- [Troubleshooting Guide](./troubleshooting.md) - Common issues and solutions
- [Use Cases](./use-cases.md) - Real-world scenarios

---

**Last Updated:** January 2025
