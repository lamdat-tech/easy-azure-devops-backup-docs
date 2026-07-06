# Azure DevOps API Limitations - Boards, Dashboards & Wiki

This document provides detailed technical information about Azure DevOps API limitations discovered during testing and implementation of the backup and restore utility.

## 📋 Overview

Most limitations stem from **Azure DevOps REST API constraints**, not the backup/restore utility itself. The utility successfully backs up all data, but restore operations may encounter API restrictions when trying to recreate certain configurations.

---

## 🎯 Work Items - Detailed Limitations

### ✅ What Restores Successfully

| Feature | Support Level | Notes |
|---------|---------------|-------|
| **Work item fields** | ✅ Full support | All standard and custom fields |
| **Original author (`System.CreatedBy`)** | ✅ With `--bypass-rules true` (default) | Identity must exist in target org |
| **Creation date (`System.CreatedDate`)** | ✅ With `--bypass-rules true` (default) | Written on work item creation |
| **Revision editor (`System.ChangedBy`)** | ✅ With `--bypass-rules true` (default) | Written per revision during history replay |
| **Revision date (`System.ChangedDate`)** | ✅ With `--bypass-rules true` (default) | Written per revision during history replay |
| **Comment text** | ✅ Full support | Original text fully preserved |
| **Comment attribution** | ✅ Markdown attribution block | Original author and date prepended; see below |
| **Revision history** | ✅ Full replay | Each revision replayed as a sequential update in original order |
| **Attachments** | ✅ Full support | Files re-uploaded and linked |
| **Relations (parent/child/related)** | ✅ Full support | IDs rewritten for cross-org restore |
| **Area and iteration paths** | ✅ Full support | Paths validated against target; fallback to project root if not found |

### ⚠️ What Cannot Be Fully Restored

| Feature | Limitation | Reason | Impact |
|---------|-----------|--------|--------|
| **Comment author** | Attribution via markdown block only | ADO Comments API (`/_apis/wit/workItems/{id}/comments`) does not accept a `createdBy` parameter | Comments show restore service account; original author shown in prepended block |
| **Comment timestamp** | Not preserved | Comments API always uses current timestamp | Comment dates reflect restore time; original date shown in attribution block |
| **Identities not in target org** | Author fields silently skipped | ADO rejects `System.CreatedBy`/`System.ChangedBy` values for identities that don't exist in the target org | Affected items/revisions fall back to restore service account identity; all other fields restore correctly |
| **`System.BoardColumn` / `System.BoardColumnDone`** | Not restored | Read-only Kanban board fields managed by ADO | Board column placement reflects target board state |
| **`WEF_*` Kanban fields** | Not restored | Extension-managed read-only fields | Board extension state resets to default |
| **Source numeric area/iteration IDs** | Intentionally excluded | `System.AreaId`, `System.IterationId`, `System.AreaLevel*`, `System.IterationLevel*` are org-specific; sending them causes `TF51541: The Area/Iteration ID is not recognized` | Area/iteration resolved by path name from `System.AreaPath` / `System.IterationPath` instead |

### 📝 Comment Attribution Format

Because the ADO Comments API does not support setting the comment author, all restored comments are prefixed with a markdown attribution block:

```markdown
> 🗣️ Originally by **John Doe** (john@contoso.com) — Jan 15, 2023

---

[original comment text]
```

This ensures original authorship and date are visible, searchable, and auditable in the restored work item.

### 🔧 Identity Resolution

When `--bypass-rules true` (default), author fields are sent to ADO during restore. If the identity does not exist in the target org, ADO returns a validation error for that field only. The restore service catches this per-field and continues restoring the remaining fields. To see which identities failed, check the restore log for messages like:

```
⚠️ Failed to apply revision 3 to work item 42: ...identity not found...
```

**Resolution:** Ensure all user accounts exist in the target Azure DevOps organization before running work item restore, or set `--bypass-rules false` to skip authorship entirely.

---

## 🎯 Boards - Detailed Limitations

### ✅ What Restores Successfully

| Feature | Support Level | Notes |
|---------|---------------|-------|
| **Standard columns** | ✅ Full support | New, Active, Resolved, Closed |
| **Board name** | ✅ Full support | Board name preserved |
| **Swimlanes (rows)** | ✅ Full support | Row names, colors preserved |
| **Board settings** | ✅ Full support | Basic settings preserved |

### ⚠️ What May Not Restore Fully

