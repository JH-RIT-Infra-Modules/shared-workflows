# GitHub CI/CD Workflows - Terraform Azure Deployment

## 📋 Overview

This project uses GitHub Actions to automate Terraform infrastructure deployment to Azure. The CI/CD pipeline consists of three main workflows:

1. **`validate.yaml`** - Validates and lints Terraform code on PR/Push to main
2. **`plan-deploy.yaml`** - Plans and applies Terraform changes after validation
3. **`destroy.yaml`** - Destroys infrastructure when needed (manual trigger)

All workflows authenticate with Azure using Service Principal credentials stored as GitHub Secrets.

---

## ⚠️ PREREQUISITES & REQUIRED SETUP

**IMPORTANT:** Before any workflows can execute, you must complete this setup. Without these prerequisites, all workflows will fail with authentication errors.

### 🔐 Required GitHub Secrets (8 Total)

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

### ✅ Pre-Deployment Checklist

Before deploying, verify:

- [ ] All 8 GitHub Secrets are configured correctly
- [ ] Service Principal has permissions to Azure subscription
- [ ] Azure Storage Account exists for Terraform state
- [ ] Storage Account container exists for state files
- [ ] GitHub environments are created (`production`, `production-destroy`)
- [ ] Branch protection rules configured on `main`

---

## 🔄 Complete Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    TERRAFORM CI/CD PIPELINE                     │
└─────────────────────────────────────────────────────────────────┘

1️⃣  VALIDATE (Automatic on PR/Push)
    ├─ TFLint Check
    ├─ Format Check
    └─ Validate Configuration
    
    ✅ Success → Triggers Plan
    ❌ Failure → Blocks Merge

         ↓

2️⃣  PLAN (Auto-triggered after Validate)
    ├─ Generate Terraform Plan
    ├─ Convert to JSON
    ├─ Display Summary
    └─ Store as Artifact
    
    ✅ Success → Ready for Deploy
    ❌ Failure → Shows errors

         ↓

3️⃣  DEPLOY (Manual Approval Required)
    ├─ Download Plan
    ├─ Apply Terraform Changes
    └─ Update Azure Infrastructure
    
    ✅ Success → Resources Created/Modified
    ❌ Failure → Rollback not available

         ↓

💾  Azure Cloud
    └─ Infrastructure deployed via Terraform


PARALLEL: DESTROY (Manual Trigger Only)
    ├─ Generate Destroy Plan
    ├─ Requires Approval
    └─ Delete All Resources
    
    ⚠️  Irreversible - No rollback
```

---

## 📊 Workflow 1: Validate (`validate.yaml`)

### ⚡ Trigger
- Pull Request to `main` branch
- Push to `main` branch

### 🎯 Purpose
Validate Terraform code syntax, format, and configuration before any changes are applied.

### 📋 Job: Validate

**Runs On:** `ubuntu-latest`

**Steps:**

| Step | Command | Purpose |
|------|---------|----------|
| 1. Checkout Code | `actions/checkout@v3` | Retrieve repository code |
| 2. Setup Terraform | `hashicorp/setup-terraform@v2` | Install Terraform v1.5.0 |
| 3. Azure Login | `azure/login@v1` | Authenticate using Service Principal |
| 4. Setup TFLint | `terraform-linters/setup-tflint@v3` | Install Terraform linter tool |
| 5. Initialize TFLint | `tflint --init` | Download TFLint plugins |
| 6. Run TFLint | `tflint -f compact` | Check for best practices violations |
| 7. Terraform Init | `terraform init` | Initialize Terraform with Azure backend |
| 8. Format Check | `terraform fmt -check -recursive` | Verify code formatting is correct |
| 9. Validate | `terraform validate` | Validate Terraform syntax and structure |

### ✅ Success Criteria
- ✅ All linting checks pass (no TFLint errors)
- ✅ Code formatting is consistent
- ✅ Terraform configuration is valid
- ✅ No syntax errors detected

### ❌ Failure Handling
- ❌ If any step fails, workflow stops immediately
- ❌ PR cannot be merged until validation passes
- ❌ Error messages displayed in GitHub PR checks
- ❌ Developer must fix issues and push again

---

## 📊 Workflow 2: Plan & Deploy (`plan-deploy.yaml`)

### ⚡ Trigger
- **Automatic:** When `validate.yaml` completes successfully on `main` branch
- **Manual:** Workflow dispatch with `run_deploy=true` flag

### 🎯 Purpose
Generate and execute Terraform execution plan to deploy changes to Azure.

### 📋 Job 1: Plan

**Runs On:** `ubuntu-latest`

**Condition:** Only runs if validate workflow succeeded AND branch is main

**Steps:**

| Step | Command | Output |
|------|---------|--------|
| 1. Checkout Code | `actions/checkout@v3` | Code retrieved |
| 2. Setup Terraform | `hashicorp/setup-terraform@v2` | Terraform v1.5.0 ready |
| 3. Azure Login | `azure/login@v1` | Authenticated |
| 4. Terraform Init | `terraform init` | Backend initialized |
| 5. Generate Plan | `terraform plan -out=tfplan` | Binary: `tfplan` |
| 6. Convert to JSON | `terraform show -json tfplan > tfplan.json` | JSON: `tfplan.json` |
| 7. Display Summary | Parse and display changes | Human-readable output |
| 8. Upload Artifact | `actions/upload-artifact@v3` | Stored for 5 days |

**Plan Output Example:**
```
Terraform Plan Summary:
  + azurerm_resource_group.example (create)
  + azurerm_app_service_plan.example (create)
  + azurerm_app_service.example (create)

