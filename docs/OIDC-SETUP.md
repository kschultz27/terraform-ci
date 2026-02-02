# Azure OIDC Setup for GitHub Actions

This guide provides step-by-step instructions for setting up Azure OIDC authentication for GitHub Actions Terraform workflows. OIDC eliminates the need for stored credentials (client secrets) by using federated identity.

## Architecture

```
Calling Repos                            terraform-ci              Azure
─────────────                            ────────────              ─────
healthcare-data-platform
terraform-infra             ──calls──▶   Reusable workflows
github-runners-aks                       (no Azure access)
        │
        │ OIDC token
        ▼
                                                                   Entra ID
                                                                      │
                                                           validates token
                                                                      │
                                                                      ▼
        ◀──────────────────── access token ──────────────────────────┘
        │
        │ Terraform runs
        ▼
                                                                   ARM API
```

**Key point:** Federated credentials are configured for the **calling repos** (healthcare-data-platform, terraform-infra, github-runners-aks), NOT for terraform-ci. The terraform-ci repo only contains reusable workflow definitions and never directly authenticates to Azure.

**Benefits:**
- No client secrets to rotate
- Short-lived tokens (valid only during workflow run)
- Audit trail of which repo/workflow accessed Azure
- Fine-grained access control per environment

---

## Prerequisites

- **Azure permissions:** Ability to create App Registrations and assign roles (typically requires Application Administrator or Global Administrator in Entra ID, plus Owner or User Access Administrator on target subscription)
- **GitHub permissions:** Admin access to the repositories you're configuring

---

## Step 1: Create App Registration

### Option A: Azure Portal

1. Navigate to **Microsoft Entra ID** > **App registrations**
2. Click **New registration**
3. Configure:
   - **Name:** `terraform-<org>-nonprod` (e.g., `terraform-acme-nonprod`)
   - **Supported account types:** "Accounts in this organizational directory only (Single tenant)"
   - **Redirect URI:** Leave blank
