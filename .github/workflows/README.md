# GitHub CI/CD Workflows - Terraform Azure Deployment

## 📋 Overview

This project uses GitHub Actions to automate Terraform infrastructure deployment to Azure. The CI/CD pipeline consists of three main workflows that support **multi-environment parallel execution**:

1. **`validate.yaml`** - Validates and lints Terraform code across all environments (dev, test, stage, prod)
2. **`plan-deploy.yaml`** - Plans and applies Terraform changes across selected environments
3. **`destroy.yaml`** - Destroys infrastructure in selected environments (manual trigger with confirmation)

All workflows:
- ✅ Use **matrix strategy** for parallel execution across environments
- ✅ Support **environment-specific secrets** via GitHub Environments
- ✅ Support **environment-specific tfvars** files
- ✅ Authenticate with Azure using Service Principal credentials

### 🌍 Supported Environments

| Environment | Purpose | GitHub Environment | Destroy Environment |
|-------------|---------|-------------------|--------------------|
| `dev` | Development/Testing | `dev` | `dev-destroy` |
| `test` | Integration Testing | `test` | `test-destroy` |
| `stage` | Pre-production | `stage` | `stage-destroy` |
| `prod` | Production | `prod` | `prod-destroy` |

---

## ⚠️ PREREQUISITES & REQUIRED SETUP

**IMPORTANT:** Before any workflows can execute, you must complete this setup. Without these prerequisites, all workflows will fail with authentication errors.

### 🔐 Required GitHub Environment Secrets (8 Per Environment)

**IMPORTANT:** Secrets must be configured at the **GitHub Environment level**, not repository level. This allows each environment (dev, test, stage, prod) to have its own Azure credentials and backend configuration.

| Secret Name | Value | Source | Status |
|-------------|-------|--------|--------|
| `ARM_CLIENT_ID` | Service Principal Client ID | Azure Portal | ✅ Required |
| `ARM_CLIENT_SECRET` | Service Principal Client Secret | Azure Portal | ✅ Required |
| `ARM_SUBSCRIPTION_ID` | Azure Subscription ID | Azure Portal | ✅ Required |
| `ARM_TENANT_ID` | Azure Tenant/Directory ID | Azure Portal | ✅ Required |
| `BACKEND_RESOURCE_GROUP` | Resource Group for Terraform state | Azure Portal | ✅ Required |
| `BACKEND_STORAGE_ACCOUNT` | Storage Account name for state | Azure Portal | ✅ Required |
| `BACKEND_CONTAINER_NAME` | Container name in storage account | Azure Portal | ✅ Required |
| `BACKEND_KEY` | State file blob name | Any (e.g., `terraform.tfstate`) | ✅ Required |

### 🏗️ GitHub Environments Setup

You must create the following GitHub Environments in your repository settings:

**Required Environments (8 total):**

| Environment | Purpose | Protection Rules |
|-------------|---------|------------------|
| `dev` | Development deployments | Optional |
| `test` | Test deployments | Optional |
| `stage` | Staging deployments | Recommended: Required reviewers |
| `prod` | Production deployments | **Required: Required reviewers** |
| `dev-destroy` | Dev destruction approval | Recommended: Required reviewers |
| `test-destroy` | Test destruction approval | Recommended: Required reviewers |
| `stage-destroy` | Stage destruction approval | **Required: Required reviewers** |
| `prod-destroy` | Prod destruction approval | **Required: Required reviewers** |

**To create environments:**
1. Go to Repository → Settings → Environments
2. Click "New environment"
3. Enter environment name (e.g., `dev`)
4. Configure protection rules (required reviewers for prod)
5. Add all 8 secrets for that environment
6. Repeat for each environment

### 🗄️ Azure Storage Account Setup (Manual Prerequisite)

**IMPORTANT:** Before configuring the GitHub Secrets above, you must manually create an Azure Storage Account and Container to store Terraform state files. This cannot be managed by Terraform itself (chicken-and-egg problem).

#### Option 1: Create via Azure Portal

1. **Create Resource Group:**
   - Go to Azure Portal → Resource Groups → Create
   - Name: `rg-terraform-state-<environment>` (e.g., `rg-terraform-state-prod`)
   - Region: Choose your preferred region
   - Click **Create**

