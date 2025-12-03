# Work Item History Backup Feature

## Overview

The work item backup functionality now includes automatic backup of work item revision history (updates). This feature captures the complete change history for each work item, providing a comprehensive audit trail.

## What is Backed Up

When backing up work items, the system now saves:

1. **workitem.json** - The current state of the work item with all fields
2. **comments.json** - All comments/discussions on the work item
3. **history.json** - Complete revision history showing all changes made to the work item (NEW)
4. **attachments/** - All file attachments
5. **metadata.json** - Backup metadata

## History Format

The `history.json` file contains the complete revision history from Azure DevOps, including:

- All field changes with before/after values
- Timestamp of each change
- User who made the change
- Revision numbers
- Full change details

Example structure:
```json
{
  "count": 5,
  "value": [
    {
      "id": 1,
      "rev": 1,
      "fields": {
        "System.Title": {
          "oldValue": null,
          "newValue": "Initial title"
        },
        "System.State": {
          "oldValue": null,
          "newValue": "New"
        }
      },
      "revisedDate": "2024-01-15T10:30:00Z",
      "revisedBy": {
        "displayName": "John Doe"
      }
    }
  ]
}
```

## Incremental Backup Support

Work item history is fully supported in incremental backup mode:

- When using `--incremental` flag, changed work items are identified by both:
  - New work item IDs (from last max ID + 1)
  - Work items modified since last backup date
  
- The complete history for each changed work item is backed up
- History is always backed up completely (not incrementally) to maintain integrity

## Usage

### Standard Backup (includes history)
```bash
# Backup all work items including history
adobackuprestore workitems-backup

# Backup specific project
adobackuprestore workitems-backup --projects "MyProject"
```

### Incremental Backup (includes history for changed items)
```bash
# Incremental backup - includes history for new/changed work items
adobackuprestore workitems-backup --incremental

# With specific project
adobackuprestore workitems-backup --projects "MyProject" --incremental
```

### Backup All (includes history)
```bash
# Full organization backup including work item history
adobackuprestore backup-all

# Incremental backup of everything
adobackuprestore backup-all --incremental
```

## Restore Behavior

**Important**: Work item history is **READ-ONLY** in Azure DevOps and cannot be restored via the API.

### What Happens During Restore:

1. ? Work item fields are restored
2. ? Comments are restored (only for newly created work items)
3. ? Attachments are restored
4. ? Relations/links are restored
5. ? History is **NOT** restored (read-only in Azure DevOps)

### Why History Cannot be Restored:

- Azure DevOps automatically generates history entries when work items are created or modified
- The history/updates API endpoint is read-only
- There is no API to create or modify historical revisions
- This is a security and audit feature of Azure DevOps

### History File Purpose:

The `history.json` file serves as:
- **Audit trail** - Complete record of all changes
- **Reference documentation** - What changed and when
- **Compliance** - Historical record for regulatory requirements
- **Analysis** - Pattern analysis, velocity tracking, etc.

When work items are restored:
- New history entries will be automatically created by Azure DevOps
- The restored work item will have a fresh history starting from the restore point
- The original `history.json` backup remains available for reference

## API Details

### New API Method

**IAzureDevOpsClient.GetWorkItemUpdatesAsync(int workItemId)**
- Retrieves complete revision history for a work item
- Uses Azure DevOps REST API: `_apis/wit/workitems/{id}/updates?api-version=7.1`
- Returns JSON containing all revisions with field changes
- Includes rate limiting and retry logic

## Storage Structure

Work item backups are organized by ID ranges:

```
backups/
??? workitems/
    ??? ProjectName/
        ??? 0000000-0009999/  (range folder)
            ??? 12345/        (work item ID)
                ??? workitem.json
                ??? comments.json
                ??? history.json      ? NEW
                ??? metadata.json
                ??? attachments/
                    ??? ...
```

## Performance Considerations

- History backup adds one additional API call per work item
- API calls include rate limiting protection
- History files are typically small (< 100KB for most work items)
- Parallel processing is maintained for optimal performance

## Troubleshooting

### History Not Being Backed Up

If you notice history.json is missing:

1. **Check API permissions** - Ensure PAT has `vso.work_write` scope
2. **Check work item age** - Very new work items may have minimal history
3. **Check logs** - Look for warnings about history backup failures
4. **API limitations** - Some work item types may have limited history access

### Build or compilation errors

If you see errors related to history:
- Ensure you're using the latest version
- Check that all dependencies are up to date
- Verify Azure DevOps API version compatibility

## Examples

### Viewing Backed Up History

```powershell
# View history for a specific work item
$historyPath = "backups/workitems/MyProject/0000000-0009999/12345/history.json"
Get-Content $historyPath | ConvertFrom-Json | ConvertTo-Json -Depth 10
```

### Analyzing Change Patterns

```powershell
# Count revisions for a work item
$history = Get-Content "backups/workitems/MyProject/0000000-0009999/12345/history.json" | ConvertFrom-Json
Write-Host "Total revisions: $($history.count)"
```

## License Considerations

Work item history backup is included in the standard licensing:
- Each work item with history counts as one work item toward your license limit
- History data does not count separately
- Incremental backups maintain the same license counting

## Future Enhancements

Potential future features (not currently implemented):
- History comparison between backups
- Change velocity reporting
- History export to different formats
- History-based change tracking

## Related Documentation

- [Command Reference](command-reference.md) - All command options
- [Getting Started](getting-started.md) - Initial setup
- [Best Practices](best-practices.md) - Backup strategies
- [Use Cases](use-cases.md) - Common scenarios
