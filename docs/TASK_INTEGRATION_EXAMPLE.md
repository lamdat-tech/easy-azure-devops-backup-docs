# Example: Adding License Check to BuildBackupTask

This example shows how to add license validation to the BuildBackupTask.

## Before (Original Code)

```typescript
import tl = require('azure-pipelines-task-lib/task');

async function run() {
    try {
        tl.setResourcePath(path.join(__dirname, 'task.json'));

        // Get input parameters
        const projects = tl.getInput('Projects');
        const definitions = tl.getInput('Definitions');
        
        // ... rest of task logic
        
        tl.setResult(tl.TaskResult.Succeeded, 'Backup completed successfully');
    } catch (err: any) {
        tl.setResult(tl.TaskResult.Failed, err.message);
    }
}

run();
```

## After (With License Check)

```typescript
import tl = require('azure-pipelines-task-lib/task');
import { requireLicense, getLicenseKey } from '../shared/licenseHelper';

async function run() {
    try {
        tl.setResourcePath(path.join(__dirname, 'task.json'));

        // OPTION 1: Require license (task fails if not found)
        console.log('Validating license...');
        const licenseKey = await requireLicense();
        console.log('License validated successfully');

        // Get input parameters
        const projects = tl.getInput('Projects');
        const definitions = tl.getInput('Definitions');
        
        // ... rest of task logic
        
        tl.setResult(tl.TaskResult.Succeeded, 'Backup completed successfully');
    } catch (err: any) {
        tl.setResult(tl.TaskResult.Failed, err.message);
    }
}

run();
```

## Alternative: Optional License (Graceful Degradation)

```typescript
import tl = require('azure-pipelines-task-lib/task');
import { getLicenseKey, hasLicense } from '../shared/licenseHelper';

async function run() {
    try {
        tl.setResourcePath(path.join(__dirname, 'task.json'));

        // OPTION 2: Check for license but continue without it
        let isPremium = false;
        if (await hasLicense()) {
            const license = await getLicenseKey();
            console.log('Running with licensed features enabled');
            isPremium = true;
        } else {
            console.log('Running in free mode - some features limited');
        }

        // Get input parameters
        const projects = tl.getInput('Projects');
        const definitions = tl.getInput('Definitions');
        
        // Conditional logic based on license
        if (isPremium) {
            // Enable all features
            console.log('Processing all projects...');
        } else {
            // Limit features
            console.log('Processing up to 5 projects (upgrade for unlimited)...');
        }
        
        // ... rest of task logic
        
        tl.setResult(tl.TaskResult.Succeeded, 'Backup completed successfully');
    } catch (err: any) {
        tl.setResult(tl.TaskResult.Failed, err.message);
    }
}

run();
```

## Pipeline YAML Configuration

Remember to enable OAuth token access:

```yaml
steps:
- task: BuildBackupTask@1
  displayName: 'Backup Build Definitions'
  inputs:
    Projects: 'MyProject'
    Definitions: 'Build1,Build2'
    IncludeHistory: true
  env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

## Error Messages

### When License is Required but Missing:

```
##[error]No license key found. Please configure the license in Project Settings > Backup & Restore License
```

### When OAuth Token is Missing:

```
##[warning]System.AccessToken not available. Make sure to enable "Allow scripts to access the OAuth token" in pipeline settings.
```

## Testing the Integration

1. **Without License**:
   - Run the task without configuring a license
   - Verify appropriate error/warning messages appear
   - Ensure task fails gracefully (if using `requireLicense()`)

2. **With License**:
   - Configure license in Project Settings
   - Run the task
   - Verify license is retrieved successfully
   - Confirm task executes with all features enabled

3. **Without OAuth Token**:
   - Run without `SYSTEM_ACCESSTOKEN` in env
   - Verify warning message appears
   - Ensure task handles the missing token appropriately