2. **Create Storage Account:**
   - Go to Azure Portal → Storage Accounts → Create
   - Resource Group: Select the one created above
   - Name: `st<project>tfstate<env>` (e.g., `stmyprojecttfstateprod`)
     - ⚠️ Must be globally unique, 3-24 characters, lowercase letters and numbers only
   - Region: Same as Resource Group
   - Performance: **Standard**
   - Redundancy: **LRS** (or **GRS** for production)
   - Click **Review + Create** → **Create**

3. **Create Blob Container:**
   - Go to the new Storage Account → **Containers** (under Data storage)
   - Click **+ Container**
   - Name: `tfstate` (recommended)
   - Public access level: **Private (no anonymous access)**
   - Click **Create**

#### Option 2: Create via Azure CLI

### 🔑 Service Principal Authentication Configuration

All workflows use this authentication payload:

```json
{
  "clientId": "${{ secrets.ARM_CLIENT_ID }}",
  "clientSecret": "${{ secrets.ARM_CLIENT_SECRET }}",
  "subscriptionId": "${{ secrets.ARM_SUBSCRIPTION_ID }}",
  "tenantId": "${{ secrets.ARM_TENANT_ID }}"
}
```

**What This Does:**
- Authenticates GitHub Actions runner to Azure
- Uses Service Principal (not interactive login)
- Scoped to specific Azure subscription
- Credentials stored securely (never exposed in logs)

### 💾 Terraform Backend Configuration

All workflows initialize Terraform with this backend config:

```bash
terraform init \\
  -backend-config=\"resource_group_name=${{ BACKEND_RESOURCE_GROUP }}\" \\
  -backend-config=\"storage_account_name=${{ BACKEND_STORAGE_ACCOUNT }}\" \\
  -backend-config=\"container_name=${{ BACKEND_CONTAINER_NAME }}\" \\
  -backend-config=\"key=${{ BACKEND_KEY }}\"
```

**Backend Type:** Azure Storage Account

**What This Does:**
- Stores Terraform state in Azure Storage Account (not locally)
- Enables team collaboration (prevents concurrent modifications)
- Provides audit trail of infrastructure changes
- Keeps sensitive data out of Git repository

### 📁 Environment-Specific Variables (Optional)

Create environment-specific Terraform variable files in an `environments/` directory:

```
project-root/
├── environments/
│   ├── dev.tfvars
│   ├── test.tfvars
│   ├── stage.tfvars
│   └── prod.tfvars
├── main.tf
├── variables.tf
└── ...
```

**Example `environments/dev.tfvars`:**
```hcl
environment     = "dev"
instance_count  = 1
instance_size   = "Standard_B1s"
enable_monitoring = false
```

**Example `environments/prod.tfvars`:**
```hcl
environment     = "prod"
instance_count  = 3
instance_size   = "Standard_D2s_v3"
enable_monitoring = true
```

Workflows automatically detect and use these files when present.

### ✅ Pre-Deployment Checklist

Before deploying, verify:

- [ ] All 8 GitHub Environments are created (dev, test, stage, prod + destroy variants)
- [ ] All 8 secrets are configured in **each** environment
- [ ] Service Principal has permissions to Azure subscription
- [ ] Azure Storage Account exists for Terraform state (per environment)
- [ ] Storage Account container exists for state files
- [ ] Protection rules configured for prod environments
- [ ] Branch protection rules configured on `main`

---

## 🔄 Complete Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│           TERRAFORM CI/CD PIPELINE (Multi-Environment)          │
└─────────────────────────────────────────────────────────────────┘

1️⃣  VALIDATE (Automatic on PR/Push) - Parallel Matrix Execution
    ┌─────────────────────────────────────────────────────────┐
    │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
    │  │   DEV   │ │  TEST   │ │  STAGE  │ │  PROD   │       │
    │  │ ─────── │ │ ─────── │ │ ─────── │ │ ─────── │       │
    │  │ TFLint  │ │ TFLint  │ │ TFLint  │ │ TFLint  │       │
    │  │ Format  │ │ Format  │ │ Format  │ │ Format  │       │
    │  │Validate │ │Validate │ │Validate │ │Validate │       │
    │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
    └─────────────────────────────────────────────────────────┘
    
    ✅ All environments pass → Triggers Plan
    ❌ Any failure → Blocks Merge

         ↓

