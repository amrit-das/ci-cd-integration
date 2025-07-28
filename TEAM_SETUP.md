# GitHub Team and Branch Protection Setup Guide

This document outlines how to configure GitHub repository settings, team permissions, and branch protection rules for the Dataiku CI/CD workflow.

## 🔐 Team Structure

### Required Teams

1. **Admin Support** (`admin-support`)
   - Repository admin access
   - Can manage settings, teams, and branch protection rules
   - Can merge any PR in emergency situations

2. **Developers** (`developers`)
   - Write access to `dev` branch only
   - Can create feature branches from `dev`
   - Can create PRs to `dev` branch
   - Cannot directly push to `preprod` or `prod`

3. **Triage Team** (`triage`)
   - Can merge approved PRs into `preprod`
   - Can review and test features in preprod environment
   - Cannot merge to production

4. **MLOps Team** (`mlops`)
   - Required reviewers for `preprod` and `prod` PRs
   - Can approve and merge production deployments
   - Usually overlaps with admin team members
   - Responsible for production stability

## 🌿 Branch Structure

```
main/prod (production)
├── preprod (pre-production)
│   └── dev (development)
│       ├── feature/user-auth
│       ├── feature/data-pipeline
│       └── hotfix/bug-123
```

## ⚙️ Repository Configuration

### 1. Create Teams

Go to your GitHub organization settings:

```bash
# Using GitHub CLI (optional)
gh api orgs/{ORG}/teams -f name="admin-support" -f description="Admin support team"
gh api orgs/{ORG}/teams -f name="developers" -f description="Development team"
gh api orgs/{ORG}/teams -f name="triage" -f description="Triage and testing team"
gh api orgs/{ORG}/teams -f name="mlops" -f description="MLOps and production team"
```

### 2. Branch Protection Rules

#### Dev Branch Protection
- **Branch name pattern**: `dev`
- **Require pull request reviews**: ✅
  - Required approving reviews: 1
  - Dismiss stale reviews: ✅
- **Require status checks**: ✅
  - Required checks:
    - `Lint with Ruff`
    - `Validate Team Permissions`
    - `Dataiku-Specific Validations`
- **Restrict pushes**: ✅
  - Teams: `developers`, `admin-support`
- **Allow force pushes**: ❌
- **Allow deletions**: ❌

#### Preprod Branch Protection
- **Branch name pattern**: `preprod`
- **Require pull request reviews**: ✅
  - Required approving reviews: 2
  - Require review from CODEOWNERS: ✅
  - Dismiss stale reviews: ✅
- **Require status checks**: ✅
  - Required checks:
    - `Validate PR Requirements`
    - `Comprehensive Testing Suite`
    - `Integration Testing`
- **Restrict pushes**: ✅
  - Teams: `triage`, `mlops`, `admin-support`
- **Allow force pushes**: ❌
- **Allow deletions**: ❌

#### Production Branch Protection
- **Branch name pattern**: `main` or `prod`
- **Require pull request reviews**: ✅
  - Required approving reviews: 2
  - Require review from CODEOWNERS: ✅
  - Dismiss stale reviews: ✅
- **Require status checks**: ✅
  - Required checks:
    - `Validate MLOps Team Access`
    - `Pre-Production Validation Suite`
    - `Backup and Rollback Preparation`
- **Restrict pushes**: ✅
  - Teams: `mlops`, `admin-support`
- **Allow force pushes**: ❌
- **Allow deletions**: ❌

### 3. CODEOWNERS File

Create `.github/CODEOWNERS`:

```
# Global owners
* @your-org/admin-support

# Development branch - developers can review
/dev/** @your-org/developers

# Preprod requires MLOps review
/preprod/** @your-org/mlops

# Production requires MLOps team
/ @your-org/mlops
*.yml @your-org/mlops
*.yaml @your-org/mlops
pyproject.toml @your-org/mlops

# Critical Dataiku files
/recipes/ @your-org/mlops
/datasets/ @your-org/mlops
params.json @your-org/mlops
```

### 4. Repository Settings

#### General Settings
- **Default branch**: `dev`
- **Allow merge commits**: ✅
- **Allow squash merging**: ✅
- **Allow rebase merging**: ❌
- **Automatically delete head branches**: ✅

#### Actions Settings
- **Actions permissions**: Allow all actions and reusable workflows
- **Fork pull request workflows**: Require approval for all outside collaborators

## 🚀 Workflow Configuration

### Environment Secrets

Configure these secrets in repository settings:

```
# Production Environment
DATAIKU_PROD_API_KEY
DATAIKU_PROD_URL
PROD_DATABASE_URL
PROD_NOTIFICATION_WEBHOOK

# Preprod Environment  
DATAIKU_PREPROD_API_KEY
DATAIKU_PREPROD_URL
PREPROD_DATABASE_URL

# Dev Environment
DATAIKU_DEV_API_KEY
DATAIKU_DEV_URL
DEV_DATABASE_URL

# General
SLACK_WEBHOOK_URL
TEAMS_WEBHOOK_URL
```

### Environment Protection Rules

#### Development Environment
- **Required reviewers**: None
- **Wait timer**: 0 minutes
- **Deployment branches**: `dev` only

#### Preprod Environment
- **Required reviewers**: `@your-org/mlops` (1 reviewer)
- **Wait timer**: 5 minutes
- **Deployment branches**: `preprod` only

#### Production Environment
- **Required reviewers**: `@your-org/mlops` (2 reviewers)
- **Wait timer**: 30 minutes
- **Deployment branches**: `main`/`prod` only

## 👥 Team Assignment Script

Use this script to assign team members:

```bash
#!/bin/bash

# Add members to teams (replace with actual usernames)
gh api orgs/{ORG}/teams/admin-support/memberships/{USERNAME} -f role=maintainer
gh api orgs/{ORG}/teams/developers/memberships/{USERNAME} -f role=member
gh api orgs/{ORG}/teams/triage/memberships/{USERNAME} -f role=member
gh api orgs/{ORG}/teams/mlops/memberships/{USERNAME} -f role=maintainer
```

## 🔄 Workflow Process

### Developer Workflow
1. Create feature branch from `dev`: `git checkout -b feature/my-feature dev`
2. Make changes and commit
3. Push branch: `git push origin feature/my-feature`
4. Create PR to `dev` branch
5. Wait for automatic linting and validation
6. After approval, merge to `dev`

### Preprod Deployment
1. Create PR from `dev` to `preprod`
2. MLOps team reviews and approves
3. Triage team merges after approval
4. Comprehensive testing runs automatically
5. Environment deployed and ready for testing

### Production Deployment
1. Create PR from `preprod` to `main`/`prod`
2. MLOps team performs thorough review
3. Only MLOps team can approve and merge
4. Production deployment with monitoring setup
5. 24-hour monitoring period with rollback ready

## 🚨 Emergency Procedures

### Hotfix Process
1. Create hotfix branch from `main`: `git checkout -b hotfix/critical-fix main`
2. Make minimal fix
3. Create PR directly to `main` (requires admin approval)
4. After merge, cherry-pick to `preprod` and `dev`

### Rollback Process
1. Use rollback plan generated during deployment
2. MLOps team lead approves rollback
3. Execute automated rollback workflow
4. Verify system stability
5. Document incident and lessons learned

## 📊 Monitoring and Alerts

Set up monitoring for:
- Failed CI/CD workflows
- Deployment status changes  
- Security scan results
- Performance degradation
- Team permission changes

Configure alerts to notify:
- MLOps team for production issues
- Developers for dev branch failures
- Admin team for security alerts 