4. Click **Register**
5. **Record these values** (you'll need them later):
   - **Application (client) ID** — displayed on the Overview page
   - **Directory (tenant) ID** — displayed on the Overview page

### Option B: Azure CLI

```bash
# Set variables
APP_NAME="terraform-acme-nonprod"

# Create the app registration
az ad app create --display-name "$APP_NAME"

# Get the Application (client) ID
APP_ID=$(az ad app list --display-name "$APP_NAME" --query "[0].appId" -o tsv)
echo "Application (client) ID: $APP_ID"

# Get the Tenant ID
TENANT_ID=$(az account show --query "tenantId" -o tsv)
echo "Tenant ID: $TENANT_ID"
```

---

## Step 2: Create Service Principal

The service principal is the identity that Terraform will use to authenticate to Azure.

### Option A: Azure Portal

1. In your app registration, go to **Overview**
2. Note: A service principal is automatically created when you create role assignments (Step 3)
3. Alternatively, navigate to **Enterprise applications** and verify the app appears there

### Option B: Azure CLI

```bash
# Create service principal from the app registration
az ad sp create --id "$APP_ID"

# Get the Service Principal Object ID (needed for some role assignments)
SP_OBJECT_ID=$(az ad sp show --id "$APP_ID" --query "id" -o tsv)
echo "Service Principal Object ID: $SP_OBJECT_ID"
```

---

## Step 3: Assign Azure Roles (Least Privilege)

The service principal needs permissions to manage Azure resources. Follow the principle of **least privilege**: grant only the permissions needed, at the narrowest scope possible.

### Role Assignment Options (From Most to Least Restrictive)

Choose the approach that matches your security requirements:

#### Option 1: Resource Group Scoped (Most Restrictive)

Best for: Teams managing specific resource groups, not entire subscriptions.

| Scope | Role | Purpose |
|-------|------|---------|
| Resource Group(s) | **Contributor** | Create/modify/delete resources within the RG |
| Resource Group(s) | **User Access Administrator** | Assign roles to managed identities within the RG |
| Storage Account (tfstate) | **Storage Blob Data Contributor** | Read/write Terraform state only |

**Limitation:** Terraform cannot create new resource groups. Pre-create them or use Option 2.

#### Option 2: Subscription Scoped (Standard)

Best for: Platform teams managing diverse infrastructure across the subscription.

| Scope | Role | Purpose |
|-------|------|---------|
| Subscription | **Contributor** | Create/modify/delete resources anywhere in subscription |
| Subscription | **User Access Administrator** | Assign roles to managed identities (required for AKS, Key Vault) |
| Storage Account (tfstate) | **Storage Blob Data Contributor** | Read/write Terraform state only |

**Why not Owner?** Owner includes all Contributor permissions plus the ability to assign *any* role, including Owner itself. This creates privilege escalation risk.

#### Why User Access Administrator is Required

Terraform creates managed identities (e.g., AKS kubelet identity, Key Vault access connector) and must assign roles to them:

| Terraform Creates | Role Assignment Needed |
|-------------------|------------------------|
| AKS cluster | AcrPull on ACR for kubelet identity |
| AKS cluster | Private DNS Zone Contributor for private clusters |
| Key Vault | Key Vault Secrets User for workload identity |
| Databricks | Storage Blob Data Contributor for Unity Catalog |

Without User Access Administrator, these role assignments fail with `AuthorizationFailed`.

#### Why Not a Custom Role?

A custom role scoped to specific actions would require:
1. Predicting every `Microsoft.*/*/write` action Terraform might need
2. Updating the role whenever new resource types are added
3. Debugging cryptic "authorization failed" errors

For platform teams, **Contributor + User Access Administrator** is the pragmatic minimum. Restrict at the **subscription boundary** instead (separate SPs for nonprod vs prod).

---

### Terraform State Storage (Always Least Privilege)

The state storage account requires **only** `Storage Blob Data Contributor`:

| Role | What it Grants | Needed? |
|------|----------------|---------|
| Storage Blob Data Contributor | Read/write blobs | **Yes** — state files |
| Contributor | Manage storage account settings, keys, networking | **No** — SP doesn't configure the storage account |
| Storage Account Contributor | Manage storage account (not data) | **No** |

This limits blast radius: if the SP is compromised, the attacker can read/write state but cannot reconfigure the storage account, disable soft delete, or exfiltrate keys.

---

### Assigning Roles

#### Option A: Azure Portal (Subscription-Scoped)

**Subscription Roles:**

1. Navigate to **Subscriptions** > select your subscription (e.g., `Data-Dev-QA-001`)
2. Go to **Access control (IAM)**
3. Click **Add** > **Add role assignment**
4. **Role tab:** Select "Contributor"
5. **Members tab:**
   - Select "User, group, or service principal"
   - Click **Select members**
   - Search for your app name (e.g., `terraform-acme-nonprod`)
   - Select it and click **Select**
6. **Review + assign**
7. **Repeat** for "User Access Administrator" role

**Storage Account Role (for Terraform State):**

1. Navigate to **Storage accounts** > select your state storage account (e.g., `edmrunnerstfstate`)
2. Go to **Access control (IAM)**
3. Click **Add** > **Add role assignment**
4. **Role tab:** Select "Storage Blob Data Contributor"
5. **Members tab:** Select your app registration
6. **Review + assign**

#### Option B: Azure CLI (Subscription-Scoped)

```bash
# Set variables
SUBSCRIPTION_ID="<your-subscription-id>"  # e.g., Data-Dev-QA-001 subscription ID
STORAGE_ACCOUNT_NAME="edmrunnerstfstate"
STORAGE_RESOURCE_GROUP="edm-runners-tfstate-rg"

# Assign Contributor on subscription
az role assignment create \
  --assignee "$APP_ID" \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

# Assign User Access Administrator on subscription
az role assignment create \
  --assignee "$APP_ID" \
  --role "User Access Administrator" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

# Assign Storage Blob Data Contributor on state storage account (NOT subscription!)
STORAGE_ID=$(az storage account show \
  --name "$STORAGE_ACCOUNT_NAME" \
  --resource-group "$STORAGE_RESOURCE_GROUP" \
  --query "id" -o tsv)

az role assignment create \
  --assignee "$APP_ID" \
  --role "Storage Blob Data Contributor" \
  --scope "$STORAGE_ID"
```

#### Option C: Azure CLI (Resource Group-Scoped)

For tighter control, scope to specific resource groups:

```bash
# Assign Contributor on specific resource group only
az role assignment create \
  --assignee "$APP_ID" \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/edm-dp-dev-rg"

# Assign User Access Administrator on specific resource group only
az role assignment create \
  --assignee "$APP_ID" \
  --role "User Access Administrator" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/edm-dp-dev-rg"

# Repeat for each resource group the SP needs to manage
```

**Note:** With RG-scoped roles, Terraform cannot create resource groups. Either:
1. Pre-create resource groups manually, or
2. Grant subscription-level `Microsoft.Resources/subscriptions/resourceGroups/write` only

---

## Step 4: Add Federated Credentials

Federated credentials define which GitHub repos/workflows can authenticate as this service principal.

**Important:** Add credentials for the repos that **run Terraform** (healthcare-data-platform, terraform-infra, github-runners-aks), not for terraform-ci (which only contains reusable workflows).

### Credential Types Needed

Each Terraform repo needs credentials for:
1. **Pull requests** — for `terraform plan` on PRs
2. **Each environment** — for `terraform apply` (dev, qa, staging, prod)

### Federated Credential Configuration

| Field | Value |
|-------|-------|
| **Issuer** | `https://token.actions.githubusercontent.com` |
| **Audience** | `api://AzureADTokenExchange` |
| **Subject** | Varies by credential type (see below) |

### Subject Claim Formats

| Trigger | Subject Format | Example |
|---------|---------------|---------|
| Pull Request | `repo:<org>/<repo>:pull_request` | `repo:kschultz27/healthcare-data-platform:pull_request` |
| Environment | `repo:<org>/<repo>:environment:<env>` | `repo:kschultz27/healthcare-data-platform:environment:dev` |
| Branch | `repo:<org>/<repo>:ref:refs/heads/<branch>` | `repo:kschultz27/terraform-infra:ref:refs/heads/main` |

### Option A: Azure Portal

1. In your app registration, go to **Certificates & secrets**
2. Select **Federated credentials** tab
3. Click **Add credential**
4. Select **GitHub Actions deploying Azure resources**
5. Configure:
   - **Organization:** `kschultz27` (your GitHub org or username)
   - **Repository:** `healthcare-data-platform`
   - **Entity type:** Select based on trigger:
     - "Pull request" for PR-triggered plans
     - "Environment" for environment-gated applies
   - **GitHub environment name:** (only for Environment type) e.g., `dev`
   - **Name:** Descriptive name (e.g., `hdp-plan`, `hdp-dev`)
6. Click **Add**
7. **Repeat** for each credential needed

### Option B: Azure CLI

```bash
# Variables
APP_ID="<your-app-id>"
GITHUB_ORG="kschultz27"

# --- healthcare-data-platform credentials ---

# PR credential (for terraform plan)
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "hdp-plan",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GITHUB_ORG"'/healthcare-data-platform:pull_request",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Dev environment credential
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "hdp-dev",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GITHUB_ORG"'/healthcare-data-platform:environment:dev",
  "audiences": ["api://AzureADTokenExchange"]
}'

# QA environment credential
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "hdp-qa",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GITHUB_ORG"'/healthcare-data-platform:environment:qa",
  "audiences": ["api://AzureADTokenExchange"]
}'

# --- terraform-infra credentials ---

# PR credential
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "infra-plan",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GITHUB_ORG"'/terraform-infra:pull_request",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Dev environment credential
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "infra-dev",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GITHUB_ORG"'/terraform-infra:environment:dev",
  "audiences": ["api://AzureADTokenExchange"]
}'

# --- github-runners-aks credentials ---

# PR credential
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "runners-plan",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GITHUB_ORG"'/github-runners-aks:pull_request",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Dev environment credential
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "runners-dev",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GITHUB_ORG"'/github-runners-aks:environment:dev",
  "audiences": ["api://AzureADTokenExchange"]
}'
```

### List Existing Federated Credentials

```bash
az ad app federated-credential list --id "$APP_ID" --query "[].{name:name, subject:subject}" -o table
```

---

## Step 5: Configure GitHub Environments

GitHub Environments enable approval gates and environment-specific secrets/variables.

### Create Environments

1. In your GitHub repo, go to **Settings** > **Environments**
2. Click **New environment**
3. Create environments:

| Environment | Protection Rules | Purpose |
|-------------|-----------------|---------|
| `dev` | None | Auto-apply on merge |
| `qa` | Required reviewers: 1 | Manual approval |
| `staging` | Required reviewers: 1 | Pre-prod validation |
| `prod` | Required reviewers: 2 | Production gate |

### Configure Protection Rules (for qa/staging/prod)

1. Click on the environment name
2. Check **Required reviewers**
3. Add team members who can approve deployments
4. Optionally enable **Wait timer** (e.g., 5 minutes for prod)
5. Click **Save protection rules**

---

## Step 6: Configure GitHub Repository Variables

These variables are used by the reusable workflows to authenticate to Azure.

### Set Repository Variables

1. In your GitHub repo, go to **Settings** > **Secrets and variables** > **Actions**
2. Select **Variables** tab
3. Click **New repository variable**
4. Add these variables:

| Variable | Value | Description |
|----------|-------|-------------|
| `ARM_CLIENT_ID` | `<Application (client) ID>` | From Step 1 |
| `ARM_TENANT_ID` | `<Directory (tenant) ID>` | From Step 1 |
| `ARM_SUBSCRIPTION_ID` | `<Target subscription ID>` | The subscription to deploy to |

**Note:** These are **variables**, not secrets. They're not sensitive because:
- The client ID is not a credential (like a client secret would be)
- OIDC tokens only work from the specific repos/environments defined in federated credentials
- An attacker with only these IDs cannot authenticate

### Organization-Level Variables (Optional)

If multiple repos target the same subscription, set variables at the org level:

1. Go to **Organization settings** > **Secrets and variables** > **Actions**
2. Add the same variables
3. Configure repository access

---

## Step 7: Configure Terraform Backend

Update your Terraform configuration to use OIDC authentication.

### Provider Configuration

```hcl
# providers.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }

  backend "azurerm" {
    resource_group_name  = "edm-runners-tfstate-rg"
    storage_account_name = "edmrunnerstfstate"
    container_name       = "healthcare-data-platform"  # or your container
    key                  = "dev.tfstate"               # or your state file
    use_oidc             = true
  }
}

provider "azurerm" {
  features {}
  use_oidc = true
}
```

**Key settings:**
- `use_oidc = true` in both the backend and provider blocks
- No client_id, client_secret, or tenant_id in the config (these come from environment variables)

---

## Step 8: Add Caller Workflows

Add these workflows to each Terraform repo.

### `.github/workflows/terraform-plan.yml`

```yaml
name: Terraform Plan

on:
  pull_request:
    paths:
      - 'terraform/**'
      - '.github/workflows/terraform-*.yml'

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

### `.github/workflows/terraform-apply.yml`

```yaml
name: Terraform Apply

on:
  push:
    branches: [main]
    paths:
      - 'terraform/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - qa

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

---

## Verification

### Test OIDC Authentication

1. Create a test PR with a minor Terraform change
2. Verify the plan workflow:
   - Successfully authenticates to Azure
   - Posts plan output as PR comment
3. Merge the PR
4. Verify the apply workflow:
   - Successfully authenticates to Azure
   - Applies changes

### Verify Local Access is Blocked

```bash
# From a developer laptop (without az login)
cd terraform/environments/dev
terraform init
terraform plan
# Should fail with authentication error
```

### Check Audit Logs

**Azure:**
1. Go to **Microsoft Entra ID** > **Sign-in logs**
2. Filter by application name
3. Verify sign-ins match GitHub workflow runs

**GitHub:**
1. Go to repo **Settings** > **Actions** > **General**
2. Review workflow run history
3. Check which user/workflow triggered each run

---

## Troubleshooting

### "OIDC token request failed"

- Verify the federated credential subject matches exactly
- Check that the workflow has `id-token: write` permission
- Ensure the issuer URL is exactly `https://token.actions.githubusercontent.com`

### "No subscription found"

- Verify `ARM_SUBSCRIPTION_ID` is set correctly
- Ensure the service principal has role assignments on the subscription

### "Authorization failed"

- Check role assignments on the subscription
- For state backend errors, verify Storage Blob Data Contributor on the storage account

### "Subject does not match"

The subject claim must match exactly. Common mismatches:

| Issue | Fix |
|-------|-----|
| Wrong org/repo | Check GitHub org name and repo name |
| Environment vs branch | Use `environment:dev` not `ref:refs/heads/dev` |
| Case sensitivity | Subject claims are case-sensitive |

### Debugging Subject Claims

Add this step to your workflow to see the actual subject claim:

```yaml
- name: Debug OIDC
  run: |
    echo "GitHub context:"
    echo "  Repository: ${{ github.repository }}"
    echo "  Event: ${{ github.event_name }}"
    echo "  Ref: ${{ github.ref }}"
    echo "  Environment: ${{ github.environment }}"
```

---

## Quick Reference: Federated Credentials by Repo

### healthcare-data-platform

| Name | Subject |
|------|---------|
| `hdp-plan` | `repo:kschultz27/healthcare-data-platform:pull_request` |
| `hdp-dev` | `repo:kschultz27/healthcare-data-platform:environment:dev` |
| `hdp-qa` | `repo:kschultz27/healthcare-data-platform:environment:qa` |

### terraform-infra

| Name | Subject |
|------|---------|
| `infra-plan` | `repo:kschultz27/terraform-infra:pull_request` |
| `infra-dev` | `repo:kschultz27/terraform-infra:environment:dev` |
| `infra-qa` | `repo:kschultz27/terraform-infra:environment:qa` |

### github-runners-aks

| Name | Subject |
|------|---------|
| `runners-plan` | `repo:kschultz27/github-runners-aks:pull_request` |
| `runners-dev` | `repo:kschultz27/github-runners-aks:environment:dev` |
| `runners-qa` | `repo:kschultz27/github-runners-aks:environment:qa` |

---

## Security Considerations

### Least Privilege Checklist

| Area | Recommendation | Why |
|------|----------------|-----|
| **Role scope** | Scope to resource groups when possible, subscription when necessary | Limits blast radius if SP is compromised |
| **Role selection** | Contributor + User Access Administrator, never Owner | Owner allows privilege escalation |
| **State storage** | Storage Blob Data Contributor only (not Contributor) | SP doesn't need to reconfigure storage account |
| **Production separation** | Separate SP for prod subscription | Nonprod compromise doesn't affect prod |
| **Federated credentials** | Per-repo, per-environment credentials | Attacker who compromises one repo can't access others |

### Subscription Separation (Critical)

**Never** share a service principal across nonprod and prod subscriptions:

```
terraform-acme-nonprod          terraform-acme-prod
────────────────────            ────────────────────
Subscriptions:                  Subscriptions:
  • Data-Dev-QA-001               • Data-Stage-Prod

Federated Credentials:          Federated Credentials:
  • *:pull_request                • *:environment:staging
  • *:environment:dev             • *:environment:prod
  • *:environment:qa
```

This ensures:
- Compromised nonprod credentials cannot touch production
- Blast radius is limited to a single subscription tier
- Different approval chains for different risk levels

### Environment Gates

| Environment | Required Reviewers | Wait Timer | Deployment Branches |
|-------------|-------------------|------------|---------------------|
| `dev` | 0 (auto-apply) | None | `main` |
| `qa` | 1 | None | `main` |
| `staging` | 1 | None | `main` |
| `prod` | 2 | 5 minutes (recommended) | `main` |

### Audit Logging

Enable logging to detect unauthorized access:

1. **Entra ID Sign-in Logs:** Shows every OIDC authentication
2. **Azure Activity Log:** Shows every ARM API call from the SP
3. **GitHub Actions Logs:** Shows who triggered each workflow
4. **Blob Storage Logs:** Shows every state file read/write

Set up alerts for:
- Failed SP sign-ins (wrong federated credential)
- Activity outside expected GitHub Actions IP ranges
- State file access outside workflow runs

### Additional Security Controls

1. **No client secrets** — OIDC eliminates credential rotation and leak risk
2. **Short-lived tokens** — OIDC tokens expire after the workflow run
3. **Immutable audit trail** — Every apply is tied to a commit SHA and GitHub user
4. **Branch protection** — Require PR approval before merge → apply
5. **CODEOWNERS** — Require platform team approval for infrastructure changes
