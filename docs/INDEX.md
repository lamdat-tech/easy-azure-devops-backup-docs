# ADO Backup & Restore VSIX Extension - Complete Index

## ?? Documentation Guide

### Start Here ??
1. **[SOLUTION_SUMMARY.md](SOLUTION_SUMMARY.md)** - Overview of everything that's been created
2. **[QUICK_START.md](QUICK_START.md)** - Get running in 5 minutes
3. **[README.md](README.md)** - Extension architecture and usage

### For Different Audiences

#### ????? Developers
1. Read: [BUILD_GUIDE.md](BUILD_GUIDE.md) - How to build and package
2. Reference: [CLI_INTEGRATION_GUIDE.md](CLI_INTEGRATION_GUIDE.md) - How tasks integrate with CLI
3. Explore: Source files in each `*Task/` directory

#### ?? DevOps Engineers
1. Read: [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) - Complete deployment procedure
2. Reference: [QUICK_START.md](QUICK_START.md) - Common pipeline examples
3. Check: [CLI_INTEGRATION_GUIDE.md](CLI_INTEGRATION_GUIDE.md) - Agent setup details

#### ?? End Users
1. Start: [QUICK_START.md](QUICK_START.md) - Common tasks and examples
2. Reference: [OVERVIEW.md](OVERVIEW.md) - Features and capabilities
3. Learn: Specific task examples in QUICK_START

### Comprehensive Guides

| Document | Purpose | Length | Audience |
|----------|---------|--------|----------|
| **SOLUTION_SUMMARY.md** | What was created and how to use it | 5-10 min | Everyone |
| **QUICK_START.md** | Get started in 5 minutes with examples | 5 min | Everyone |
| **README.md** | Extension architecture and features | 10 min | Developers & DevOps |
| **BUILD_GUIDE.md** | How to build and package the extension | 15 min | Developers |
| **DEPLOYMENT_GUIDE.md** | Complete deployment procedure | 20-30 min | DevOps Engineers |
| **CLI_INTEGRATION_GUIDE.md** | How tasks work with CLI | 15 min | Advanced users |
| **OVERVIEW.md** | User-facing feature documentation | 10 min | End users |

## ??? Directory Structure

```
ADOBackupRestore.VSIX/
?
??? ?? Documentation Files
?   ??? README.md ............................ Extension overview
?   ??? SOLUTION_SUMMARY.md .................. Complete solution summary (START HERE)
?   ??? QUICK_START.md ....................... 5-minute setup guide
?   ??? BUILD_GUIDE.md ....................... Build & packaging instructions
?   ??? DEPLOYMENT_GUIDE.md .................. Full deployment procedure
?   ??? CLI_INTEGRATION_GUIDE.md ............. Task-CLI integration details
?   ??? OVERVIEW.md .......................... User documentation
?   ??? INDEX.md ............................ This file!
?
??? ?? Configuration Files
?   ??? vss-extension.json ................... Extension manifest
?   ??? package.json ......................... npm dependencies
?   ??? tsconfig.json ........................ TypeScript config
?   ??? .gitignore ........................... Git ignore rules
?
??? ??? Build Tasks (8 Total)
?   ??? BuildBackupTask/
?   ?   ??? task.json ........................ Task definition
?   ?   ??? index.ts ......................... Task implementation
?   ?
?   ??? BuildRestoreTask/
?   ?   ??? task.json
?   ?   ??? index.ts
?   ?
?   ??? GitBackupTask/
? ?   ??? task.json
?   ?   ??? index.ts
? ?
?   ??? GitRestoreTask/
?   ?   ??? task.json
?   ?   ??? index.ts
?   ?
?   ??? WorkItemsBackupTask/
?   ?   ??? task.json
? ?   ??? index.ts
?   ?
?   ??? WorkItemsRestoreTask/
?   ?   ??? task.json
?   ?   ??? index.ts
?   ?
?   ??? PipelineVariablesBackupTask/
?   ?   ??? task.json
?   ?   ??? index.ts
?   ?
?   ??? PipelineVariablesRestoreTask/
?       ??? task.json
?       ??? index.ts
?
??? ??? Build Scripts
?   ??? build.sh ............................. Linux/macOS build script
???? build.cmd ............................ Windows build script
?
??? ?? Generated Files (after build)
    ??? node_modules/ ........................ npm dependencies (after npm install)
    ??? dist/ ................................ Compiled JavaScript (after npm run build)
    ??? *.vsix ............................... VSIX package (after npm run package)
    ??? ...other build artifacts...
```

