# GitHub CI/CD Workflows Documentation

This document provides detailed information about all GitHub Actions workflows in the project.

## Table of Contents

- [GitHub CI/CD Workflows Documentation](#github-cicd-workflows-documentation)
  - [Table of Contents](#table-of-contents)
  - [Deployment Workflows](#deployment-workflows)
    - [Deploy to Staging](#deploy-to-staging)
    - [Deploy to Production](#deploy-to-production)
  - [Testing Workflows](#testing-workflows)
    - [Test Backend](#test-backend)
    - [Playwright Tests](#playwright-tests)
    - [Test Docker Compose](#test-docker-compose)
    - [Smokeshow](#smokeshow)
  - [Code Quality Workflows](#code-quality-workflows)
    - [pre-commit](#pre-commit)
  - [Project Management Workflows](#project-management-workflows)
    - [Issue Manager](#issue-manager)
    - [Labels](#labels)
    - [Latest Changes](#latest-changes)
    - [Add to Project](#add-to-project)
    - [Conflict Detector](#conflict-detector)
  - [Workflow Dependencies](#workflow-dependencies)
  - [Secrets Configuration](#secrets-configuration)
    - [Deployment Secrets](#deployment-secrets)
    - [Application Configuration](#application-configuration)
    - [SMTP Email Configuration](#smtp-email-configuration)
    - [CI/CD Tools](#cicd-tools)
  - [Branch Protection Requirements](#branch-protection-requirements)
  - [Best Practices](#best-practices)
  - [Troubleshooting](#troubleshooting)
    - [Pre-commit Failures](#pre-commit-failures)
    - [Playwright Test Failures](#playwright-test-failures)
    - [Deployment Failures](#deployment-failures)
    - [Insufficient Coverage](#insufficient-coverage)

---

## Deployment Workflows

### Deploy to Staging

**File**: `.github/workflows/deploy-staging.yml`

**Triggers**:
- When code is pushed to the `master` branch

**Description**:
Automatically deploys the application to the staging environment. This workflow only runs in user projects (not in the main fastapi repository).

**Main Steps**:
1. Checkout code
2. Build Docker images using Docker Compose
3. Start service containers

**Runtime Environment**:
- Uses self-hosted runners with labels `self-hosted` and `staging`

**Environment Variables** (configured via Secrets):
- `DOMAIN_STAGING` - Staging environment domain
- `STACK_NAME_STAGING` - Docker Stack name
- `SECRET_KEY` - Application secret key
- `FIRST_SUPERUSER` - First superuser account
- `FIRST_SUPERUSER_PASSWORD` - Superuser password
- `SMTP_HOST`, `SMTP_USER`, `SMTP_PASSWORD` - Email server configuration
- `EMAILS_FROM_EMAIL` - Sender email address
- `POSTGRES_PASSWORD` - Database password
- `SENTRY_DSN` - Sentry error tracking configuration

---

### Deploy to Production

**File**: `.github/workflows/deploy-production.yml`

**Triggers**:
- When a GitHub Release is published (`types: published`)

**Description**:
Automatically deploys the application to the production environment. This workflow only runs in user projects (not in the main fastapi repository).

**Main Steps**:
1. Checkout code
2. Build Docker images using Docker Compose
3. Start service containers

**Runtime Environment**:
- Uses self-hosted runners with labels `self-hosted` and `production`

**Environment Variables** (configured via Secrets):
Similar to Staging, but uses secrets with `PRODUCTION` suffix.

---

## Testing Workflows

### Test Backend

**File**: `.github/workflows/test-backend.yml`

**Triggers**:
- Push to `master` branch
- Pull Request opened or synchronized

**Description**:
Runs backend unit and integration tests, generating test coverage reports.

**Main Steps**:
1. Checkout code
2. Setup Python 3.10 environment
3. Install uv package manager
4. Start database and mail catcher services (using Docker Compose)
5. Run database migrations
6. Execute test suite and generate coverage report
7. Upload coverage HTML report as artifact
8. Verify code coverage is at least 90%

**Dependent Services**:
- PostgreSQL database
- Mailcatcher (email testing tool)

**Quality Gate**:
- Requires test coverage of at least 90%

---

### Playwright Tests

**File**: `.github/workflows/playwright.yml`

**Triggers**:
- Push to `master` branch
- Pull Request opened or synchronized
- Manual trigger (supports tmate debugging mode)

**Description**:
Runs end-to-end (E2E) tests using the Playwright testing framework. Tests are executed in parallel shards for efficiency.

**Main Steps**:
1. **changes job**: Detects relevant file changes
   - Only runs tests when backend, frontend, .env, compose files, or the workflow itself changes
   
2. **test-playwright job**: Execute tests (4 parallel shards)
   - Checkout code
   - Setup Bun and Python environments
   - Install dependencies (uv and bun)
   - Generate API client
   - Build Docker containers
   - Run Playwright tests (each shard runs independently)
   - Upload test report blob data

3. **merge-playwright-reports job**: Merge test reports
   - Download all shard test reports
   - Merge into unified HTML report
   - Upload final HTML report

**Test Sharding Strategy**:
- Total of 4 shards (shardTotal: 4)
- Run in parallel to speed up testing

**Debugging Features**:
- Supports manual trigger via workflow_dispatch with tmate debugging session

**Quality Assurance**:
- Uses `alls-green-playwright` job for branch protection checks

---

### Test Docker Compose

**File**: `.github/workflows/test-docker-compose.yml`

**Triggers**:
- Push to `master` branch
- Pull Request opened or synchronized

**Description**:
Tests that Docker Compose configuration works properly, ensuring all services can start and be accessed correctly.

**Main Steps**:
1. Checkout code
2. Build Docker images
3. Start services (backend, frontend, adminer)
4. Test backend health check endpoint (`http://localhost:8000/api/v1/utils/health-check`)
5. Test frontend accessibility (`http://localhost:5173`)
6. Clean up containers and volumes

**Validation**:
- Docker images build successfully
- All services start properly
- Backend API is accessible
- Frontend page is accessible

---

### Smokeshow

**File**: `.github/workflows/smokeshow.yml`

**Triggers**:
- When "Test Backend" workflow completes

**Description**:
Automatically uploads and displays test coverage reports using the Smokeshow service.

**Main Steps**:
1. Checkout code
2. Setup Python 3.13 environment
3. Install smokeshow tool
4. Download coverage HTML report generated by Test Backend
5. Upload report to Smokeshow

**Integration Features**:
- Automatically adds coverage status to Pull Requests
- Sets coverage threshold to 90%
- Provides coverage visualization link

**Permissions Required**:
- `actions: read` - Read workflow run information
- `statuses: write` - Update commit status

---

## Code Quality Workflows

### pre-commit

**File**: `.github/workflows/pre-commit.yml`

**Triggers**:
- Pull Request opened or synchronized

**Description**:
Automatically runs code formatting and linting checks, and commits fixes when necessary.

**Main Steps**:
1. Checkout code
   - For repositories with permissions: Check out PR branch head (can commit changes)
   - For forks or Dependabot: Use read-only checkout
   
2. Setup environment
   - Python 3.11
   - Bun (frontend)
   - uv (Python package manager)
   
3. Install dependencies
   - Backend and frontend dependencies
   
4. Run pre-commit checks
   - Use `prek` tool to run pre-commit hooks
   - Only check changed files (from base branch to HEAD)
   
5. Commit and push changes (if permissions allow)
   - Automatically commit formatted code
   - Or use pre-commit-ci lite action (for forks)

**Dual Path Strategy**:
- **Main Repository/With Permissions**: Directly commit changes and push
- **Forks/Dependabot**: Use pre-commit-ci lite action

**Quality Assurance**:
- Uses `pre-commit-alls-green` job for branch protection checks

---

## Project Management Workflows

### Issue Manager

**File**: `.github/workflows/issue-manager.yml`

**Triggers**:
- Scheduled: Daily at 17:21 UTC
- Issue comment created
- Issue labeled
- Pull Request labeled
- Manual trigger

**Description**:
Automatically manages the lifecycle of Issues and Pull Requests, closing or reminding based on labels and activity status.

**Only Runs in Main Repository**:
- `if: github.repository_owner == 'fastapi'`

**Management Rules**:

1. **answered label**:
   - Delay: 10 days (864000 seconds)
   - Message: Assuming the original need was handled, automatically closes

2. **waiting label**:
   - Delay: 30.4 days (2628000 seconds)
   - Reminder: 3 days before closing
   - Message: Waiting for original user response timeout, automatically closes

3. **invalid label**:
   - Delay: Immediate (0 seconds)
   - Message: Marked as invalid, closes immediately

4. **maybe-ai label**:
   - Delay: Immediate (0 seconds)
   - Message: Marked as potentially AI-generated, closes immediately

**Permissions Required**:
- `issues: write`
- `pull-requests: write`

---

### Labels

**File**: `.github/workflows/labeler.yml`

**Triggers**:
- Pull Request opened, synchronized, or reopened
- Pull Request label added or removed

**Description**:
Automatically adds labels to Pull Requests and validates that PRs have at least one required label.

**Main Steps**:

1. **labeler job**: Auto-add labels
   - Automatically adds appropriate labels based on file change paths
   - Only runs on non-label operations

2. **check-labels job**: Validate labels
   - Ensures PR has at least one of the following labels:
     - `breaking` - Breaking changes
     - `security` - Security related
     - `feature` - New feature
     - `bug` - Bug fix
     - `refactor` - Refactoring
     - `upgrade` - Dependency upgrade
     - `docs` - Documentation
     - `lang-all` - Multi-language
     - `internal` - Internal changes

**Permissions Required**:
- `contents: read`
- `pull-requests: write`

---

### Latest Changes

**File**: `.github/workflows/latest-changes.yml`

**Triggers**:
- Pull Request merged to `master` branch and closed
- Manual trigger (can specify PR number)

**Description**:
Automatically updates the `release-notes.md` file, adding merged PR information to release notes.

**Main Steps**:
1. Checkout code (using special token to allow commits)
2. Run latest-changes action
3. Automatically commit updated release-notes.md

**Configuration**:
- File: `./release-notes.md`
- Header: `## Latest Changes`
- End regex: `^## ` (next level 2 heading)
- Label header prefix: `### `

**Special Token**:
- Uses `LATEST_CHANGES` secret (instead of default GITHUB_TOKEN) to trigger subsequent CI

**Debugging Features**:
- Supports manual trigger with PR number specification
- Debug logging enabled

---

### Add to Project

**File**: `.github/workflows/add-to-project.yml`

**Triggers**:
- Pull Request created (all types)
- Issue opened or reopened

**Description**:
Automatically adds new Issues and Pull Requests to a GitHub Projects board for easier project management.

**Main Steps**:
1. Use `actions/add-to-project` action
2. Add to specified project board: `https://github.com/orgs/fastapi/projects/2`

**Permissions Required**:
- Uses `PROJECTS_TOKEN` secret (requires project management permissions)

---

### Conflict Detector

**File**: `.github/workflows/detect-conflicts.yml`

**Triggers**:
- Any push event
- Pull Request synchronized

**Description**:
Automatically detects if Pull Requests have merge conflicts and adds labels and comments as reminders.

**Main Steps**:
1. Check if PR has merge conflicts
2. If conflicts exist:
   - Add `conflicts` label
   - Comment reminder: "This pull request has a merge conflict that needs to be resolved."

**Action Used**:
- `eps1lon/actions-label-merge-conflict@v3`

**Permissions Required**:
- `contents: read`
- `pull-requests: write`

---

## Workflow Dependencies

```
Test Backend → Smokeshow
              ↓
         (generates coverage)
         
Playwright Tests (4 parallel shards) → Merge reports
                                      ↓
                                  HTML report

pre-commit → pre-commit-alls-green (branch protection)

Latest Changes ← PR merged
              ↓
        Update release-notes.md
```

---

## Secrets Configuration

The following GitHub Secrets need to be configured for the project:

### Deployment Secrets
- `DOMAIN_STAGING` - Staging environment domain
- `DOMAIN_PRODUCTION` - Production environment domain
- `STACK_NAME_STAGING` - Staging Docker Stack name
- `STACK_NAME_PRODUCTION` - Production Docker Stack name

### Application Configuration
- `SECRET_KEY` - Application secret key
- `FIRST_SUPERUSER` - Superuser account
- `FIRST_SUPERUSER_PASSWORD` - Superuser password
- `POSTGRES_PASSWORD` - Database password
- `SENTRY_DSN` - Sentry error tracking DSN

### SMTP Email Configuration
- `SMTP_HOST` - SMTP server address
- `SMTP_USER` - SMTP username
- `SMTP_PASSWORD` - SMTP password
- `EMAILS_FROM_EMAIL` - Sender email address

### CI/CD Tools
- `SMOKESHOW_AUTH_KEY` - Smokeshow authentication key
- `PRE_COMMIT` - Pre-commit token for commits
- `LATEST_CHANGES` - Latest Changes token for commits
- `PROJECTS_TOKEN` - GitHub Projects management token

---

## Branch Protection Requirements

Based on the workflow configuration, the following protection rules are recommended for the `master` branch:

1. **Required Status Checks**:
   - `pre-commit-alls-green`
   - `alls-green-playwright`
   - `test-backend`
   - `check-labels`

2. **Required Reviews**:
   - At least 1 approval

3. **Other Rules**:
   - Update branch before merging
   - Enforce rules for administrators

---

## Best Practices

1. **Local Development**:
   - Configure local pre-commit hooks to check code quality before pushing
   - Run `bash scripts/test.sh` to ensure local tests pass

2. **Pull Requests**:
   - Ensure appropriate labels are added (feature/bug/docs, etc.)
   - Wait for all CI checks to pass before merging
   - Resolve all merge conflicts

3. **Deployment**:
   - Staging: Push to `master` to automatically deploy to Staging
   - Production: Create a Release to automatically deploy to Production
   - Ensure all required secrets are configured

4. **Testing**:
   - Backend test coverage must be ≥ 90%
   - All Playwright tests must pass
   - Docker Compose configuration must be validated

---

## Troubleshooting

### Pre-commit Failures
- Check if code formatting meets standards
- Run `uvx prek run` locally to see specific issues

### Playwright Test Failures
- Review uploaded test report artifacts
- Use workflow_dispatch to enable tmate debugging mode

### Deployment Failures
- Check if secrets configuration is complete
- Verify self-hosted runners are online
- Review Docker Compose logs

### Insufficient Coverage
- Run `bash scripts/test.sh` to generate local coverage report
- Check `backend/htmlcov/index.html` for detailed information

---

**Last Updated**: 2026-02-17
