# ADO Backup & Restore - Complete Deployment Guide

## Overview

This guide provides step-by-step instructions for installing and configuring the Easy Azure DevOps Backup extension from the Azure DevOps marketplace. The extension includes:
- **Pipeline Tasks**: Pre-built backup and restore tasks for Azure Pipelines
- **Built-in CLI**: Command-line tools are bundled within the extension (no separate installation needed)

## Prerequisites

- Azure DevOps organization with admin access
- Personal Access Token (PAT) with appropriate permissions:
  - **For Backups:** Code (Read), Build (Read), Work Items (Read), Variable Groups (Read)
  - **For Restores:** Code (Read & Write), Build (Read & Write), Work Items (Read & Write), Variable Groups (Read & Write)
- Backup storage location (local path, network share, or cloud storage)
- Valid license key (trial or production)

## Part 1: Build the CLI Application

### 1.1 Compile the .NET Application

```bash
cd C:\Users\aharo\source\repos\ado-backup-restore

# Restore packages
dotnet restore

# Build in Release mode
dotnet build -c Release

# Run tests (optional)
dotnet test ADOBackupRestore.Tests/Lamdat.ADOBackupRestore.UnitTests.csproj -c Release
```

### 1.2 Publish as Self-Contained Executable

Create or update the CLI project file to include publishing configuration:

```bash
# For Windows
dotnet publish ADOBackupRestore.CLI/Lamdat.ADOBackupRestore.CLI.csproj \
  -c Release \
  -r win-x64 \
  --self-contained

# For Linux
dotnet publish ADOBackupRestore.CLI/Lamdat.ADOBackupRestore.CLI.csproj \
  -c Release \
  -r linux-x64 \
  --self-contained

# For macOS
dotnet publish ADOBackupRestore.CLI/Lamdat.ADOBackupRestore.CLI.csproj \
  -c Release \
  -r osx-x64 \
  --self-contained
```

Output will be in: `ADOBackupRestore.CLI/bin/Release/net9.0/win-x64/publish/`

### 1.3 Verify CLI Works

```bash
cd "ADOBackupRestore.CLI/bin/Release/net9.0/win-x64/publish"
.\adobackup.exe --help
```

Expected output shows available commands:
- build-backup / build-restore
- git-backup / git-restore
- workitems-backup / workitems-restore
- pipeline-variables-backup / pipeline-variables-restore

## Part 2: Build the VSIX Extension

### 2.1 Install Node Dependencies

```bash
cd ADOBackupRestore.VSIX
npm install
```

### 2.2 Build TypeScript

```bash
npm run build
```

This compiles all `.ts` files to `.js` files.

### 2.3 Create VSIX Package

```bash
npm run package
```

Output: `ado-backup-restore-extension-1.0.0.vsix`

## Part 3: Deploy CLI to Build Agents

Choose ONE of the following approaches:

### Approach A: Include in Extension Package (Recommended)

1. **Prepare CLI in extension:**
   ```bash
   cd ADOBackupRestore.VSIX
   mkdir -p tools
   
   # Copy executables for all platforms
   cp ../ADOBackupRestore.CLI/bin/Release/net9.0/win-x64/publish/adobackup.exe tools/
   ```

2. **Update tasks to use included CLI:**
   - Edit each task's `index.ts`
   - Update CLI path to: `path.join(__dirname, '..', '..', 'tools', 'adobackup.exe')`

3. **Rebuild extension:**
   ```bash
   npm run build
   npm run package
   ```

### Approach B: Deploy to Build Agents

1. **Create agent setup script:**

   **setup-agent-windows.ps1:**
   ```powershell
   $ToolsPath = "C:\tools"
   if (!(Test-Path $ToolsPath)) {
       New-Item -ItemType Directory -Path $ToolsPath -Force
   }
   
 # Copy CLI
   Copy-Item -Path ".\adobackup.exe" -Destination "$ToolsPath\adobackup.exe" -Force
   
   # Set environment variable (for all users)
   [Environment]::SetEnvironmentVariable("ADOBACKUP_CLI_PATH", "$ToolsPath\adobackup.exe", "Machine")
   
   # Verify
   & "$ToolsPath\adobackup.exe" --help
   ```