## ?? Reading Paths

### ?? I want to...

#### ...get the extension running immediately
1. Read: **QUICK_START.md** (5 minutes)
2. Run: `npm install && npm run package`
3. Deploy to Azure DevOps
4. Start using tasks!

#### ...understand how everything works
1. Read: **SOLUTION_SUMMARY.md** (10 minutes)
2. Read: **README.md** (10 minutes)
3. Browse: Task files in each `*Task/` directory (10 minutes)
4. Reference: **BUILD_GUIDE.md** as needed

#### ...deploy to production
1. Read: **QUICK_START.md** (5 minutes)
2. Build and test locally
3. Read: **DEPLOYMENT_GUIDE.md** (30 minutes)
4. Follow step-by-step deployment
5. Reference: **CLI_INTEGRATION_GUIDE.md** for agent setup

#### ...customize or extend
1. Read: **README.md** (Development section)
2. Study: One `*Task/` directory as template
3. Reference: **BUILD_GUIDE.md** (Development Tips)
4. Reference: **CLI_INTEGRATION_GUIDE.md** (Extending section)

#### ...troubleshoot issues
1. Check: **QUICK_START.md** (Troubleshooting section)
2. Check: **DEPLOYMENT_GUIDE.md** (Troubleshooting section)
3. Check: **CLI_INTEGRATION_GUIDE.md** (Troubleshooting section)
4. Enable verbose logging in task
5. Check pipeline logs and CLI output

## ?? Key Concepts

### Tasks
- **8 Build pipeline tasks** provided by the extension
- Each task performs a backup or restore operation
- Tasks use the CLI tool for actual operations
- Configurable via task inputs in pipeline

### Service Connections
- Required for authentication
- Holds Azure DevOps PAT token
- Automatically used by tasks
- See **DEPLOYMENT_GUIDE.md** Step 5

### CLI Application
- Separate .NET application (not in this folder)
- Performs actual backup/restore operations
- Invoked by tasks with command-line arguments
- See **CLI_INTEGRATION_GUIDE.md** for details

### Backup Operations
- Backup specific or all projects
- Support for incremental backups via date filtering
- Stores in configurable backup location
- Includes manifests and logs

### Restore Operations
- Requires specific backup date
- Can restore to original or different projects
- Supports dry-run mode for validation
- Full audit trail in logs

## ?? Common Workflows

### Workflow 1: Initial Setup & Test
```
1. npm install
2. npm run package
3. Upload .vsix to Azure DevOps
4. Create service connection
5. Add task to test pipeline
6. Run pipeline
7. Verify success ?
```

### Workflow 2: Production Deployment
```
1. Build & test locally (use QUICK_START)
2. Prepare CLI deployment
3. Follow DEPLOYMENT_GUIDE steps 1-5
4. Create production pipeline
5. Schedule regular backups
6. Test restore procedures
7. Monitor operations ?
```

### Workflow 3: Using in Pipeline
```
1. Create pipeline YAML
2. Add task (copy from QUICK_START)
3. Configure service connection
4. Set input parameters
5. Run pipeline
6. Check results in logs ?
```

## ?? Task Quick Reference

### Backup Tasks
| Task | CLI Command | Backups |
|------|-------------|---------|
| BuildBackupTask | `build-backup` | Build definitions + history |
| GitBackupTask | `git-backup` | Git repositories |
| WorkItemsBackupTask | `workitems-backup` | Work items |
| PipelineVariablesBackupTask | `pipeline-variables-backup` | Pipeline variables |

