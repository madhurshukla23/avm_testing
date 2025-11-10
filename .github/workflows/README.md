# GitHub Workflows Documentation

## 📁 Workflow Structure

This repository uses a **modular workflow architecture** with reusable workflows for better organization, maintainability, and reusability.

### Workflow Files

```
.github/workflows/
├── terraform-ci-scalable.yml  # 🎯 Main scalable pipeline (USE THIS)
├── terraform-test.yml         # Legacy workflow (can be archived)
├── main-pipeline.yml          # Alternative main pipeline
└── template/                  # 📁 Workflow templates
    ├── validate-module.yml    # ✅ Module validation
    ├── test-example.yml       # 🧪 Example testing
    ├── lint-module.yml        # 🔍 Code quality
    ├── generate-docs.yml      # 📚 Documentation
    └── unit-tests.yml         # 🧬 Unit tests
```

---

## 🎯 Main Pipeline (`main-pipeline.yml`)

**Purpose:** Orchestrates all testing stages for Terraform modules

**Triggers:**
- Push to `main` or `develop` branches
- Pull requests to `main` or `develop`
- Manual workflow dispatch

**Pipeline Stages:**

### 1️⃣ **Module Validation**
- Validates both Storage Account and Key Vault modules
- Checks formatting, initialization, and syntax
- Runs in parallel

### 2️⃣ **Example Testing**
- **Storage Account:** 11 examples
  - default, customer-managed-key, data_lake_gen2, diagnostic-settings, local_user, management-policy, private-endpoint-basic, private-endpoint-manage_dns_zone_group, private-endpoint-static-ip, private-endpoint-unmanage_dns_zone_group, role_assignments
- **Key Vault:** 6 examples
  - default, access-policies, create-key, create-secret, diagnostic-settings, private-endpoint
- All examples run in parallel after validation

### 3️⃣ **Code Quality (Linting)**
- Runs TFLint on all modules
- Parallel execution

### 4️⃣ **Documentation**
- Auto-generates README with terraform-docs
- Runs for all modules

### 5️⃣ **Unit Tests**
- Runs Terraform native tests for Key Vault module
- Uses mock providers (no real Azure resources)

### 6️⃣ **Summary**
- Generates comprehensive pipeline summary
- Shows pass/fail status for each stage

---

## 📋 Reusable Workflows

### `validate-module.yml`

**Purpose:** Validate a single Terraform module

**Inputs:**
- `module_name` (required): Module directory name
- `terraform_version` (optional): Default `1.9.0`

**Secrets:**
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

**Usage:**
```yaml
jobs:
  validate:
    uses: ./.github/workflows/validate-module.yml
    with:
      module_name: terraform-azurerm-avm-res-storage-storageaccount-main
      terraform_version: "1.9.0"
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

### `test-example.yml`

**Purpose:** Test a single Terraform example

**Inputs:**
- `module_name` (required): Module directory name
- `example_name` (required): Example directory name
- `terraform_version` (optional): Default `1.9.0`

**Secrets:**
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

**Usage:**
```yaml
jobs:
  test-example:
    uses: ./.github/workflows/test-example.yml
    with:
      module_name: terraform-azurerm-avm-res-keyvault-vault-main
      example_name: default
      terraform_version: "1.9.0"
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

### `lint-module.yml`

**Purpose:** Run TFLint on a module

**Inputs:**
- `module_name` (required): Module directory name

**Usage:**
```yaml
jobs:
  lint:
    uses: ./.github/workflows/lint-module.yml
    with:
      module_name: terraform-azurerm-avm-res-storage-storageaccount-main
```

---

### `generate-docs.yml`

**Purpose:** Generate module documentation

**Inputs:**
- `module_name` (required): Module directory name

**Usage:**
```yaml
jobs:
  docs:
    uses: ./.github/workflows/generate-docs.yml
    with:
      module_name: terraform-azurerm-avm-res-keyvault-vault-main
```

---

### `unit-tests.yml`

**Purpose:** Run Terraform native unit tests

**Inputs:**
- `module_name` (required): Module directory name
- `terraform_version` (optional): Default `1.9.0`

**Usage:**
```yaml
jobs:
  unit-tests:
    uses: ./.github/workflows/unit-tests.yml
    with:
      module_name: terraform-azurerm-avm-res-keyvault-vault-main
      terraform_version: "1.9.0"
```

---

## 🚀 Adding a New Module

### Step 1: Add to `main-pipeline.yml`