| Feature | Issue | API Error | Impact |
|---------|-------|-----------|--------|
| **Custom columns** | API rejects certain column configurations | `400 Bad Request: "Value cannot be null. Parameter name: options"` | Custom intermediate columns (e.g., "Elaboration") may be missing after restore |
| **Column state mappings** | Complex state mappings not supported | `400 Bad Request` | Columns that map to custom work item states may fail |
| **WIP limits** | Column WIP limits may revert | Silent failure | WIP limits may need manual reconfiguration |
| **Split columns** | "Doing" / "Done" split columns | API limitation | Split column configuration may not be preserved |
| **Column order** | Custom column ordering | May revert to default | Column order may not match backup |
| **Card styling** | Custom card colors/styles | API doesn't support | Card appearance may revert to defaults (cosmetic only) |
| **Board creation** | Boards cannot be auto-created | API limitation | Target board must already exist in project |

### 🔬 Technical Details - Custom Column Issue

**Example scenario from E2E tests:**

```
Backup columns (5):  New, Elaboration, Active, Resolved, Closed
Restored columns (4): New, Active, Resolved, Closed
Missing: Elaboration
```

**Root cause:**
- Azure DevOps Boards API (v7.1) validates column configurations against the project's **process template**
- Custom columns that don't map to standard work item states may be rejected
- The API returns: `"Value cannot be null. Parameter name: options"` when encountering unsupported column configurations

**Workaround:**
1. Backup completes successfully (all data captured)
2. Restore attempts full board update
3. If API rejects, utility falls back to partial restore (rows/swimlanes only)
4. User manually configures custom columns after restore

**Code implementation:**
```csharp
try
{
    // Try full board restore
    targetBoard.Columns = boardData.Columns;
    targetBoard.Rows = boardData.Rows;
    await _boardsClient.UpdateBoardAsync(...);
}
catch (HttpRequestException ex) when (ex.Message.Contains("400"))
{
    // Fallback: Restore rows only
    targetBoard.Rows = boardData.Rows;
    await _boardsClient.UpdateBoardAsync(...);
    _logger.Warning("Custom columns could not be restored due to API limitations");
}
```

### 📊 Test Results

**Test:** `BackupSpecificBoard_RestoreToDifferentProject_StoriesBoardCrossProjectRestore`