### Restore Tasks
| Task | CLI Command | Restores |
|------|-------------|----------|
| BuildRestoreTask | `build-restore` | Build definitions from backup |
| GitRestoreTask | `git-restore` | Git repositories from backup |
| WorkItemsRestoreTask | `workitems-restore` | Work items from backup |
| PipelineVariablesRestoreTask | `pipeline-variables-restore` | Pipeline variables from backup |

## ?? Learning Resources

### In This Package
- Complete source code for all 8 tasks
- Examples in **QUICK_START.md**
- Integration details in **CLI_INTEGRATION_GUIDE.md**
- Troubleshooting guides in multiple documents

### External Resources
- [Azure DevOps Extension Development](https://learn.microsoft.com/en-us/azure/devops/extend/)
- [Azure Pipelines Task Development](https://learn.microsoft.com/en-us/azure/devops/extend/develop/add-build-task)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [Azure Pipelines Task Lib](https://github.com/Microsoft/azure-pipelines-task-lib)

## ? Checklist

### Before First Use
- [ ] Read SOLUTION_SUMMARY.md
- [ ] Read QUICK_START.md
- [ ] Build extension: `npm run package`
- [ ] Prepare CLI executable
- [ ] Have Azure DevOps access

### Before Production
- [ ] Test in non-production environment
- [ ] Read DEPLOYMENT_GUIDE.md
- [ ] Create service connection
- [ ] Configure build agents
- [ ] Document procedures
- [ ] Train team members

### During Operations
- [ ] Monitor backup schedules
- [ ] Verify backup completion
- [ ] Test restore procedures
- [ ] Review logs regularly
- [ ] Update documentation as needed

## ?? Cross-References

### vss-extension.json
- Defines 8 task contributions
- References task directories
- Sets extension metadata
- See: **BUILD_GUIDE.md** for modification

### package.json
- npm dependencies listed
- Scripts: build, package
- See: **BUILD_GUIDE.md** Step 2

### Each task.json
- Task metadata and inputs
- Execution configuration
- Input grouping and help text
- See: **CLI_INTEGRATION_GUIDE.md** for details

### Each index.ts
- Task implementation in TypeScript
- CLI invocation logic
- Error handling
- See: **CLI_INTEGRATION_GUIDE.md** Template

## ?? File Descriptions

### Configuration
| File | Purpose |
|------|---------|
| vss-extension.json | Extension definition and task contributions |
| package.json | npm dependencies and build scripts |
| tsconfig.json | TypeScript compiler options |
| .gitignore | Git ignore patterns |

### Documentation
| File | Best For |
|------|----------|
| README.md | Architecture and features |
| SOLUTION_SUMMARY.md | Complete overview |
| QUICK_START.md | Getting started |
| BUILD_GUIDE.md | Building and packaging |
| DEPLOYMENT_GUIDE.md | Production deployment |
| CLI_INTEGRATION_GUIDE.md | Understanding integration |
| OVERVIEW.md | End-user reference |
| INDEX.md | This file! Navigation help |

### Scripts
| File | Platform | Use Case |
|------|----------|----------|
| build.sh | Linux/macOS | Automated build process |
| build.cmd | Windows | Automated build process |

## ?? Next Steps

1. **Start**: Read [SOLUTION_SUMMARY.md](SOLUTION_SUMMARY.md)
2. **Learn**: Read [QUICK_START.md](QUICK_START.md)
3. **Build**: Follow **BUILD_GUIDE.md**
4. **Deploy**: Follow **DEPLOYMENT_GUIDE.md**
5. **Use**: Refer to **QUICK_START.md** for examples

---

## ?? Support

### Documentation Issues
- Check if answer is in relevant guide
- Look in Troubleshooting sections
- Review examples in QUICK_START.md

### Technical Issues
- Enable verbose logging
- Check CLI output
- Review relevant guide's troubleshooting
- Check pipeline logs

### Extension Issues
- Verify installation
- Check service connection
- Run test pipeline
- Enable verbose output

---

**Last Updated**: 2024
**Version**: 1.0.0
**Status**: Complete and Ready for Deployment ?

Start with [QUICK_START.md](QUICK_START.md) or [SOLUTION_SUMMARY.md](SOLUTION_SUMMARY.md)!