Plan: 3 to add, 0 to change, 0 to destroy.
```

**Artifacts Saved:**
- `tfplan` (binary plan file)
- `tfplan.json` (machine-readable format)
- Retention: 5 days

### 📋 Job 2: Apply/Deploy

**Runs On:** `ubuntu-latest`

**Depends On:** `plan` job (must succeed first)

**Environment:** `production` (requires approval)

**Condition:** Only runs if manually triggered with `run_deploy=true`

**Steps:**

| Step | Command | Action |
|------|---------|--------|
| 1. Checkout Code | `actions/checkout@v3` | Code retrieved |
| 2. Setup Terraform | `hashicorp/setup-terraform@v2` | Terraform ready |
| 3. Azure Login | `azure/login@v1` | Authenticated |
| 4. Terraform Init | `terraform init` | Backend initialized |
| 5. Download Plan | `actions/download-artifact@v3` | Retrieve `tfplan` |
| 6. Apply Plan | `terraform apply -auto-approve tfplan` | **Creates/Modifies Resources** |

### 🔒 Safety Features

- ✅ **Approval Gate:** Requires manual approval in `production` environment
- ✅ **Separation of Concerns:** Plan and Apply are separate jobs
- ✅ **Plan Reuse:** Uses stored plan (prevents unintended changes)
- ✅ **Audit Trail:** All actions logged in GitHub
- ✅ **No Automatic Deploy:** Requires explicit manual trigger

### 📈 Workflow Diagram

```
Developer Push
    ↓
Validate ✅
    ↓
Plan (Auto-triggered)
    ├─ Generate tfplan ✅
    ├─ Convert to JSON ✅
    └─ Upload Artifact ✅
    ↓
Deploy (Awaiting Manual Approval)
    ├─ Review & Approve
    ├─ Download Plan
    └─ Apply → Azure ✅
```

---

## 📊 Workflow 3: Destroy (`destroy.yaml`)

### ⚡ Trigger
- **Manual Only** - Workflow dispatch (no automatic triggers)

### 🎯 Purpose
Completely tear down and destroy all Azure infrastructure managed by Terraform.

### ⚠️ WARNING
This workflow is **IRREVERSIBLE**. Once executed, all Azure resources are permanently deleted. There is NO rollback.

### 📋 Job: Destroy

**Runs On:** `ubuntu-latest`

**Environment:** `production-destroy` (requires approval)

**Steps:**

| Step | Command | Action |
|------|---------|--------|
| 1. Checkout Code | `actions/checkout@v3` | Code retrieved |
| 2. Setup Terraform | `hashicorp/setup-terraform@v2` | Terraform v1.5.0 ready |
| 3. Azure Login | `azure/login@v1` | Authenticated |
| 4. Terraform Init | `terraform init` | Backend initialized |
| 5. Destroy Plan | `terraform plan -destroy -out=tfdestroy` | Shows what will be deleted |
| 6. Execute Destroy | `terraform apply -auto-approve tfdestroy` | **🗑️ Deletes All Resources** |

### 🔒 Safety Features

- ⚠️ **Manual Trigger Only:** No automatic execution
- ⚠️ **Approval Required:** Must approve `production-destroy` environment
- ⚠️ **Plan Generation:** Shows all resources before deletion
- ⚠️ **Explicit Action:** Requires deliberate workflow dispatch
- ❌ **No Rollback:** Destruction is permanent

### 📈 Destruction Flow

```
Manual Workflow Trigger
    ↓