- ✅ Backup: 5 columns captured
- ⚠️ Restore: 4 columns restored
- 🔍 Deep verification: Detected missing "Elaboration" column
- ✅ Test: Passed (lenient - warns about differences but doesn't fail)

---

## 📊 Dashboards - Detailed Limitations

### ✅ What Restores Successfully

| Feature | Support Level | Notes |
|---------|---------------|-------|
| **Dashboard name** | ✅ Full support | Dashboard name preserved |
| **Simple widgets** | ✅ Full support | Query results, sprint burndown, velocity |
| **Widget layout** | ✅ Full support | Widget positions preserved |
| **Dashboard permissions** | ✅ Full support | Team visibility preserved |

### ⚠️ What May Not Restore Fully

| Feature | Issue | Impact |
|---------|-------|--------|
| **Complex widgets** | Some widget types have limited API support | Widget may be missing or revert to defaults |
| **Widget configurations** | Custom widget settings may not restore | Widget exists but settings may be lost |
| **Dashboard creation** | Dashboards cannot be auto-created | Target dashboard must already exist |
| **Widget counts** | Some widgets may not restore | Dashboard structure preserved but widget count may differ |

### 🔬 Technical Details

**Widget restoration:**
- Azure DevOps Dashboards API (v7.1) supports widget creation but has limitations on certain widget types
- Complex widgets with custom data sources or configurations may fail silently
- The utility logs widget count discrepancies for verification

**Deep verification:**
```
Backup widgets: 5 widget(s)
Restored widgets: 5 widget(s)
✅ Widget count matches
```

### 📊 Test Results

**Test:** `CompleteDashboardsWorkflow_BackupRestoreVerifyCleanup_AllDashboards`

- ✅ Backup: All dashboards captured
- ✅ Restore: Dashboards updated
- 🔍 Deep verification: Widget counts compared
- ✅ Test: 8/8 PASS

---

## 📚 Wiki - Detailed Limitations

### ✅ What Restores Successfully

| Feature | Support Level | Notes |
|---------|---------------|-------|
| **Wiki pages** | ✅ Full support | All page content preserved |
| **Page hierarchy** | ✅ Full support | Page paths and structure maintained |
| **Page content** | ✅ Full support | Markdown content preserved |
| **Page metadata** | ✅ Full support | Page titles, paths preserved |

### ⚠️ What May Not Restore Fully

| Feature | Issue | Impact |
|---------|-------|--------|
| **Empty wikis** | Empty wikis restore as empty (expected) | Wiki structure exists but contains no pages |
| **Attachments** | Images/attachments embedded in pages | May need separate handling |
| **Page revisions** | Historical page versions | Only latest version restored |

### 🔬 Technical Details

**Wiki restoration:**
- Azure DevOps Wiki API (v7.1) supports full page restoration
- Empty wikis (0 pages) remain empty after restore (expected behavior)
- Page paths are preserved exactly

**Deep verification:**
```
Backup pages: 0
Restored pages: 0
✅ Empty wiki (0 pages) - matches backup
```

### 📊 Test Results

**Test:** `BackupWiki_VerifyFiles_RestoreWiki_VerifySuccess`

- ✅ Backup: All wiki pages captured
- ✅ Restore: Wiki pages restored
- 🔍 Deep verification: Page counts and paths compared
- ✅ Test: 5/5 PASS

---

## 🛠️ Mitigation Strategies

### For Boards

**Option 1: Manual Configuration (Recommended)**
1. Run backup (captures all configuration)
2. Run restore (restores what API supports)
3. Use backup JSON as reference
4. Manually configure custom columns that failed

**Option 2: Process Template Alignment**
1. Ensure target project uses same process template as source
2. Custom columns that exist in process template are more likely to restore
3. Verify work item states match between projects

**Option 3: Documentation**
1. Document custom column configurations separately
2. Include column names, state mappings, WIP limits
3. Use as checklist after restore

### For Dashboards

**Option 1: Manual Widget Reconfiguration**
1. Run backup (captures all widgets)
2. Run restore (restores supported widgets)
3. Use backup JSON to identify missing widgets
4. Manually add missing widgets after restore

**Option 2: Widget Screenshots**
1. Take screenshots of dashboards before backup
2. Use as visual reference for post-restore configuration

### For Wiki

**Option 1: Verify Page Restoration**
1. Run backup (captures all pages)
2. Run restore (restores all pages)
3. Use deep verification to confirm page counts match
4. No manual steps typically needed

---

## 🔍 Deep Verification Features

The E2E tests include **deep verification** that compares backed-up data with restored data by querying Azure DevOps after restore.

### Boards Deep Verification
```csharp
// Compares backup JSON with Azure DevOps API response
var backupColumns = ["New", "Elaboration", "Active", "Resolved", "Closed"];
var restoredColumns = ["New", "Active", "Resolved", "Closed"];

// Detects differences
⚠️ Columns missing in restored board: Elaboration
⚠️ Column count mismatch - Azure DevOps API may not support all board configurations
```

### Dashboards Deep Verification
```csharp
// Compares widget counts
Backup widgets: 5 widget(s)
Restored widgets: 5 widget(s)
✅ Widget count matches
```

### Wiki Deep Verification
```csharp
// Compares page counts and paths
Backup pages: 0
Restored pages: 0
✅ Empty wiki (0 pages) - matches backup

// For non-empty wikis
Backup pages: 12
Restored pages: 12
✅ Page count matches
✅ All page paths match
```

---

## 📝 Reporting Issues

If you encounter limitations not documented here:

1. **Capture logs** with `--verbose` flag
2. **Note the API error** (if any) from Azure DevOps
3. **Document the configuration** that failed to restore
4. **Submit an issue** with:
   - Component (Boards/Dashboards/Wiki)
   - Expected behavior vs actual behavior
   - Backup JSON snippet (if applicable)
   - Azure DevOps API version
   - Project process template (for Boards issues)

**Contact:** support@lamdat.com

---

## 🔗 Related Documentation

- [Getting Started](./getting-started.md) - Quick start guide
- [Pipeline Integration Guide](./pipeline-integration.md) - Task configuration and examples
- [Troubleshooting](./troubleshooting.md) - Common issues
- [FAQ](./faq.md) - Frequently asked questions
- [Best Practices](./best-practices.md) - Optimization tips

---

## 📅 Last Updated

This document reflects testing results as of **February 2026** against:
- Azure DevOps API version: **7.1**
- Utility version: **Latest**
- Test coverage: **22 E2E tests** (9 Boards, 8 Dashboards, 5 Wiki)
