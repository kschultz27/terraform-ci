# terraform-ci

Reusable GitHub Actions workflows for Terraform CI/CD with Azure OIDC authentication.

## Available Workflows

| Workflow | Purpose |
|----------|---------|
| `terraform-plan.yml` | Run plan on PRs, post output as comment |
| `terraform-apply.yml` | Run plan + apply with environment gates |
| `terraform-validate.yml` | Validate-only (for local state configs like bootstrap) |

## Prerequisites

> **Detailed Setup Guide:** See [docs/OIDC-SETUP.md](docs/OIDC-SETUP.md) for step-by-step Azure Portal and CLI instructions.

### 1. Azure App Registration with Federated Credentials

Create an app registration with federated credentials for each **calling repo** (not terraform-ci):

| Credential | Subject |
|------------|---------|
| `<repo>-plan` | `repo:<org>/<repo>:pull_request` |
| `<repo>-<env>` | `repo:<org>/<repo>:environment:<env>` |

All credentials use:
- Issuer: `https://token.actions.githubusercontent.com`
- Audience: `api://AzureADTokenExchange`

### 2. GitHub Repository Variables

Set these as repository variables (not secrets) in each calling repo:

| Variable | Value |
|----------|-------|
| `ARM_CLIENT_ID` | Application (client) ID |
| `ARM_TENANT_ID` | Azure tenant ID |
| `ARM_SUBSCRIPTION_ID` | Target subscription ID |

### 3. GitHub Environments

Create environments in each calling repo to enable approval gates:

| Environment | Protection Rules |
|-------------|-----------------|
| `dev` | None (auto-apply) |
| `qa` | 1 required reviewer |
| `prod` | 2 required reviewers |

## Usage

### Terraform Plan (PRs)

```yaml
# .github/workflows/terraform-plan.yml
name: Terraform Plan

on:
  pull_request:
    paths:
      - 'terraform/**'

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  plan-dev:
    uses: kschultz27/terraform-ci/.github/workflows/terraform-plan.yml@main
    with:
      working_directory: terraform/environments/dev
      terraform_version: "1.6.0"
    secrets: inherit
```

### Terraform Apply (merge to main)

```yaml
# .github/workflows/terraform-apply.yml
name: Terraform Apply

on:
  push:
    branches: [main]
    paths:
      - 'terraform/**'
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, qa, prod]

permissions:
  id-token: write
  contents: read

jobs:
  apply:
    uses: kschultz27/terraform-ci/.github/workflows/terraform-apply.yml@main
    with:
      environment: ${{ github.event.inputs.environment || 'dev' }}
      working_directory: terraform/environments/${{ github.event.inputs.environment || 'dev' }}
      terraform_version: "1.6.0"
    secrets: inherit
```

### Terraform Validate (local state configs)

```yaml
# For bootstrap or other configs that use local state
jobs:
  validate:
    uses: kschultz27/terraform-ci/.github/workflows/terraform-validate.yml@main
    with:
      working_directory: bootstrap
      terraform_version: "1.6.0"
```

## Workflow Inputs

### terraform-plan.yml

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `working_directory` | Yes | - | Path to terraform config |
| `terraform_version` | No | `1.6.0` | Terraform version |
| `var_file` | No | `terraform.tfvars` | tfvars file name |
| `runs_on` | No | `arc-runner-set` | Runner label |

### terraform-apply.yml

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `environment` | Yes | - | GitHub Environment name |
| `working_directory` | Yes | - | Path to terraform config |
| `terraform_version` | No | `1.6.0` | Terraform version |
| `var_file` | No | `terraform.tfvars` | tfvars file name |
| `runs_on` | No | `arc-runner-set` | Runner label |

### terraform-validate.yml

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `working_directory` | Yes | - | Path to terraform config |
| `terraform_version` | No | `1.6.0` | Terraform version |
| `runs_on` | No | `arc-runner-set` | Runner label |

## Architecture

```
caller repo                          terraform-ci
───────────                          ────────────
.github/workflows/
├── terraform-plan.yml    ──calls──▶  terraform-plan.yml
└── terraform-apply.yml   ──calls──▶  terraform-apply.yml
```

Changes to CI/CD logic are made once in `terraform-ci` and automatically apply to all caller repos.