2. **Deploy to agents:**
   - Run setup script on each Windows agent
   - Restart agents if needed
   - Verify with: `echo %ADOBACKUP_CLI_PATH%`

3. **Update agent configuration:**
   - Edit agent's `.env` or startup script
   - Set `ADOBACKUP_CLI_PATH` environment variable

### Approach C: Deploy via Agent Pool Setup

1. **Create Azure Pipelines setup step:**
   ```yaml
   steps:
     - task: PowerShell@2
       displayName: 'Setup ADO Backup CLI'
    inputs:
targetType: 'inline'
         script: |
           $ToolsPath = "$env:AGENT_TOOLSDIRECTORY\ado-backup"
        if (!(Test-Path $ToolsPath)) {
     New-Item -ItemType Directory -Path $ToolsPath -Force
    }
           
           # Download/copy CLI
           Copy-Item -Path "$(Build.SourcesDirectory)\publish\adobackup.exe" -Destination "$ToolsPath\adobackup.exe" -Force
           
     echo "##vso[task.setvariable variable=ADOBACKUP_CLI_PATH]$ToolsPath\adobackup.exe"
   ```

## Part 2: Obtain License Key

### 2.1 Request Trial License

1. Visit the extension page in the marketplace
2. Click **"Start free trial"** or contact **support@lamdat.com**
3. Provide:
   - Organization name
   - Contact email
   - Intended use case

### 2.2 Receive License Key

You will receive:
- **License key** (encrypted string)
- **Expiration date**
- **Allowed organization(s)**

### 2.3 Store License Key Securely

Store the license key as a secret variable:
1. Go to **Pipelines** → **Library** → **Variable groups**
2. Create or edit your variable group (e.g., "ADO Backup Restore")
3. Add variable: `Backup.LicenseKey` (mark as **Secret**)

## Part 3: Configure Variables and Credentials

### 5.1 Create Azure DevOps Service Connection

In your Azure DevOps project:

1. Go to **Project Settings** ? **Service connections**
2. Click **Create service connection**
3. Select **Azure DevOps**
4. Configure:
   - **Connection name:** ADO_ServiceConnection
   - **Organization URL:** https://dev.azure.com/YOUR_ORGANIZATION
   - **Personal access token:** Create new PAT with scopes:
     - Build (read and execute)
 - Code (read and write)
     - Work Item (read and write)
     - Release (read, create, and execute)
     - Variable Groups (read and manage)

### 5.2 Create and Test Token

```bash
# Use Azure DevOps CLI
az devops login --organization https://dev.azure.com/YOUR_ORGANIZATION

# Create PAT
az devops user token create --org-owner <org_owner> --token-expiration 365
```

## Part 4: Test the Extension

### 4.1 Verify Extension is Available

1. Create or open a pipeline in your project
2. Click **+** to add a task
3. Search for "Backup" or "Restore"
4. You should see:
   - **Backup All Azure DevOps Resources**
   - **Restore All Azure DevOps Resources**

### 4.2 Create Test Pipeline

Create `.azure-pipelines/test-backup.yml`:

```yaml
trigger:
  - main

pool:
  vmImage: 'windows-latest'

stages:
  - stage: TestBackup
    displayName: 'Test Backup Operations'
    jobs:
      - job: TestBuildBackup
        displayName: 'Test Build Backup'
        steps:
    - task: BuildBackupTask@1
    displayName: 'Backup Build Definitions'
   inputs:
          connectedServiceName: 'ADO_ServiceConnection'
       Projects: ''
  Definitions: ''
      IncludeHistory: true
        MaxBuilds: '10'
        Days: '7'
        Verbose: true

      - job: TestGitBackup
   displayName: 'Test Git Backup'
        steps:
          - task: GitBackupTask@1
            displayName: 'Backup Git Repositories'
     inputs:
      connectedServiceName: 'ADO_ServiceConnection'
        Projects: ''
              Repositories: ''
         Verbose: true

  - job: TestWorkItemsBackup
 displayName: 'Test Work Items Backup'
        steps:
          - task: WorkItemsBackupTask@1
   displayName: 'Backup Work Items'
 inputs:
              connectedServiceName: 'ADO_ServiceConnection'
              Projects: ''
       MinId: ''
   MaxId: ''
              Verbose: true

      - job: TestPipelineVariablesBackup
        displayName: 'Test Pipeline Variables Backup'
        steps:
   - task: PipelineVariablesBackupTask@1
    displayName: 'Backup Pipeline Variables'
            inputs:
            connectedServiceName: 'ADO_ServiceConnection'
    Projects: ''
              Verbose: true
```