2️⃣  PLAN (Auto-triggered after Validate) - Parallel Matrix Execution
    ┌─────────────────────────────────────────────────────────┐
    │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
    │  │   DEV   │ │  TEST   │ │  STAGE  │ │  PROD   │       │
    │  │ ─────── │ │ ─────── │ │ ─────── │ │ ─────── │       │
    │  │  Plan   │ │  Plan   │ │  Plan   │ │  Plan   │       │
    │  │ tfplan- │ │ tfplan- │ │ tfplan- │ │ tfplan- │       │
    │  │   dev   │ │  test   │ │  stage  │ │   prod  │       │
    │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
    └─────────────────────────────────────────────────────────┘
    
    ✅ Plans generated → Ready for Deploy
    📦 Artifacts: tfplan-dev, tfplan-test, tfplan-stage, tfplan-prod

         ↓

3️⃣  DEPLOY (Manual Trigger + Approval) - Parallel Matrix Execution
    ┌─────────────────────────────────────────────────────────┐
    │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
    │  │   DEV   │ │  TEST   │ │  STAGE  │ │  PROD   │       │
    │  │ ─────── │ │ ─────── │ │ ─────── │ │ ─────── │       │
    │  │ Apply   │ │ Apply   │ │ Apply   │ │ Apply   │       │
    │  │(auto)   │ │(auto)   │ │(approve)│ │(approve)│       │
    │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
    └─────────────────────────────────────────────────────────┘
    
    ✅ Success → Resources Created/Modified per environment

         ↓

💾  Azure Cloud (Multiple Subscriptions/Environments)
    ├─ dev subscription   → Dev infrastructure
    ├─ test subscription  → Test infrastructure
    ├─ stage subscription → Stage infrastructure
    └─ prod subscription  → Prod infrastructure


PARALLEL: DESTROY (Manual Trigger + Confirmation Required)
    ┌─────────────────────────────────────────────────────────┐
    │  Input: environments="dev,test" confirm_destroy="DESTROY"│
    │  ┌─────────┐ ┌─────────┐                               │
    │  │   DEV   │ │  TEST   │  (Only selected environments) │
    │  │ ─────── │ │ ─────── │                               │
    │  │ Destroy │ │ Destroy │                               │
    │  │(approve)│ │(approve)│                               │
    │  └─────────┘ └─────────┘                               │
    └─────────────────────────────────────────────────────────┘
    
    ⚠️  Requires typing "DESTROY" + environment approval
