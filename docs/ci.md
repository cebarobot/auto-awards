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
    - [PR Labels](#pr-labels)
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
    - [PR Title Validation Failures](#pr-title-validation-failures)
    - [Deployment Failures](#deployment-failures)
    - [Insufficient Coverage](#insufficient-coverage)

---

## Deployment Workflows

### Deploy to Staging

**File**: `.github/workflows/deploy-staging.yml`

**Status**: ⚠️ Temporarily Disabled

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

**Status**: ⚠️ Temporarily Disabled

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
- Uses `playwright-alls-green` job for branch protection checks

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

### PR Labels

**File**: `.github/workflows/pr-labels.yml`

**Triggers**:
- Pull Request opened, synchronized, reopened, or edited

**Description**:
Validates that Pull Request titles follow the [Conventional Commits](https://www.conventionalcommits.org/) specification and automatically adds labels based on commit types.

**Main Steps**:
1. Validate PR title format against conventional commit pattern
2. Automatically add labels based on task type (feat, fix, docs, etc.)
3. Add scope labels if specified in PR title

**Supported Task Types**:
- `feat` → `feature` label - New features
- `fix` → `bug` label - Bug fixes
- `docs` → `docs` label - Documentation changes
- `test` → `test` label - Test updates
- `ci` → `internal` label - CI/CD changes
- `refactor` → `refactor` label - Code refactoring
- `perf` → `performance` label - Performance improvements
- `chore` → `internal` label - Maintenance tasks
- `revert` → `revert` label - Revert changes
- `build` → `internal` label - Build system changes
- `style` → `internal` label - Code style changes

**PR Title Format**:
- Basic: `<type>: <description>`
- With scope: `<type>(<scope>): <description>`
- Breaking change: `<type>!: <description>` (adds `breaking change` label)

**Examples**:
- ✅ `feat: add user authentication`
- ✅ `fix(api): resolve timeout issue`
- ✅ `docs: update installation guide`
- ✅ `feat!: redesign API endpoints` (breaking change)

**Features**:
- Automatic label assignment based on commit type
- Scope label support (e.g., `scope:api` for `fix(api): message`)
- Breaking change detection with `!` marker

**Action Used**:
- `ytanikin/pr-conventional-commits@1.5.1`

**Permissions Required**:
- `contents: read`
- `pull-requests: write`

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
```

---

## Secrets Configuration

The following GitHub Secrets need to be configured for the project:

### Deployment Secrets
- `DOMAIN_STAGING` - Staging environment domain (currently disabled)
- `DOMAIN_PRODUCTION` - Production environment domain (currently disabled)
- `STACK_NAME_STAGING` - Staging Docker Stack name (currently disabled)
- `STACK_NAME_PRODUCTION` - Production Docker Stack name (currently disabled)

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

---

## Branch Protection Requirements

Based on the workflow configuration, the following protection rules are recommended for the `master` branch:

1. **Required Status Checks**:
   - `pre-commit-alls-green`
   - `playwright-alls-green`
   - `test-backend`
   - `validate-pr-title` (PR Labels)

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
   - Follow Conventional Commits format for PR titles (e.g., `feat: add new feature`)
   - Labels will be automatically added based on commit type
   - Wait for all CI checks to pass before merging
   - Resolve all merge conflicts

3. **Deployment**:
   - Staging and Production deployments are currently disabled
   - To enable: Update the `if: false` condition in deployment workflow files
   - Ensure all required secrets are configured before enabling

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

### PR Title Validation Failures
- Ensure PR title follows Conventional Commits format: `<type>: <description>`
- Supported types: feat, fix, docs, test, ci, refactor, perf, chore, revert, build, style
- Examples: `feat: add login`, `fix(api): resolve timeout`, `docs: update README`

### Deployment Failures
- Deployments are currently disabled (`if: false` in workflow files)
- To enable: Remove or modify the `if: false` condition
- Check if secrets configuration is complete
- Verify self-hosted runners are online
- Review Docker Compose logs

### Insufficient Coverage
- Run `bash scripts/test.sh` to generate local coverage report
- Check `backend/htmlcov/index.html` for detailed information

---

**Last Updated**: 2026-02-17
