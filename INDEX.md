# Azure DevOps Backup & Restore - Documentation Index

Welcome to the comprehensive documentation for the Azure DevOps Backup & Restore utility (`adobackup`).

## 📚 Documentation Structure

### Getting Started

Start here if you're new to the solution:

- **[README](./README.md)** - Overview, features, and quick start guide
- **[Getting Started Guide](./getting-started.md)** - Step-by-step tutorial for first-time users
  - License setup
  - Variable group configuration
  - Your first backup pipeline
  - Your first restore pipeline

### Integration Guides

Learn how to integrate the utility into your workflows:

- **[Pipeline Integration Guide](./pipeline-integration.md)** - Using with Azure Pipelines
  - Setting up variable groups
  - Backup pipeline examples
  - Restore pipeline examples
  - Self-hosted agent configuration
  - Multi-stage pipelines
  - Troubleshooting pipeline issues

### Best Practices

Optimize your backup and restore strategy:

- **[Best Practices Guide](./best-practices.md)** - Recommended patterns and strategies
  - 3-2-1 backup rule
  - Incremental vs full backups
  - Retention policies
  - Storage management
  - Security best practices
  - Performance optimization
  - Disaster recovery planning
  - Compliance and auditing

### Real-World Scenarios

Learn from practical examples:

- **[Use Cases and Scenarios](./use-cases.md)** - Real-world implementations
  - Disaster recovery
  - Organization migration
  - Project cloning
  - Development environment setup
  - Compliance and auditing
  - Continuous backup strategy
  - Multi-organization management

### Support Resources

Get help when you need it:

- **[FAQ (Frequently Asked Questions)](./faq.md)** - Answers to common questions
  - General questions
  - Backup questions
  - Restore questions
  - Security questions
  - Performance questions
  - Licensing questions
  - Integration questions

- **[Troubleshooting Guide](./troubleshooting.md)** - Solutions to common issues
  - Authentication issues
  - Backup issues
  - Restore issues
  - Performance issues
  - Network issues
  - Storage issues
  - License issues
  - Pipeline integration issues
  - Error messages reference

## 🗂️ Documentation by Topic

### Backup Operations