### 6.3 Run Test Pipeline

1. Create pipeline in Azure DevOps
2. Connect to repository branch
3. Select the YAML file
4. Run pipeline
5. Verify all tasks complete successfully

## Part 5: Production Deployment Checklist

### Pre-Deployment

- [ ] Extension installed from marketplace
- [ ] License key obtained (trial or production)
- [ ] Variable group created with credentials
- [ ] PAT tokens created with appropriate scopes
- [ ] Backup storage location prepared and accessible
- [ ] Team trained on backup/restore procedures

### Deployment

- [ ] Extension enabled in organization
- [ ] Variable group configured with all credentials
- [ ] Test backup pipeline executed successfully
- [ ] Test restore pipeline executed successfully (dry-run)
- [ ] Backup schedule configured
- [ ] Monitoring and alerts set up

### Post-Deployment

- [ ] Monitor pipeline executions
- [ ] Verify backups are created
- [ ] Check log files for errors
- [ ] Perform restore test with sample data
- [ ] Document any issues encountered
- [ ] Update team documentation

## Troubleshooting

### Extension Not Installing

**Problem:** Extension won't install from marketplace

**Solution:**
1. Verify you have organization admin permissions
2. Check organization allows marketplace extensions
3. Try refreshing the marketplace page
4. Contact support@lamdat.com if issue persists

### Tasks Not Appearing

**Problem:** Tasks don't show up in pipeline editor

**Solution:**
1. Verify extension is installed
2. Check extension is enabled
3. Refresh browser/reload VS Code
4. Check task version in vss-extension.json

### License Key Invalid

**Problem:** Tasks fail with "Invalid license key"

**Solution:**
1. Verify license key is correctly copied (no extra spaces)
2. Check license hasn't expired
3. Ensure organization URL matches license
4. Contact support@lamdat.com for license issues

### Authentication Failures

**Problem:** Tasks fail with auth errors

**Solution:**
1. Verify PAT is valid and not expired
2. Check PAT has required scopes
3. Test PAT with CLI: `adobackup --help`
4. Verify organization URL is correct

### Backup/Restore Not Working

**Problem:** Operations succeed but no changes occur

**Solution:**
1. Enable verbose logging
2. Check CLI output logs
3. Verify backup location accessibility
4. Check user permissions on organization
5. Run test restore with dry-run first

## Update Process

### Extension Updates

The extension is automatically updated from the marketplace:

1. **Automatic Updates:**
   - Minor updates install automatically
   - No action required

2. **Manual Updates:**
   - Go to **Organization Settings** → **Extensions**
   - Find "Easy Azure DevOps Backup"
   - Click **Update** if available

3. **Version Compatibility:**
   - Backups from older versions are compatible with newer versions
   - Review release notes before updating production environments

## Support and Documentation

- **Task Issues:** Enable verbose logging in task inputs
- **Extension Issues:** Check pipeline logs for detailed output
- **License Issues:** Contact support@lamdat.com
- **Documentation:** https://github.com/lamdat-tech/easy-azure-devops-backup-docs
- **Support Email:** support@lamdat.com

## Additional Resources

- [Azure DevOps Extension Development](https://learn.microsoft.com/en-us/azure/devops/extend/)
- [Building Tasks](https://learn.microsoft.com/en-us/azure/devops/extend/develop/add-build-task)
- [Publishing to Marketplace](https://learn.microsoft.com/en-us/azure/devops/extend/publish/overview)
- [Service Connections](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints)