```

---

## 📊 Workflow 1: Validate (`validate.yaml`)

### ⚡ Trigger
- Pull Request to `main`, `develop`, or `release/**` branches
- Push to `main`, `develop`, or `release/**` branches
- Manual dispatch with environment selection

### 🎯 Purpose
Validate Terraform code syntax, format, and configuration across all environments in parallel.

### 📥 Inputs (Manual Dispatch)

| Input | Description | Default | Example |
|-------|-------------|---------|----------|
| `environments` | Comma-separated list or "all" | `all` | `dev,test` or `all` |

### 📋 Jobs

#### Job 1: Setup
Parses the environment input and prepares the matrix.

#### Job 2: Validate (Matrix: dev, test, stage, prod)

**Runs On:** `ubuntu-latest` (4 parallel jobs)

**Environment:** `${{ matrix.environment }}` (pulls environment-specific secrets)

**Steps per environment:**

| Step | Command | Purpose |
|------|---------|----------|
| 1. Display Info | Echo environment | Show which environment is being validated |
| 2. Checkout Code | `actions/checkout@v3` | Retrieve repository code |
| 3. Setup Terraform | `hashicorp/setup-terraform@v2` | Install Terraform v1.5.0 |
| 4. Debug Secrets | Check secret presence | Verify credentials are configured |
| 5. Azure Login | `azure/login@v2` | Authenticate using environment's Service Principal |
| 6. Setup TFLint | `terraform-linters/setup-tflint@v3` | Install Terraform linter tool |
| 7. Initialize TFLint | `tflint --init` | Download TFLint plugins |
| 8. Run TFLint | `tflint -f compact` | Check for best practices violations |
| 9. Terraform Init | `terraform init` | Initialize with environment's backend config |
| 10. Format Check | `terraform fmt -check -recursive` | Verify code formatting |
| 11. Validate | `terraform validate` | Validate configuration (uses env tfvars if present) |

### ✅ Success Criteria
- ✅ All environments pass validation in parallel
- ✅ All linting checks pass (no TFLint errors)
- ✅ Code formatting is consistent
- ✅ Terraform configuration is valid for all environments

### ❌ Failure Handling
- ❌ `fail-fast: false` - Other environments continue even if one fails
- ❌ PR cannot be merged until all environment validations pass
- ❌ Error messages displayed per environment in GitHub PR checks
- ❌ Developer must fix issues and push again

---

## 📊 Workflow 2: Plan & Deploy (`plan-deploy.yaml`)

### ⚡ Trigger
- **Automatic:** When `validate.yaml` completes successfully on `main`, `develop`, or `release/**` branches
- **Manual:** Workflow dispatch with environment selection and `run_deploy` flag

### 🎯 Purpose
Generate and execute Terraform execution plans across multiple environments in parallel.

### 📥 Inputs (Manual Dispatch)

| Input | Description | Required | Default | Example |
|-------|-------------|----------|---------|----------|
| `environments` | Comma-separated list or "all" | Yes | `dev` | `dev,test,stage,prod` or `all` |
| `run_deploy` | Whether to run the apply job | Yes | `false` | `true` |

### 📋 Jobs

#### Job 1: Setup
Parses the environment input and prepares the matrix for parallel execution.

#### Job 2: Plan (Matrix: selected environments)

**Runs On:** `ubuntu-latest` (up to 4 parallel jobs)

**Environment:** `${{ matrix.environment }}` (pulls environment-specific secrets)

**Steps per environment:**

| Step | Command | Output |
|------|---------|--------|
| 1. Display Info | Echo environment | Show which environment |
| 2. Checkout Code | `actions/checkout@v3` | Code retrieved |
| 3. Setup Terraform | `hashicorp/setup-terraform@v2` | Terraform v1.5.0 ready |
| 4. Azure Login | `azure/login@v2` | Authenticated with env credentials |
| 5. Terraform Init | `terraform init` | Backend initialized for environment |
| 6. Generate Plan | `terraform plan -var-file=environments/<env>.tfvars -out=tfplan` | Uses env-specific vars |
| 7. Convert to JSON | `terraform show -json tfplan > tfplan.json` | JSON format |
| 8. Display Summary | Parse and display changes | Human-readable output |
| 9. Upload Artifact | `actions/upload-artifact@v3` | `tfplan-<env>` stored for 5 days |

**Plan Output Example:**
```
📊 Terraform Plan Summary for dev:
==========================================
create azurerm_resource_group.example
create azurerm_app_service_plan.example
create azurerm_app_service.example
==========================================
```

**Artifacts Saved (per environment):**
- `tfplan-dev` - Dev environment plan
- `tfplan-test` - Test environment plan
- `tfplan-stage` - Stage environment plan
- `tfplan-prod` - Prod environment plan
- Retention: 5 days

#### Job 3: Apply/Deploy (Matrix: selected environments)

**Runs On:** `ubuntu-latest` (up to 4 parallel jobs)

**Depends On:** `setup` and `plan` jobs

**Environment:** `${{ matrix.environment }}` (may require approval for stage/prod)

**Condition:** Only runs if `run_deploy=true`

**Steps per environment:**

| Step | Command | Action |
|------|---------|--------|
| 1. Display Info | Echo environment | Show deployment target |
| 2. Checkout Code | `actions/checkout@v3` | Code retrieved |
| 3. Setup Terraform | `hashicorp/setup-terraform@v2` | Terraform ready |
| 4. Azure Login | `azure/login@v2` | Authenticated |
| 5. Terraform Init | `terraform init` | Backend initialized |
| 6. Download Plan | `actions/download-artifact@v3` | Retrieve `tfplan-<env>` |
| 7. Apply Plan | `terraform apply -auto-approve tfplan` | **Creates/Modifies Resources** |

### 🔒 Safety Features

- ✅ **Environment-Specific Approval:** Each environment can have its own approval requirements
- ✅ **Parallel Execution:** All environments planned/deployed simultaneously
- ✅ **Isolation:** Each environment uses its own secrets and state
- ✅ **Plan Reuse:** Uses stored plan per environment
- ✅ **Audit Trail:** All actions logged per environment
- ✅ **No Automatic Deploy:** Requires explicit `run_deploy=true`
- ✅ **Fail-Fast Disabled:** One environment failure doesn't stop others

### 📈 Workflow Diagram

```
Developer Push / Manual Trigger (environments="dev,test,stage,prod")
    ↓
Setup (Parse environments)
    ↓
┌─────────────────────────────────────────────────────────┐
│                 PARALLEL PLAN EXECUTION                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │Plan dev │ │Plan test│ │Plan stg │ │Plan prod│       │
│  │ tfplan- │ │ tfplan- │ │ tfplan- │ │ tfplan- │       │
│  │   dev   │ │  test   │ │  stage  │ │   prod  │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
└─────────────────────────────────────────────────────────┘
    ↓ (if run_deploy=true)
┌─────────────────────────────────────────────────────────┐
│                PARALLEL DEPLOY EXECUTION                │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │Deploy   │ │Deploy   │ │Deploy   │ │Deploy   │       │
│  │  dev    │ │  test   │ │  stage  │ │  prod   │       │
│  │ (auto)  │ │ (auto)  │ │(approve)│ │(approve)│       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
└─────────────────────────────────────────────────────────┘
    ↓
☁️ Azure (All environments updated)
```

---

## 📊 Workflow 3: Destroy (`destroy.yaml`)

### ⚡ Trigger
- **Manual Only** - Workflow dispatch with explicit confirmation required

### 🎯 Purpose
Completely tear down and destroy Azure infrastructure in selected environments.

### ⚠️ WARNING
This workflow is **IRREVERSIBLE**. Once executed, all Azure resources in the selected environments are permanently deleted. There is NO rollback.

### 📥 Inputs (Required)

| Input | Description | Required | Example |
|-------|-------------|----------|----------|
| `environments` | Comma-separated list or "all" | Yes | `dev,test` or `all` |
| `confirm_destroy` | Must type "DESTROY" exactly | Yes | `DESTROY` |

### 📋 Jobs

#### Job 1: Setup & Validate

- Validates that `confirm_destroy` equals "DESTROY" (fails immediately if not)
- Parses environment list into matrix format

#### Job 2: Destroy (Matrix: selected environments)

**Runs On:** `ubuntu-latest` (up to 4 parallel jobs)

**Environment:** `${{ matrix.environment }}-destroy` (requires approval)

**Steps per environment:**

| Step | Command | Action |
|------|---------|--------|
| 1. Display Warning | Echo warning | Show destruction warning |
| 2. Checkout Code | `actions/checkout@v3` | Code retrieved |
| 3. Setup Terraform | `hashicorp/setup-terraform@v2` | Terraform v1.5.0 ready |
| 4. Azure Login | `azure/login@v2` | Authenticated with env credentials |
| 5. Terraform Init | `terraform init` | Backend initialized for environment |
| 6. Destroy Plan | `terraform plan -destroy -var-file=environments/<env>.tfvars -out=tfdestroy` | Shows what will be deleted |
| 7. Execute Destroy | `terraform apply -auto-approve tfdestroy` | **🗑️ Deletes All Resources** |

### 🔒 Safety Features

- ⚠️ **Manual Trigger Only:** No automatic execution
- ⚠️ **Explicit Confirmation:** Must type "DESTROY" to proceed
- ⚠️ **Per-Environment Approval:** Each environment requires separate approval via `<env>-destroy` environment
- ⚠️ **Plan Generation:** Shows all resources before deletion
- ⚠️ **Parallel but Independent:** Each environment's destruction is independent
- ❌ **No Rollback:** Destruction is permanent

### 📈 Destruction Flow

```
Manual Workflow Trigger
    │
    ├─ Input: environments="dev,test"
    └─ Input: confirm_destroy="DESTROY"
    ↓
Setup & Validate
    ├─ Verify confirm_destroy == "DESTROY"
    └─ Parse environments → ["dev", "test"]
    ↓
┌─────────────────────────────────────────────────────────┐
│              PARALLEL DESTROY EXECUTION                 │
│  ┌─────────────────────┐ ┌─────────────────────┐       │
│  │    DEV-DESTROY      │ │   TEST-DESTROY      │       │
│  │ ─────────────────── │ │ ─────────────────── │       │
│  │ 1. Approval Gate    │ │ 1. Approval Gate    │       │
│  │ 2. Plan Destroy     │ │ 2. Plan Destroy     │       │
│  │ 3. Execute Destroy  │ │ 3. Execute Destroy  │       │
│  └─────────────────────┘ └─────────────────────┘       │
└─────────────────────────────────────────────────────────┘
    ↓
☁️ Selected Environments Destroyed
    ├─ dev: All resources deleted
    └─ test: All resources deleted
```

### 🛡️ Destroy Safeguards Summary

| Safeguard | Description |
|-----------|-------------|
| Confirmation Input | Must type "DESTROY" exactly |
| Environment Selection | Choose specific environments (not forced to destroy all) |
| Approval Gates | Each `<env>-destroy` environment can require reviewers |
| Parallel Independence | Failure in one environment doesn't affect others |
| Audit Trail | Full logging of what was destroyed |

---

## 🚀 How to Use These Workflows

### Scenario 1: Deploy New Infrastructure (All Environments)

```bash
# 1. Create feature branch
git checkout -b feature/add-app-service

# 2. Modify Terraform files
vi main.tf

# 3. Create/update environment-specific variables (optional)
vi environments/dev.tfvars
vi environments/prod.tfvars

# 4. Push changes
git push origin feature/add-app-service

# 5. Create Pull Request on GitHub
# → Validate workflow runs automatically for ALL environments in parallel
# → Review validation results for dev, test, stage, prod
# → Merge PR when all validations pass

# 6. Plan workflow triggers automatically
# → Plans generated for all environments in parallel
# → Review plans in GitHub Actions (tfplan-dev, tfplan-test, etc.)

# 7. Manually trigger deploy when ready
# → Go to Actions → Terraform Build & Deploy → Run workflow
# → Set environments: "all" (or specific: "dev,test")
# → Set run_deploy: true
# → Click "Run workflow"
# → Approve deployments per environment as needed
```

### Scenario 2: Deploy to Specific Environments Only

```bash
# 1. Go to GitHub Actions tab
# 2. Select "Terraform Build & Deploy" workflow
# 3. Click "Run workflow"
# 4. Enter inputs:
#    - environments: "dev,test"  (only deploy to dev and test)
#    - run_deploy: true
# 5. Click "Run workflow"
# 6. Plans generated for dev and test only
# 7. Deployments execute (approval may be required)
```

### Scenario 3: Destroy Specific Environments

```bash
# 1. Go to GitHub Actions tab
# 2. Select "Terraform Destroy" workflow
# 3. Click "Run workflow"
# 4. Enter inputs:
#    - environments: "dev,test"  (only destroy dev and test)
#    - confirm_destroy: "DESTROY"  (must type exactly)
# 5. Click "Run workflow"
# 6. Setup validates confirmation
# 7. Approval required for dev-destroy environment
# 8. Approval required for test-destroy environment
# 9. Destruction executes in parallel
# 10. ☁️ Dev and Test resources deleted
```

### Scenario 4: Destroy ALL Environments (Emergency)

```bash
# ⚠️ WARNING: This will destroy EVERYTHING
# 1. Go to GitHub Actions tab
# 2. Select "Terraform Destroy" workflow
# 3. Click "Run workflow"
# 4. Enter inputs:
#    - environments: "all"
#    - confirm_destroy: "DESTROY"
# 5. Click "Run workflow"
# 6. Approve each environment's destruction separately
# 7. All infrastructure destroyed
```

---

## ✅ Best Practices

### ✅ DO:
- ✅ Always review plan output for **each environment** before approving deploy
- ✅ Use feature branches for all changes
- ✅ Keep secrets secure (rotate periodically) **per environment**
- ✅ Test changes in dev environment first before deploying to stage/prod
- ✅ Document infrastructure changes in PR descriptions
- ✅ Never manually edit Azure resources (use Terraform only)
- ✅ Use specific Terraform version (v1.5.0) consistently
- ✅ Backup state before destroy operations
- ✅ Use environment-specific tfvars files for different configurations
- ✅ Set up required reviewers for stage and prod environments
- ✅ Deploy to dev/test first, then stage, then prod (progressive rollout)

### ❌ DON'T:
- ❌ Run destroy workflow without backup
- ❌ Commit secrets or credentials
- ❌ Manually edit Azure resources (breaks Terraform state)
- ❌ Skip validation before deploying
- ❌ Use `terraform destroy` locally
- ❌ Modify Terraform state files directly
- ❌ Deploy to prod without testing in lower environments first
- ❌ Use "all" for destroy without careful consideration
- ❌ Put secrets in repository-level settings (use environment-level)
- ❌ Share credentials between environments (each should have its own Service Principal)

---

## 🔍 Monitoring & Troubleshooting

### View Workflow Execution

1. Go to **Actions** tab in GitHub
2. Select workflow name (Validate, Plan/Deploy, or Destroy)
3. Click on run to view details
4. Check individual job logs

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Validation Fails** | Syntax errors in .tf files | Run `terraform fmt` and `terraform validate` locally |
| **401 Unauthorized** | Invalid credentials | Verify ARM_* secrets are set in the **GitHub Environment**, not repository level |
| **Plan Shows Drift** | Manual Azure changes | Revert manual changes to match Terraform |
| **Artifact Not Found** | Plan job failed | Check Plan job logs for the specific environment (e.g., `tfplan-dev`) |
| **Approve Button Missing** | Environment not configured | Create all 8 environments in repo settings (dev, test, stage, prod + destroy variants) |
| **Matrix Job Skipped** | Environment doesn't exist | Create the GitHub Environment and add all required secrets |
| **Destroy Fails Immediately** | Confirmation not typed correctly | Must type exactly "DESTROY" (case-sensitive) |
| **Only Some Environments Run** | Incorrect environments input | Use comma-separated list without spaces: `dev,test,stage,prod` or `all` |
| **Secrets Not Found** | Secrets at wrong level | Secrets must be in GitHub **Environment** settings, not repository secrets |

---

## 📊 Workflow Statistics

| Workflow | Trigger | Jobs | Parallelism | Duration | Approval |
|----------|---------|------|-------------|----------|----------|
| Validate | PR/Push | 2 (setup + matrix) | 4 environments | ~2-3 min | None |
| Plan/Deploy | Auto/Manual | 3 (setup + plan matrix + apply matrix) | 4 environments | ~5-10 min | Per environment |
| Destroy | Manual | 2 (setup + matrix) | 4 environments | ~5-10 min | Per `<env>-destroy` |

### Matrix Execution Details

| Environment | Validate | Plan | Deploy | Destroy |
|-------------|----------|------|--------|---------|
| `dev` | ✅ Parallel | ✅ Parallel | ✅ Parallel (auto) | ✅ Parallel (approval) |
| `test` | ✅ Parallel | ✅ Parallel | ✅ Parallel (auto) | ✅ Parallel (approval) |
| `stage` | ✅ Parallel | ✅ Parallel | ✅ Parallel (approval) | ✅ Parallel (approval) |
| `prod` | ✅ Parallel | ✅ Parallel | ✅ Parallel (approval) | ✅ Parallel (approval) |

---

## 📞 Support & Documentation

- **GitHub Actions Docs:** https://docs.github.com/en/actions
- **Terraform Docs:** https://www.terraform.io/docs
- **Azure Terraform Provider:** https://registry.terraform.io/providers/hashicorp/azurerm
- **TFLint:** https://github.com/terraform-linters/tflint

---

## 🔧 Quick Reference

### Environment Input Examples

| Input Value | Result |
|-------------|--------|
| `all` | Runs for dev, test, stage, prod |
| `dev` | Runs for dev only |
| `dev,test` | Runs for dev and test |
| `dev,test,stage,prod` | Same as "all" |
| `prod` | Runs for prod only |

### Required GitHub Environments

```
dev              → For dev deployments (secrets: ARM_*, BACKEND_*)
test             → For test deployments (secrets: ARM_*, BACKEND_*)
stage            → For stage deployments (secrets: ARM_*, BACKEND_*)
prod             → For prod deployments (secrets: ARM_*, BACKEND_*)
dev-destroy      → For dev destruction approval
test-destroy     → For test destruction approval
stage-destroy    → For stage destruction approval
prod-destroy     → For prod destruction approval
```

### Artifact Naming Convention

| Artifact | Description |
|----------|-------------|
| `tfplan-dev` | Terraform plan for dev environment |
| `tfplan-test` | Terraform plan for test environment |
| `tfplan-stage` | Terraform plan for stage environment |
| `tfplan-prod` | Terraform plan for prod environment |

---

**Last Updated:** 2024
**Version:** 2.0 (Multi-Environment Matrix Support)
**Status:** Active