**Module Validation:**
```yaml
validate-newmodule:
  name: Validate New Module
  uses: ./.github/workflows/validate-module.yml
  with:
    module_name: terraform-azurerm-avm-res-newmodule-main
    terraform_version: ${{ env.TERRAFORM_VERSION }}
  secrets:
    AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
    AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
    AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**Example Testing:**
```yaml
test-newmodule-default:
  name: New Module - Default
  needs: validate-newmodule
  uses: ./.github/workflows/test-example.yml
  with:
    module_name: terraform-azurerm-avm-res-newmodule-main
    example_name: default
    terraform_version: ${{ env.TERRAFORM_VERSION }}
  secrets:
    AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
    AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
    AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**Linting:**
```yaml
lint-newmodule:
  name: Lint New Module
  uses: ./.github/workflows/lint-module.yml
  with:
    module_name: terraform-azurerm-avm-res-newmodule-main
```

**Documentation:**
```yaml
docs-newmodule:
  name: Docs for New Module
  uses: ./.github/workflows/generate-docs.yml
  with:
    module_name: terraform-azurerm-avm-res-newmodule-main
```

**Unit Tests (if applicable):**
```yaml
unit-tests-newmodule:
  name: Unit Tests - New Module
  uses: ./.github/workflows/unit-tests.yml
  with:
    module_name: terraform-azurerm-avm-res-newmodule-main
    terraform_version: ${{ env.TERRAFORM_VERSION }}
```

### Step 2: Update Summary Job

Add the new jobs to the `needs` list and update the summary logic.

---

## ✨ Benefits of This Architecture

### 1. **Modularity**
- Each workflow has a single responsibility
- Easy to understand and maintain
- Reusable across different modules

### 2. **Reusability**
- Can call the same workflow multiple times with different inputs
- No code duplication
- Consistent testing across all modules

### 3. **Scalability**
- Adding new modules is straightforward
- Just call existing workflows with new parameters
- No need to modify core testing logic

### 4. **Parallel Execution**
- All validations run in parallel
- All examples run in parallel (after validation)
- Faster pipeline execution

### 5. **Easier Debugging**
- Each job has a clear, descriptive name
- Easier to identify which specific test failed
- Isolated logs for each workflow

### 6. **Maintainability**
- Update testing logic in one place
- Changes automatically apply to all modules
- Cleaner, more organized workflow structure

---

## 🔧 Configuration

### Environment Variables
Set in `main-pipeline.yml`:
- `TERRAFORM_VERSION`: Terraform version to use (default: `1.9.0`)

### Required Secrets
Configure in repository settings:
- `AZURE_CLIENT_ID`: Azure AD application client ID
- `AZURE_TENANT_ID`: Azure AD tenant ID
- `AZURE_SUBSCRIPTION_ID`: Azure subscription ID

### Authentication
Uses **Azure OIDC (Federated Identity)** for secure, credential-less authentication.

---

## 📊 Pipeline Visualization

```
main-pipeline.yml
│
├─ validate-storage ──┬─ test-storage-default
│                     ├─ test-storage-cmk
│                     ├─ test-storage-datalake
│                     ├─ ...
│                     └─ test-storage-rbac
│
├─ validate-keyvault ─┬─ test-keyvault-default
│                     ├─ test-keyvault-accesspolicies
│                     ├─ test-keyvault-createkey
│                     ├─ ...
│                     └─ test-keyvault-pe
│
├─ lint-storage
├─ lint-keyvault
│
├─ docs-storage
├─ docs-keyvault
│
├─ unit-tests-keyvault
│
└─ summary (waits for all)
```

---

## 🎯 Best Practices Implemented

✅ **Single Responsibility Principle** - Each workflow does one thing well  
✅ **DRY (Don't Repeat Yourself)** - Reusable workflows eliminate duplication  
✅ **Fail-Fast Disabled** - Continue testing even if one job fails  
✅ **Clear Naming** - Descriptive job names for easy identification  
✅ **Dependency Management** - Jobs run in logical order  
✅ **Comprehensive Summary** - Clear reporting of all results  
✅ **Scalable Architecture** - Easy to add new modules  

---

## 📝 Migration from Old Workflow

If you want to switch from `terraform-test.yml` to this new structure:

1. The new `main-pipeline.yml` is the replacement for `terraform-test.yml`
2. All functionality is preserved and improved
3. You can delete or archive `terraform-test.yml` once the new structure is tested
4. Workflows are backward compatible - no changes needed to secrets or triggers

---

## 🔍 Troubleshooting

### Workflow not triggering
- Check that changes are in the correct paths (`modules/**/*.tf`)
- Ensure you're pushing to `main` or `develop` branches

### Job failing
- Check the specific reusable workflow logs
- Verify all required secrets are configured
- Ensure Terraform version compatibility

### Adding new examples
- Just add a new job calling `test-example.yml`
- Set the correct `module_name` and `example_name`
- Add to the summary job's `needs` list

---

**Happy Testing! 🚀**
