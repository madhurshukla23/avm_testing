# Windows Self-Hosted Runner Setup Guide

## Changes Made

All workflows in `.github/workflows/` have been updated to use Windows self-hosted runners:

### Updated Workflows:
1. ✅ `terraform-ci-scalable.yml` - Main pipeline
2. ✅ `validate-module.yml` - Module validation
3. ✅ `test-example.yml` - Example testing
4. ✅ `lint-module.yml` - Code linting
5. ✅ `generate-docs.yml` - Documentation generation
6. ✅ `unit-tests.yml` - Unit testing

### Key Changes:
- **Runner Configuration**: Changed from `ubuntu-latest` to `[self-hosted, windows]`
- **Shell Scripts**: Converted all bash scripts to PowerShell
- **Environment Variables**: Updated syntax from `$VARIABLE` to `$env:VARIABLE`
- **File Operations**: Updated file creation and manipulation commands for Windows
- **Tool Downloads**: Changed to Windows-compatible binaries (e.g., `yq_windows_amd64.exe`)

## Prerequisites for Windows Self-Hosted Runners

### 1. Required Software
Your Windows self-hosted runners must have the following installed:

#### Essential Tools:
- **PowerShell 5.1+** (pre-installed on Windows)
- **Git for Windows** (https://git-scm.com/download/win)
- **Terraform** (version 1.9.0 or compatible)
  ```powershell
  choco install terraform
  # OR download from https://www.terraform.io/downloads
  ```

#### Additional Tools (based on workflow needs):
- **TFLint** (for lint-module.yml)
  ```powershell
  choco install tflint
  # OR download from https://github.com/terraform-linters/tflint/releases
  ```
- **Azure CLI** (for Azure authentication)
  ```powershell
  winget install -e --id Microsoft.AzureCLI
  # OR download MSI from https://aka.ms/installazurecliwindows
  ```

### 2. GitHub Actions Runner Setup

1. **Add Self-Hosted Runner to Repository:**
   - Go to your repository → Settings → Actions → Runners
   - Click "New self-hosted runner"
   - Select "Windows" as the OS
   - Follow the setup instructions provided

2. **Configure Runner Labels:**
   - Ensure the runner has the label `windows`
   - You can add custom labels if needed

3. **Install as a Service (Recommended):**
   ```powershell
   cd C:\actions-runner
   .\svc.ps1 install
   .\svc.ps1 start
   ```

### 3. Environment Configuration

#### Required Environment Variables:
The workflows expect these to be available on the runner:
- `TEMP` or `TMP` - For temporary file storage
- `PATH` - Should include Git, Terraform, and other tools

#### Azure Authentication:
Ensure the runner can authenticate to Azure using OIDC:
- Configure Azure service principal with federated credentials
- Set up repository secrets:
  - `AZURE_CLIENT_ID`
  - `AZURE_TENANT_ID`
  - `AZURE_SUBSCRIPTION_ID`
  - `TF_STATE_RESOURCE_GROUP` (for backend config)
  - `TF_STATE_STORAGE_ACCOUNT` (for backend config)
  - `TF_STATE_CONTAINER` (for backend config)

### 4. File System Permissions

Ensure the runner service account has:
- Read/Write access to the workspace directory
- Execute permissions for all required tools
- Network access to:
  - GitHub.com (for actions and checkout)
  - Azure APIs
  - Terraform Registry
  - Tool download URLs (github.com/mikefarah/yq, etc.)

### 5. Firewall and Network

Allow outbound connections to:
- `github.com` (443)
- `*.actions.githubusercontent.com` (443)
- `*.azure.com` (443)
- `registry.terraform.io` (443)
- `releases.hashicorp.com` (443)

## Testing the Setup

### Test Runner Connectivity:
```powershell
# Check if runner is online
# Go to: Repository → Settings → Actions → Runners

# Verify required tools
terraform version
git --version
tflint --version  # If using lint workflow
az --version      # If using Azure
```

### Test Workflow Execution:
1. Trigger the workflow manually using `workflow_dispatch`
2. Check the Actions tab for execution
3. Verify all jobs complete successfully

## Troubleshooting

### Common Issues:

1. **"Runner not found" Error**
   - Ensure runner is online and has the `windows` label
   - Check runner service is running: `Get-Service actions.runner.*`

2. **Tool Not Found Errors**
   - Verify tool is in PATH: `$env:PATH -split ';'`
   - Reinstall the tool or add to PATH manually

3. **Permission Denied Errors**
   - Run PowerShell as Administrator when installing tools
   - Check runner service account permissions

4. **Azure Authentication Failures**
   - Verify OIDC federation is configured correctly
   - Check repository secrets are set
   - Ensure `id-token: write` permission is present in workflows

5. **Path Issues**
   - Forward slashes (`/`) in paths work on Windows
   - GitHub Actions handles path conversions automatically

## Verification Checklist

- [ ] Windows runner is installed and running
- [ ] Runner has `windows` label
- [ ] Terraform is installed and in PATH
- [ ] Git is installed and in PATH
- [ ] TFLint is installed (if using lint workflows)
- [ ] Azure CLI is installed (if using Azure authentication)
- [ ] Repository secrets are configured
- [ ] Azure OIDC federation is set up
- [ ] Runner has network access to required endpoints
- [ ] Test workflow execution completes successfully

## Notes

- The workflows in `modules/terraform-azurerm-avm-res-*/` directories are separate and maintain their own runner configurations
- PowerShell scripts use UTF-8 encoding for file operations
- Temporary files are stored in `$env:TEMP` directory
- The `yq` tool is downloaded on-demand and cached in `$env:TEMP`
