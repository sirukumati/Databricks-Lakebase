# Databricks Lakebase

A production-ready Databricks application deployed using **Declarative Asset Bundles (DABs)** with **GitHub Actions** CI/CD pipelines and **Workload Identity Federation** for secure, token-less authentication.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
  - [Databricks Setup](#databricks-setup)
  - [GitHub Federation Policy Setup](#github-federation-policy-setup)
  - [GitHub Repository Setup](#github-repository-setup)
- [Architecture](#architecture)
- [CI/CD Workflow](#cicd-workflow)
- [Getting Started](#getting-started)
- [Asset Bundle Structure](#asset-bundle-structure)
- [Deployment](#deployment)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

## Overview

This repository contains a Databricks application built with:

- **Python** (51.8%) - Backend logic, jobs, and data processing
- **HTML** (48.2%) - Frontend dashboards and UI components
- **Databricks Asset Bundles (DABs)** - Infrastructure-as-code for Databricks resources
- **GitHub Actions** - Automated CI/CD for dev, staging, and production deployments
- **Workload Identity Federation** - Secure, token-less authentication via GitHub OIDC tokens

## Prerequisites

### Databricks Setup

1. **Databricks Workspace** (Premium or above)
   - Note your workspace URL: `https://<region>.databricks.com`
   - Get your workspace ID: Admin Console → Workspace settings
   - Get your Account ID: Account Console → Account settings

2. **Service Principal in Databricks**
   - Create a new service principal for CI/CD deployments
   - **Admin Console** → **Workspace admin settings** → **Service Principals** → **Create**
   - Assign appropriate workspace permissions (e.g., workspace admin or specific resource permissions)
   - **Note down the Service Principal ID** (you'll need this for the federation policy)

3. **Databricks CLI** (v0.19.0+)
   ```bash
   pip install databricks-cli
   ```

4. **Asset Bundles Support**
   - Ensure your workspace supports Databricks Asset Bundles
   - Verify by running: `databricks bundle validate --help`

### GitHub Federation Policy Setup

Databricks Workload Identity Federation allows GitHub Actions to authenticate securely using OIDC tokens **without storing long-lived secrets** in your repository.

#### Step 1: Create a Federation Policy in Databricks

A federation policy establishes a trust relationship between GitHub Actions and your Databricks service principal. This uses the official Databricks GitHub OIDC provider.

**Via Databricks CLI:**

```bash
# Authenticate with your Databricks workspace
databricks auth login --host https://<region>.databricks.com --token

# Create a federation policy
# Replace:
#   - <service-principal-id>: Your Databricks service principal ID
#   - <account-id>: Your Databricks account ID
#   - <github-org>: Your GitHub organization (e.g., sirukumati)
#   - <github-repo>: Your repository name (e.g., Databricks-Lakebase)

databricks service-principals federation-policy create \
  --service-principal-id <service-principal-id> \
  --policy '{
    "issuer": "https://token.actions.githubusercontent.com",
    "audience": "<account-id>",
    "subject_claim": "repo:<github-org>/<github-repo>:ref:refs/heads/main"
  }'
```

**Multiple Branches/Environments:**

If you want to support multiple branches (dev, staging, production), create separate federation policies or use wildcards:

```bash
# Allow any branch
databricks service-principals federation-policy create \
  --service-principal-id <service-principal-id> \
  --policy '{
    "issuer": "https://token.actions.githubusercontent.com",
    "audience": "<account-id>",
    "subject_claim": "repo:<github-org>/<github-repo>:*"
  }'

# Or allow specific environments
databricks service-principals federation-policy create \
  --service-principal-id <service-principal-id> \
  --policy '{
    "issuer": "https://token.actions.githubusercontent.com",
    "audience": "<account-id>",
    "subject_claim": "repo:<github-org>/<github-repo>:environment:production"
  }'
```

**Via Terraform:**

```hcl
resource "databricks_service_principal_federation_policy" "github_oidc" {
  service_principal_id = databricks_service_principal.github_ci.id
  
  policy {
    issuer   = "https://token.actions.githubusercontent.com"
    audience = var.databricks_account_id
    subject  = "repo:sirukumati/Databricks-Lakebase:ref:refs/heads/main"
  }
}
```

#### Step 2: Verify Federation Policy

List your created federation policies:

```bash
databricks service-principals federation-policy list \
  --service-principal-id <service-principal-id>
```

Expected output:
```json
{
  "policies": [
    {
      "issuer": "https://token.actions.githubusercontent.com",
      "audience": "<account-id>",
      "subject_claim": "repo:sirukumati/Databricks-Lakebase:ref:refs/heads/main"
    }
  ]
}
```

**Reference:** [Databricks GitHub Workload Identity Federation Documentation](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/auth/provider-github)

### GitHub Repository Setup

#### Step 1: Configure GitHub Secrets

Add the following secrets to your GitHub repository:

**Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret Name | Value | Description |
|---|---|---|
| `DATABRICKS_HOST` | `https://<region>.databricks.com` | Your Databricks workspace URL |
| `DATABRICKS_ACCOUNT_ID` | Your Databricks account ID | Required for OIDC token exchange |
| `DATABRICKS_CLIENT_ID` | Your service principal ID | The Databricks service principal ID |

#### Step 2: Configure GitHub Environments (Optional but Recommended)

Create separate environments for dev, staging, and production:

**Settings** → **Environments** → **New environment**

Create: `development`, `staging`, `production`

For each environment, optionally add:
- Environment-specific secrets
- Required reviewers for approval (recommended for production)
- Deployment branch restrictions

#### Step 3: Enable Workflow Permissions

**Settings** → **Actions** → **General** → **Workflow permissions**

- ✅ Enable "Read and write permissions"
- ✅ Enable "Allow GitHub Actions to create and approve pull requests"

#### Step 4: Configure OIDC Token Generation

GitHub Actions automatically generates OIDC tokens. No additional setup is needed, but ensure your workflows request the token:

```yaml
permissions:
  id-token: write
  contents: read
```

## Architecture

```
.
├── .github/
│   └── workflows/
│       ├── validate.yml          # Validate bundles on PR
│       ├── deploy-dev.yml         # Deploy to dev on merge to main
│       ├── deploy-staging.yml     # Deploy to staging on tag
│       └── deploy-production.yml  # Deploy to prod with approval
├── databricks.yml                 # Root bundle configuration
├── resources/
│   ├── jobs.yml                   # Job definitions
│   ├── pipelines.yml              # Data pipeline configurations
│   ├── dashboards.yml             # Dashboard definitions
│   └── notebooks/                 # Notebook files
├── src/
│   └── python/                    # Python source code
│       ├── jobs/                  # Job implementations
│       ├── pipelines/             # Pipeline logic
│       └── utils/                 # Utility functions
├── static/                        # HTML/frontend assets
├── tests/                         # Unit and integration tests
└── README.md
```

## CI/CD Workflow

### Workflow Overview

```
Push code
    ↓
[PR Validation] Lint, test, validate bundle
    ↓
[Code Review] Team review & approval
    ↓
[Merge to Main] Auto-deploy to development
    ↓
[Tag Release] Manual tag → Deploy to staging
    ↓
[Production Release] Manual approval → Deploy to production
```

### Validation Workflow (`.github/workflows/validate.yml`)

Runs on every pull request:

- Validates Databricks bundle syntax
- Runs unit tests
- Checks for configuration errors
- Lints Python code
- **No secrets needed** - validation is local

### Deployment Workflows with Federation

**Example: Deploy to Dev (.github/workflows/deploy-dev.yml)**

```yaml
name: Deploy to Development

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: development
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get Databricks OIDC Token
        uses: databricks/get-databricks-oidc-token@v1
        id: databricks-token
        with:
          client_id: ${{ secrets.DATABRICKS_CLIENT_ID }}
          databricks_host: ${{ secrets.DATABRICKS_HOST }}
      
      - name: Deploy Bundle
        uses: databricks/run-databricks-cli@v0
        with:
          databricks_host: ${{ secrets.DATABRICKS_HOST }}
          databricks_token: ${{ steps.databricks-token.outputs.databricks_token }}
          command: |
            bundle deploy --target development
```

**With Manual Approval (Production):**

```yaml
name: Deploy to Production

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get Databricks OIDC Token
        uses: databricks/get-databricks-oidc-token@v1
        id: databricks-token
        with:
          client_id: ${{ secrets.DATABRICKS_CLIENT_ID }}
          databricks_host: ${{ secrets.DATABRICKS_HOST }}
      
      - name: Deploy to Production
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ steps.databricks-token.outputs.databricks_token }}
        run: |
          databricks bundle deploy --target production
```

## Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/sirukumati/Databricks-Lakebase.git
cd Databricks-Lakebase
```

### 2. Install Dependencies

```bash
# Python dependencies
pip install -r requirements.txt

# Databricks CLI
pip install databricks-cli
```

### 3. Configure Local Databricks CLI (for local testing)

```bash
# Authenticate interactively
databricks auth login --host https://<region>.databricks.com

# Or set environment variables
export DATABRICKS_HOST=https://<region>.databricks.com
export DATABRICKS_TOKEN=<your-personal-token>
```

### 4. Validate the Bundle Locally

```bash
databricks bundle validate
```

### 5. Deploy Locally (for testing)

```bash
# Dry run - shows what would be deployed
databricks bundle deploy --no-lock --target development

# Actual deployment
databricks bundle deploy --target development
```

## Asset Bundle Structure

### `databricks.yml`

Root configuration file defining the bundle:

```yaml
bundle:
  name: databricks-lakebase
  description: Databricks application for lake management

targets:
  development:
    workspace:
      host: ${var.databricks_host}
      root_path: /Workspace/databricks-lakebase-dev

  staging:
    workspace:
      host: ${var.databricks_host}
      root_path: /Workspace/databricks-lakebase-staging

  production:
    workspace:
      host: ${var.databricks_host}
      root_path: /Workspace/databricks-lakebase-prod

variables:
  databricks_host:
    description: "Databricks workspace host"
    default: "https://eastus2.databricks.com"
```

### Resource Definitions

Define jobs, pipelines, dashboards, and notebooks in YAML:

```yaml
# resources/jobs.yml
resources:
  jobs:
    data_processing_job:
      name: "Data Processing Job"
      tasks:
        - task_key: "process_data"
          notebook_task:
            notebook_path: "../../src/python/jobs/process"
            source: "WORKSPACE"
```

## Deployment

### Manual Deployment (Local Development)

```bash
# Deploy to development
databricks bundle deploy --target development

# Deploy to staging
databricks bundle deploy --target staging

# Deploy to production
databricks bundle deploy --target production
```

### Automated Deployment (GitHub Actions with Federation)

Deployments are triggered automatically based on Git events:

1. **Push to main** → Auto-deploy to dev
2. **Create tag** → Deploy to staging
3. **Manual workflow trigger** → Deploy to production (with approval)

Monitor deployments in **GitHub** → **Actions** tab.

## Configuration

### Environment Variables

Set environment-specific variables in `.envrc` or GitHub environments:

```bash
# Development
DATABRICKS_HOST=https://eastus2.databricks.com
DATABRICKS_WORKSPACE_ID=123456789
DEPLOYMENT_TARGET=development

# Staging
DATABRICKS_HOST=https://eastus2.databricks.com
DATABRICKS_WORKSPACE_ID=987654321
DEPLOYMENT_TARGET=staging

# Production
DATABRICKS_HOST=https://eastus2.databricks.com
DATABRICKS_WORKSPACE_ID=555666777
DEPLOYMENT_TARGET=production
```

### Bundle Variables

Use `databricks.yml` variables for dynamic configuration:

```bash
# Deploy with custom variables
databricks bundle deploy \
  --target production \
  --var databricks_host=https://eastus2.databricks.com
```

## Troubleshooting

### Bundle Validation Fails

```bash
# Check for syntax errors
databricks bundle validate --fail-on-error

# Review error messages and fix YAML
cat databricks.yml
```

### GitHub Actions Authentication Errors

**Issue:** `OIDC token not found` or `401 Unauthorized`

**Solutions:**

1. Verify federation policy exists:
   ```bash
   databricks service-principals federation-policy list \
     --service-principal-id <service-principal-id>
   ```

2. Check GitHub Actions permissions:
   - Settings → Actions → General → Workflow permissions → Enable "Read and write"

3. Verify federation policy subject matches your repo/branch:
   ```bash
   # Should be: repo:sirukumati/Databricks-Lakebase:ref:refs/heads/main
   ```

4. Ensure service principal has workspace permissions:
   ```bash
   databricks service-principals get \
     --service-principal-id <service-principal-id>
   ```

### Workspace Deployment Issues

```bash
# Check workspace connectivity
databricks workspace list-directories --path /

# Verify CLI authentication
databricks auth verify
```

### Updating Federation Policy

```bash
# List policies
databricks service-principals federation-policy list \
  --service-principal-id <service-principal-id>

# Delete old policy (if needed)
databricks service-principals federation-policy delete \
  --service-principal-id <service-principal-id> \
  --policy-id <policy-id>

# Create new policy
databricks service-principals federation-policy create \
  --service-principal-id <service-principal-id> \
  --policy '{...}'
```

## Testing

### Unit Tests

```bash
pytest tests/unit/
```

### Integration Tests (requires Databricks access)

```bash
export DATABRICKS_HOST=https://<region>.databricks.com
export DATABRICKS_TOKEN=<token>

pytest tests/integration/
```

## Security Best Practices

1. **Never commit secrets** - Use GitHub Secrets
2. **Use Workload Identity Federation** - No long-lived tokens stored
3. **Restrict federation policies** - Use specific branches/environments
4. **Limit service principal permissions** - Grant minimal required access
5. **Enable environment protection** - Require approval for production deployments
6. **Audit access** - Monitor Databricks audit logs for CI/CD activities

## Resources

### Official Databricks Documentation

- [Databricks Asset Bundles](https://docs.databricks.com/en/dev-tools/bundles/index.html)
- [GitHub Workload Identity Federation](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/auth/provider-github)
- [Configure Federation Policy](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/auth/oauth-federation-policy)
- [CI/CD on Databricks](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/ci-cd/github)

### GitHub Actions

- [GitHub OIDC Provider Documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Databricks Asset Bundles Deploy Action](https://github.com/marketplace/actions/databricks-asset-bundles-deploy)

### Community & Examples

- [Databricks Community: CI/CD with Asset Bundles](https://community.databricks.com/t5/community-articles/ci-cd-on-databricks-with-asset-bundles-dabs-and-github-actions/td-p/149565)
- [Example Deployment Repository](https://github.com/evanaze/dbx-asset-bundle-deployment)

## Support

For issues, questions, or contributions:

1. Open an issue on [GitHub Issues](https://github.com/sirukumati/Databricks-Lakebase/issues)
2. Check [Databricks Documentation](https://docs.databricks.com)
3. Visit [Databricks Community Forum](https://community.databricks.com)
4. Contact Databricks Support for workspace-level issues

---

**Last Updated:** August 2026  
**Maintainer:** sirukumati  
**License:** See LICENSE file
