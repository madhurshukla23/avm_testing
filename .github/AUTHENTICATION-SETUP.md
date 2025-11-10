# Azure OIDC Authentication Setup for GitHub Actions

This document provides detailed steps to configure Azure OpenID Connect (OIDC) authentication for GitHub Actions workflows, enabling secure, passwordless authentication to Azure without storing credentials.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Azure AD App Registration](#azure-ad-app-registration)
3. [Configure Federated Identity Credentials](#configure-federated-identity-credentials)
4. [Assign Azure Permissions](#assign-azure-permissions)
5. [Setup Terraform State Storage](#setup-terraform-state-storage)
6. [Configure GitHub Secrets](#configure-github-secrets)
7. [Verify Configuration](#verify-configuration)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- **Azure Subscription** with appropriate permissions (Owner or User Access Administrator)
- **GitHub Repository** admin access
- **Azure CLI** installed locally ([Install Guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli))
- **Bash/PowerShell** terminal access

---

## Azure AD App Registration

### Step 1: Login to Azure

```bash
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
```

### Step 2: Create Azure AD Application

Create an Azure AD application that will represent GitHub Actions:

```bash
az ad app create --display-name "GitHub-Actions-OIDC-AVM-Testing"
```

**Output:** Note down the `appId` (also called Client ID)

```json
{
  "appId": "12345678-1234-1234-1234-123456789abc",
  "displayName": "GitHub-Actions-OIDC-AVM-Testing",
  ...
}
```

**Save this value** - you'll need it as `AZURE_CLIENT_ID`

### Step 3: Create Service Principal

Create a service principal for the application:

```bash
az ad sp create --id <APP_ID_FROM_STEP_2>
```

**Alternative:** Get App ID from existing app

```bash
# List all apps to find your app
az ad app list --display-name "GitHub-Actions-OIDC-AVM-Testing" --query "[].{Name:displayName, AppId:appId}" -o table

# Store APP_ID in variable
APP_ID=$(az ad app list --display-name "GitHub-Actions-OIDC-AVM-Testing" --query "[0].appId" -o tsv)
echo $APP_ID
```

### Step 4: Get Tenant and Subscription IDs

```bash
# Get Tenant ID
TENANT_ID=$(az account show --query tenantId -o tsv)
echo "Tenant ID: $TENANT_ID"

# Get Subscription ID
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
echo "Subscription ID: $SUBSCRIPTION_ID"
```

**Save these values** - you'll need them for GitHub secrets

---

## Configure Federated Identity Credentials

Federated credentials establish trust between GitHub Actions and Azure AD, allowing GitHub to request access tokens without storing secrets.

### Option A: Using Azure Portal (Recommended for Beginners)

1. **Navigate to Azure Portal:**
   - Go to [Azure Portal](https://portal.azure.com)
   - Search for **Azure Active Directory** or **Microsoft Entra ID**
   - Click **App registrations** in the left menu

2. **Select Your Application:**
   - Find `GitHub-Actions-OIDC-AVM-Testing`
   - Click on the application name

3. **Add Federated Credential for Main Branch:**
   - In the left menu, click **Certificates & secrets**
   - Click the **Federated credentials** tab
   - Click **+ Add credential**
   - Select scenario: **GitHub Actions deploying Azure resources**
   
   **Fill in the form:**
   ```
   Organization: madhurshukla23
   Repository: avm_testing
   Entity type: Branch
   Based on selection: main
   Name: github-actions-main-branch
   Description: GitHub Actions workflow authentication for main branch
   ```
   
   - Click **Add**

4. **Add Federated Credential for Pull Requests:**
   - Click **+ Add credential** again
   - Select scenario: **GitHub Actions deploying Azure resources**
   
   **Fill in the form:**
   ```
   Organization: madhurshukla23
   Repository: avm_testing
   Entity type: Pull request
   Name: github-actions-pull-requests
   Description: GitHub Actions workflow authentication for pull requests
   ```
   
   - Click **Add**

### Option B: Using Azure CLI

#### For Main Branch:

```bash
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-actions-main-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:madhurshukla23/avm_testing:ref:refs/heads/main",
    "description": "GitHub Actions workflow authentication for main branch",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

#### For Pull Requests:

```bash
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-actions-pull-requests",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:madhurshukla23/avm_testing:pull_request",
    "description": "GitHub Actions workflow authentication for pull requests",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

#### For Develop Branch (Optional):

```bash
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-actions-develop-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:madhurshukla23/avm_testing:ref:refs/heads/develop",
    "description": "GitHub Actions workflow authentication for develop branch",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

### Verify Federated Credentials:

```bash
az ad app federated-credential list --id $APP_ID --query "[].{Name:name, Subject:subject}" -o table
```

**Expected Output:**
```
Name                              Subject
--------------------------------  --------------------------------------------------
github-actions-main-branch        repo:madhurshukla23/avm_testing:ref:refs/heads/main
github-actions-pull-requests      repo:madhurshukla23/avm_testing:pull_request
github-actions-develop-branch     repo:madhurshukla23/avm_testing:ref:refs/heads/develop
```

---

## Assign Azure Permissions

The service principal needs permissions to create, manage, and destroy Azure resources.

### Step 1: Assign Contributor Role at Subscription Level

This grants permission to manage all resources in the subscription:

```bash
az role assignment create \
  --assignee $APP_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"
```

### Step 2: Assign User Access Administrator Role (If Managing RBAC)

Required if your Terraform modules assign role-based access control:

```bash
az role assignment create \
  --assignee $APP_ID \
  --role "User Access Administrator" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"
```

### Step 3: Verify Role Assignments

```bash
az role assignment list \
  --assignee $APP_ID \
  --scope "/subscriptions/$SUBSCRIPTION_ID" \
  --query "[].{Role:roleDefinitionName, Scope:scope}" -o table
```

**Expected Output:**
```
Role                          Scope
----------------------------  --------------------------------------------
Contributor                   /subscriptions/<SUBSCRIPTION_ID>
User Access Administrator     /subscriptions/<SUBSCRIPTION_ID>
```

### Alternative: Assign Permissions to Specific Resource Group

For more restrictive access, assign permissions only to specific resource groups:

```bash
# Create resource group for testing
az group create --name rg-terraform-testing --location eastus

# Assign Contributor role to resource group
az role assignment create \
  --assignee $APP_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-terraform-testing"
```

---

## Setup Terraform State Storage

Terraform needs a remote backend to store state files persistently across workflow runs.

### Step 1: Create Resource Group for State Storage

```bash
az group create \
  --name rg-terraform-state \
  --location eastus
```

### Step 2: Create Storage Account

**Important:** Storage account names must be globally unique, 3-24 characters, lowercase letters and numbers only.

```bash
# Generate unique storage account name
STORAGE_ACCOUNT_NAME="tfstate$(date +%s | tail -c 6)avm"

# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group rg-terraform-state \
  --location eastus \
  --sku Standard_LRS \
  --encryption-services blob \
  --https-only true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

echo "Storage Account Name: $STORAGE_ACCOUNT_NAME"
```

**Save the storage account name** - you'll need it as `TF_STATE_STORAGE_ACCOUNT`

### Step 3: Create Blob Container

```bash
az storage container create \
  --name tfstate \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login
```

### Step 4: Assign Storage Permissions to Service Principal

Grant the service principal permission to read/write state files:

```bash
az role assignment create \
  --assignee $APP_ID \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-terraform-state/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT_NAME"
```

### Step 5: Verify Storage Setup

```bash
# Check storage account
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group rg-terraform-state \
  --query "{Name:name, Location:location, SKU:sku.name}" -o table

# Check container
az storage container list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login \
  --query "[].name" -o table

# Verify permissions
az role assignment list \
  --assignee $APP_ID \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-terraform-state/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT_NAME" \
  --query "[].roleDefinitionName" -o table
```

---

## Configure GitHub Secrets

GitHub secrets securely store sensitive values that workflows can access.

### Step 1: Navigate to GitHub Repository

1. Go to `https://github.com/madhurshukla23/avm_testing`
2. Click **Settings** tab
3. In the left sidebar, expand **Secrets and variables**
4. Click **Actions**

### Step 2: Add Azure Authentication Secrets

Click **New repository secret** for each:

| Secret Name | Value | Description |
|------------|-------|-------------|
| `AZURE_CLIENT_ID` | `<APP_ID>` | Application (Client) ID from Step 2 |
| `AZURE_TENANT_ID` | `<TENANT_ID>` | Azure AD Tenant ID |
| `AZURE_SUBSCRIPTION_ID` | `<SUBSCRIPTION_ID>` | Azure Subscription ID |

### Step 3: Add Terraform State Storage Secrets

| Secret Name | Value | Description |
|------------|-------|-------------|
| `TF_STATE_RESOURCE_GROUP` | `rg-terraform-state` | Resource group containing storage account |
| `TF_STATE_STORAGE_ACCOUNT` | `<STORAGE_ACCOUNT_NAME>` | Storage account name from previous section |
| `TF_STATE_CONTAINER` | `tfstate` | Container name for state files |

### Step 4: Get Values Using Azure CLI (If Needed)

```bash
# Display all values needed for GitHub secrets
echo "=== GitHub Secrets Configuration ==="
echo "AZURE_CLIENT_ID: $APP_ID"
echo "AZURE_TENANT_ID: $TENANT_ID"
echo "AZURE_SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
echo "TF_STATE_RESOURCE_GROUP: rg-terraform-state"
echo "TF_STATE_STORAGE_ACCOUNT: $STORAGE_ACCOUNT_NAME"
echo "TF_STATE_CONTAINER: tfstate"
```

### Step 5: Verify Secrets in GitHub

After adding all secrets, you should see 6 repository secrets listed in the GitHub Actions secrets page.

---

## Verify Configuration

### Step 1: Test Authentication Locally (Optional)

Before running workflows, verify the setup:

```bash
# Login as the service principal
az login --service-principal \
  --username $APP_ID \
  --tenant $TENANT_ID \
  --federated-token $(curl -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=api://AzureADTokenExchange" | jq -r .value)

# Note: The above command only works within GitHub Actions environment
# For local testing, verify permissions instead:

az role assignment list --assignee $APP_ID --output table
```

### Step 2: Trigger GitHub Actions Workflow

1. **Push to Main Branch** or **Create Pull Request**
2. **Monitor Workflow:**
   - Go to **Actions** tab in GitHub
   - Click on the running workflow
   - Verify all jobs complete successfully

### Step 3: Check Workflow Steps

Verify these key steps succeed:

- ✅ **Azure Login** - Should authenticate without errors
- ✅ **Terraform Init** - Should initialize with remote backend
- ✅ **Terraform Plan** - Should generate execution plan
- ✅ **Terraform Apply** - Should deploy resources
- ✅ **Terraform Destroy** - Should clean up resources

### Step 4: Verify State File in Azure Storage

```bash
az storage blob list \
  --container-name tfstate \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login \
  --query "[].name" -o table
```

**Expected:** You should see state files like:
```
terraform-azurerm-avm-res-storage-storageaccount-main/default.tfstate
terraform-azurerm-avm-res-keyvault-vault-main/default.tfstate
```

---

## Troubleshooting

### Issue 1: "Failed to get OIDC token"

**Error Message:**
```
Error: Failed to get OIDC token from GitHub Actions
```

**Solution:**
- Verify workflow has `id-token: write` permission
- Check federated credential subject matches exactly: `repo:madhurshukla23/avm_testing:ref:refs/heads/main`
- Ensure credential is created in the correct Azure AD app

### Issue 2: "Insufficient Privileges"

**Error Message:**
```
Error: The client does not have authorization to perform action
```

**Solution:**
- Verify role assignments: `az role assignment list --assignee $APP_ID`
- Ensure Contributor role is assigned at subscription or resource group level
- Wait 5-10 minutes for role assignments to propagate

### Issue 3: "Failed to Initialize Backend"

**Error Message:**
```
Error: Failed to get existing workspaces: storage: service returned error: StatusCode=403
```

**Solution:**
- Verify service principal has "Storage Blob Data Contributor" role on storage account
- Check storage account name in GitHub secret matches actual storage account
- Ensure `use_oidc = true` is set in backend configuration

### Issue 4: "Subscription ID Not Found"

**Error Message:**
```
Error: building account: unable to configure ResourceManagerAccount: subscription ID could not be determined
```

**Solution:**
- Verify `ARM_SUBSCRIPTION_ID` environment variable is set in workflow
- Check GitHub secret `AZURE_SUBSCRIPTION_ID` is correct
- Ensure service principal has access to the subscription

### Issue 5: "Invalid Subject Identifier"

**Error Message:**
```
Error: AADSTS70021: No matching federated identity record found
```

**Solution:**
- Check federated credential subject format matches:
  - Branch: `repo:OWNER/REPO:ref:refs/heads/BRANCH_NAME`
  - PR: `repo:OWNER/REPO:pull_request`
- Verify organization and repository names are correct
- Ensure credential is added for the correct branch/PR

### Issue 6: Storage Account Name Already Exists

**Error Message:**
```
Error: The storage account named 'tfstate' is already taken
```

**Solution:**
- Storage account names must be globally unique
- Use the date-based naming: `tfstate$(date +%s | tail -c 6)avm`
- Or choose a different unique name

---

## Security Best Practices

### 1. Principle of Least Privilege

- Grant only required permissions (avoid subscription-wide Owner role)
- Use resource group scoped permissions when possible
- Regularly audit role assignments

### 2. Enable Storage Security Features

```bash
# Enable storage account firewall (optional)
az storage account update \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group rg-terraform-state \
  --default-action Deny

# Allow GitHub Actions IP ranges or use service endpoints
```

### 3. Enable Diagnostic Logging

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group rg-terraform-state \
  --workspace-name log-terraform-state

# Enable diagnostics on storage account
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-terraform-state \
  --workspace-name log-terraform-state \
  --query id -o tsv)

az monitor diagnostic-settings create \
  --name storage-diagnostics \
  --resource "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-terraform-state/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT_NAME" \
  --workspace $WORKSPACE_ID \
  --logs '[{"category":"StorageRead","enabled":true},{"category":"StorageWrite","enabled":true}]'
```

### 4. Rotate Credentials Regularly

- Federated credentials don't expire, but review and update them periodically
- Monitor Azure AD sign-in logs for suspicious activity
- Use Conditional Access policies if available

### 5. Environment Isolation

Consider separate service principals for different environments:

```bash
# Production
az ad app create --display-name "GitHub-Actions-OIDC-Production"

# Development
az ad app create --display-name "GitHub-Actions-OIDC-Development"
```

---

## Additional Resources

- [Azure OIDC Documentation](https://docs.microsoft.com/en-us/azure/active-directory/develop/workload-identity-federation)
- [GitHub Actions OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Terraform Azure Backend](https://developer.hashicorp.com/terraform/language/settings/backends/azurerm)
- [Azure CLI Reference](https://docs.microsoft.com/en-us/cli/azure/)

---

## Quick Reference Commands

```bash
# Display all configuration values
echo "APP_ID: $APP_ID"
echo "TENANT_ID: $TENANT_ID"
echo "SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
echo "STORAGE_ACCOUNT_NAME: $STORAGE_ACCOUNT_NAME"

# Verify authentication setup
az ad app federated-credential list --id $APP_ID -o table
az role assignment list --assignee $APP_ID -o table

# Check storage setup
az storage account show --name $STORAGE_ACCOUNT_NAME --resource-group rg-terraform-state
az storage container list --account-name $STORAGE_ACCOUNT_NAME --auth-mode login

# View state files
az storage blob list \
  --container-name tfstate \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login \
  --output table
```

---

**Document Version:** 1.0  
**Last Updated:** November 10, 2025  
**Maintained By:** DevOps Team