| Topic | Documents |
|-------|-----------|
| Basic backups | [Getting Started](./getting-started.md), [Pipeline Integration](./pipeline-integration.md) |
| Incremental backups | [Best Practices](./best-practices.md#backup-strategy), [FAQ](./faq.md#backup-questions) |
| Selective backups | [Pipeline Integration](./pipeline-integration.md), [Use Cases](./use-cases.md#selective-data-management) |
| Automated backups | [Pipeline Integration](./pipeline-integration.md), [Best Practices](./best-practices.md#backup-strategy) |
| Backup validation | [Best Practices](./best-practices.md#disaster-recovery), [Troubleshooting](./troubleshooting.md#backup-issues) |
| Areas & Iterations backup | [Getting Started](./getting-started.md#limitations--known-issues), [Pipeline Integration](./pipeline-integration.md) |

### Restore Operations

| Topic | Documents |
|-------|-----------|
| Basic restores | [Getting Started](./getting-started.md#your-first-restore-pipeline), [Pipeline Integration](./pipeline-integration.md) |
| Cross-org restores | [Use Cases](./use-cases.md#organization-migration), [Pipeline Integration](./pipeline-integration.md#example-2-cross-organization-restore) |
| Work Items cross-org restore | [Use Cases](./use-cases.md#organization-migration), [Pipeline Integration](./pipeline-integration.md) |
| Project cloning | [Use Cases](./use-cases.md#project-cloning), [Getting Started](./getting-started.md#your-first-restore-pipeline) |
| Dry-run mode | [Getting Started](./getting-started.md#your-first-restore-pipeline), [FAQ](./faq.md#restore-questions) |
| Selective restores | [Pipeline Integration](./pipeline-integration.md), [Use Cases](./use-cases.md#selective-data-management) |
| Areas & Iterations restore | [Getting Started](./getting-started.md#limitations--known-issues), [Pipeline Integration](./pipeline-integration.md) |

### Security & Compliance

| Topic | Documents |
|-------|-----------|
| PAT token management | [Best Practices](./best-practices.md#security), [FAQ](./faq.md#security-questions) |
| Backup encryption | [Best Practices](./best-practices.md#security), [Troubleshooting](./troubleshooting.md#security-issues) |
| Compliance requirements | [Best Practices](./best-practices.md#compliance-and-auditing), [Use Cases](./use-cases.md#compliance-and-auditing) |
| Audit trails | [Best Practices](./best-practices.md#compliance-and-auditing), [Use Cases](./use-cases.md#compliance-and-auditing) |
| License management | [Getting Started](./getting-started.md#license-setup), [FAQ](./faq.md#licensing-questions) |

### Performance

| Topic | Documents |
|-------|-----------|
| Performance optimization | [Best Practices](./best-practices.md#performance-optimization), [FAQ](./faq.md#performance-questions) |
| Parallelism tuning | [Best Practices](./best-practices.md#performance-optimization), [Troubleshooting](./troubleshooting.md#performance-issues) |
| Storage management | [Best Practices](./best-practices.md#storage-management), [Troubleshooting](./troubleshooting.md#storage-issues) |
| Rate limiting | [FAQ](./faq.md#backup-questions), [Troubleshooting](./troubleshooting.md#performance-issues) |

### Integration

| Topic | Documents |
|-------|-----------|
| Azure Pipelines | [Pipeline Integration](./pipeline-integration.md), [Use Cases](./use-cases.md#continuous-backup-strategy) |
| CI/CD systems | [Pipeline Integration](./pipeline-integration.md), [FAQ](./faq.md#integration-questions) |
| Monitoring | [Best Practices](./best-practices.md#disaster-recovery), [Pipeline Integration](./pipeline-integration.md#best-practices) |
| Multi-organization | [Use Cases](./use-cases.md#multi-organization-management), [Pipeline Integration](./pipeline-integration.md#example-2-cross-organization-restore) |

## ⚡ Quick Reference

### Common Tasks

| Task | Document | Section |
|------|----------|---------|
| Set up license | [Getting Started](./getting-started.md) | License Setup |
| First backup pipeline | [Getting Started](./getting-started.md) | Your First Backup Pipeline |
| Incremental backup | [Best Practices](./best-practices.md) | Backup Strategy |
| First restore pipeline | [Getting Started](./getting-started.md) | Your First Restore Pipeline |
| Migrate to new org | [Use Cases](./use-cases.md) | Organization Migration |
| Clone a project | [Use Cases](./use-cases.md) | Project Cloning |
| Advanced pipeline examples | [Pipeline Integration](./pipeline-integration.md) | Backup/Restore Pipeline Examples |
| Troubleshoot failures | [Troubleshooting](./troubleshooting.md) | All sections |

## 🎓 Learning Paths

### Path 1: New User (0-2 hours)

1. Read [README](./README.md) for overview
2. Follow [Getting Started Guide](./getting-started.md)
3. Set up your first backup pipeline
4. Review [FAQ](./faq.md) for common questions

### Path 2: Production Deployment (2-4 hours)

1. Complete Path 1
2. Study [Best Practices Guide](./best-practices.md)
3. Review [Use Cases](./use-cases.md) for your scenario
4. Explore advanced examples in [Pipeline Integration](./pipeline-integration.md)
5. Bookmark [Troubleshooting Guide](./troubleshooting.md)

### Path 3: Enterprise Administrator (4-8 hours)

1. Complete Paths 1 and 2
2. Implement disaster recovery plan from [Best Practices](./best-practices.md)
3. Set up monitoring and alerting
4. Configure multi-organization backup strategies
5. Document custom procedures
6. Train team members

## 📖 Additional Resources

### Example Files

- **[backup.yml](../DevOps/examples/backup.yml)** - Example Azure Pipeline for backup
- **[restore.yml](../DevOps/examples/restore.yml)** - Example Azure Pipeline for restore

### Internal Documentation

- **[docs/internal/](./internal/)** - Internal development and architecture documentation
  - System design
  - API documentation
  - Development guidelines

## 💬 Getting Help

### Self-Service

1. Search [FAQ](./faq.md)
2. Check [Troubleshooting Guide](./troubleshooting.md)
3. Review the [Pipeline Integration Guide](./pipeline-integration.md)
4. Look for similar scenarios in [Use Cases](./use-cases.md)

### Support Channels

- **Website:** [easyadobackup.com](https://easyadobackup.com/)
- **Documentation Issues:** Report via [easyadobackup.com](https://easyadobackup.com/)
- **Feature Requests:** Submit via [easyadobackup.com](https://easyadobackup.com/)

### Information to Include in Support Requests

- Detailed error description
- Log files from `{BackupRoot}/logs/`
- Command used (with sensitive data redacted)
- Environment details:
  - Operating system
  - .NET version
  - Azure DevOps organization size
  - Extension version



## 📄 License

Copyright � 2025 Lamdat. All rights reserved.

This software requires a valid license key for production use. Visit [easyadobackup.com](https://easyadobackup.com/) for licensing information.

---

**Need help?** Start with the [Getting Started Guide](./getting-started.md) or visit [easyadobackup.com](https://easyadobackup.com/)