Generate Destroy Plan
    ├─ Shows all resources to be deleted
    └─ Lists deletion order
    ↓
Approval Required
    ├─ Review `production-destroy` environment
    ├─ Confirm intention
    └─ Click \"Approve and deploy\"
    ↓
Execute Destruction
    ├─ Delete all resources in order
    ├─ Update Terraform state
    └─ Complete
    ↓
☁️ Azure Infrastructure Destroyed
    └─ Resource Group may remain (if not managed by Terraform)
```

---

## 🚀 How to Use These Workflows

### Scenario 1: Deploy New Infrastructure

```bash
# 1. Create feature branch
git checkout -b feature/add-app-service

# 2. Modify Terraform files
vi main.tf

# 3. Push changes
git push origin feature/add-app-service

# 4. Create Pull Request on GitHub
# → Validate workflow runs automatically
# → Review validation results
# → Merge PR

# 5. Push triggers Plan workflow
# → Review plan in GitHub Actions
# → Approve deployment when ready

# 6. Deploy workflow runs
# → Changes applied to Azure
```

### Scenario 2: Destroy All Infrastructure

```bash
# 1. Go to GitHub Actions tab
# 2. Select \"Terraform Destroy\" workflow
# 3. Click \"Run workflow\"
# 4. Confirm branch: main
# 5. Click \"Run workflow\"
# 6. Wait for approval prompt
# 7. Review action details
# 8. Click \"Approve and deploy\"
# 9. Destruction begins
# 10. ☁️ Azure resources deleted
```

---

## ✅ Best Practices

### ✅ DO:
- ✅ Always review plan output before approving deploy
- ✅ Use feature branches for all changes
- ✅ Keep secrets secure (rotate periodically)
- ✅ Test changes in dev environment first
- ✅ Document infrastructure changes in PR descriptions
- ✅ Never manually edit Azure resources (use Terraform only)
- ✅ Use specific Terraform version (v1.5.0) consistently
- ✅ Backup state before destroy operations

### ❌ DON'T:
- ❌ Run destroy workflow without backup
- ❌ Commit secrets or credentials
- ❌ Manually edit Azure resources (breaks Terraform state)
- ❌ Skip validation before deploying
- ❌ Use `terraform destroy` locally
- ❌ Modify Terraform state files directly
- ❌ Use wildcards in destroy confirmations
- ❌ Deploy to production without approval

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
| **401 Unauthorized** | Invalid credentials | Verify all ARM_* secrets are correct |
| **Plan Shows Drift** | Manual Azure changes | Revert manual changes to match Terraform |
| **Artifact Not Found** | Plan job failed | Check Plan job logs for errors |
| **Approve Button Missing** | Environment not configured | Add environment in repo settings |

---

## 📊 Workflow Statistics

| Workflow | Trigger | Jobs | Duration | Approval |
|----------|---------|------|----------|----------|
| Validate | PR/Push | 1 | ~2-3 min | None |
| Plan/Deploy | Auto/Manual | 2 | ~5-10 min | Production env |
| Destroy | Manual | 1 | ~5-10 min | Production-destroy env |

---

## 📞 Support & Documentation

- **GitHub Actions Docs:** https://docs.github.com/en/actions
- **Terraform Docs:** https://www.terraform.io/docs
- **Azure Terraform Provider:** https://registry.terraform.io/providers/hashicorp/azurerm
- **TFLint:** https://github.com/terraform-linters/tflint

---

**Last Updated:** 2024
**Version:** 1.0
**Status:** Active
