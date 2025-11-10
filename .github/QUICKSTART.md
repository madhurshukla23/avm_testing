# Quick Start: Scalable Pipeline

## 🎯 The Problem

Adding new Terraform modules requires editing workflow files, creating duplicate job definitions, and maintaining complex YAML.

## ✅ The Solution

**Configuration-driven pipeline** that auto-scales based on a simple config file.

---

## 📝 How to Add a New Module (3 Steps)

### 1. Edit Configuration File

Open `.github/modules-config.yml` and add:

```yaml
- name: terraform-azurerm-avm-res-yourmodule-main
  enabled: true
  has_unit_tests: false
  examples:
    - default
    - advanced
```

### 2. Commit & Push

```bash
git add .github/modules-config.yml
git commit -m "Add new module to CI pipeline"
git push
```

### 3. Done! ✅

The pipeline automatically:
- Validates the module
- Tests both examples (default, advanced)
- Runs linting
- Generates documentation

**No workflow edits required!**

---

## 🔄 Comparison

### Old Way (Manual)
```yaml
# Edit workflow file - add ~10 job definitions
validate-newmodule:
  uses: ./.github/workflows/validate-module.yml
  with:
    module_name: terraform-azurerm-avm-res-yourmodule-main

test-newmodule-default:
  uses: ./.github/workflows/test-example.yml
  with:
    module_name: terraform-azurerm-avm-res-yourmodule-main
    example_name: default

# ... 8 more jobs ...
```

### New Way (Config-Driven) ✨
```yaml
# Edit config file - add 6 lines
- name: terraform-azurerm-avm-res-yourmodule-main
  enabled: true
  has_unit_tests: false
  examples:
    - default
    - advanced
```

---

## 📊 Quick Stats

| Action | Old Approach | New Approach |
|--------|--------------|--------------|
| Add module | Edit workflow (30+ lines) | Edit config (6 lines) |
| Add example | Edit workflow (5 lines) | Edit config (1 line) |
| Disable module | Comment out jobs | Set `enabled: false` |
| Maintenance | Update all job definitions | Update reusable workflows |
| Scale to 10 modules | ~300 lines of YAML | ~60 lines of config |

---

## 🚀 File Structure

```
.github/
├── modules-config.yml                 # ← EDIT THIS to add modules
└── workflows/
    ├── terraform-ci-scalable.yml      # ← Main pipeline
    └── template/                      # Workflow templates
        ├── validate-module.yml
        ├── test-example.yml
        ├── lint-module.yml
        ├── generate-docs.yml
        └── unit-tests.yml
```

---

## 💡 Common Scenarios

### Scenario 1: Add Storage Module

```yaml
- name: terraform-azurerm-avm-res-storage-storageaccount-main
  enabled: true
  has_unit_tests: false
  examples:
    - default
    - cmk
```

### Scenario 2: Add Networking Module with Tests

```yaml
- name: terraform-azurerm-avm-res-network-vnet-main
  enabled: true
  has_unit_tests: true  # Has .tftest.hcl files
  examples:
    - default
    - with-subnets
    - with-peering
```

### Scenario 3: Temporarily Disable Module

```yaml
- name: terraform-azurerm-avm-res-compute-vm-main
  enabled: false  # ← Tests will be skipped
  has_unit_tests: true
  examples:
    - default
```

---

## 📖 Full Documentation

- **Detailed Guide:** `.github/SCALABLE-PIPELINE.md`
- **Workflow Reference:** `.github/workflows/README.md`
- **Config File:** `.github/modules-config.yml`

---

**Ready to scale? Just edit the config file!** 🎉
