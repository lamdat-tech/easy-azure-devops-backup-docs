# Task Group ID Remapping in Build Definitions

## Overview

When restoring build definitions to a different Azure DevOps organization (or even within the same organization), task groups referenced in build definitions may fail because the task group IDs don't match. Task groups get new IDs when restored, but build definitions still reference the old IDs from the source organization.

This document describes the automatic task group ID remapping functionality that resolves this issue.

## Problem

When performing a cross-organization restore:

1. **Task groups are backed up** from the source organization with ID `a3b1c37b-c77f-47a3-89f2-818ed8a70546`
2. **Task groups are restored** to the target organization and get a **new ID** `c5d2e38a-d88g-58b4-aa3-929fe9b81657`
3. **Build definitions are restored** but still reference the **old task group ID**
4. **Build restore fails** with error: `Task group a3b1c37b-c77f-47a3-89f2-818ed8a70546 not found`

## Solution

Build definition restore automatically detects task group references and remaps them based on task group names:

### Process Flow

1. **Fetch task groups** from the target project (with caching to avoid redundant API calls)
2. **Scan build definition JSON** to find all task references with task group IDs
3. **Look up task group names** from backup files based on the old IDs - the backup stores each task group's name alongside its ID, so the original name can always be recovered from an old ID
4. **Find matching task groups** in the target project by name
5. **Replace old IDs with new IDs** in the build definition JSON
6. **Create/update the build definition** with remapped IDs

## Usage

The task group remapping happens **automatically** during build definition restore. No special configuration is required.

### Example Command

```bash
adobackup.exe restore-all --include-builds \
    --BackupRoot "C:\Backups\AzureDevOps" \
    --OrganizationUrl "https://dev.azure.com/SourceOrg" \
    --Pat "xxx" \
    -p "SourceProject" \
    --target-org "https://dev.azure.com/TargetOrg" \
    --target-pat "xxx" \
    --target-project "TargetProject"
```

### Log Output

When task groups are remapped, you'll see log messages like:

```
Found task group name 'Deploy to Production' for ID a3b1c37b-c77f-47a3-89f2-818ed8a70546 in backup
Will remap task group 'Deploy to Production': a3b1c37b-c77f-47a3-89f2-818ed8a70546 -> c5d2e38a-d88g-58b4-aa3-929fe9b81657
Found 3 task group reference(s) to remap
Remapped task group ID: a3b1c37b-c77f-47a3-89f2-818ed8a70546 -> c5d2e38a-d88g-58b4-aa3-929fe9b81657
```

## Prerequisites

For task group remapping to work:

1. **Task groups must be backed up** - The backup must include task groups in the `TaskGroups` folder
2. **Task groups must be restored first** - Run task group restore before build restore, or use `restore-all --include-taskgroups --include-builds`
3. **Task group names must match** - The task group names in the target organization must match those in the backup (case-insensitive)

## Handling Missing Task Groups

If a task group referenced by a build definition is not found in the target organization:

- A **warning** is logged: `Task group '{Name}' (ID: {OldId}) not found in target project`
- The **old ID is preserved** in the build definition
- The **build restore will likely fail** when Azure DevOps validates the definition
- **Solution**: Restore the missing task group or remove the reference from the build definition

## Performance

Task group remapping is optimized for performance:

- **Cached API calls**: Task groups are fetched once per project and cached
- **Batch processing**: All task groups are processed in a single scan
- **Parallel processing**: Build definitions are still restored in parallel

## Troubleshooting

### Issue: Task group ID not remapped

**Possible causes:**
1. Task group not backed up
2. Task group not restored to target organization
3. Task group name mismatch
4. Backup file corrupted or missing

**Solution:**
1. Check that task groups exist in `BackupRoot/TaskGroups/{ProjectName}/`
2. Verify task groups exist in target organization with matching names
3. Check logs for warnings about missing task groups

### Issue: Build definition still fails with "Task group not found"

**Possible causes:**
1. Task group not restored yet
2. Task group name doesn't match
3. Task group in different project (cross-project reference)

**Solution:**
1. Restore task groups first: `adobackup.exe taskgroups-restore -p "Project"`
2. Verify task group names match (case-insensitive)
3. Check if task group was in a different project in the source organization

## Future Enhancements

Potential improvements:

1. **Cross-project task group references** - Support for task groups in different projects
2. **Task group version matching** - Match by version in addition to name
3. **Automatic task group restore** - Automatically restore missing task groups
4. **Detailed mapping report** - Export task group ID mappings to a report file
