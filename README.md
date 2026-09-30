# NTFS File Server Lab - Terraform & Azure AD Infrastructure

**Lab 1 — NTFS File Server** | Active Directory · NTFS · SMB · Terraform + PowerShell through the Azure VM agent

## Quick Overview

This repository contains Infrastructure as Code (Terraform) and configuration scripts to build a complete Windows file server lab environment with Active Directory, security groups, and NTFS permissions.

**What Gets Created:**
- DC01 (Domain Controller) with lab.local forest
- FS01 (File Server) with CompanyData share
- CLIENT01 (Windows 11 workstation)
- Virtual Network, NSG, and Key Vault in Azure

**Deployment Time:** 1.5–2 hours (mostly waiting on VM provisioning)

**Key Rule:** Never apply a plan that wants to destroy or replace a VM — it would wipe the entire lab.local domain.

---

## What You'll Learn

| Skill | Why It Matters |
|---|---|
| Infrastructure as Code with Terraform | Rebuild the entire lab identically, every time |
| Active Directory automation with PowerShell | Repeatable, reviewable domain configuration |
| NTFS permissions via security groups | Scale access management without editing folders |
| SMB share vs NTFS layers | Understand "most restrictive wins" |
| Secret management with Key Vault | Keep passwords out of files and terminal history |
| Azure VM agent for remote configuration | Configure machines while NSG stays locked down |
| Verification over status messages | "Provisioning succeeded" only means the script launched |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Azure VNet 10.0.0.0/16                │
│            Custom DNS: 10.0.1.4 (DC01)                  │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐           │
│  │   DC01   │  │   FS01   │  │   CLIENT01   │           │
│  │ 10.0.1.4 │  │Dynamic IP│  │  Dynamic IP  │           │
│  │   (AD)   │  │(File Srv)│  │  (Windows 11)│           │
│  └──────────┘  └──────────┘  └──────────────┘           │
│                                                         │
│        NSG: RDP 3389 from your IP only                  │
└─────────────────────────────────────────────────────────┘

Key Vault (separate): holds VM admin password
Remote state: Azure Storage account in rg-tfstate
```

**For detailed architecture diagrams, component explanations, and data flow:** See `docs/ARCHITECTURE.md`

---

## Prerequisites

**Tools Required:**
- Terraform >= 1.5.0
- Azure CLI >= 2.90.0
- PowerShell 5+ (Windows native or PowerShell Core)
- An Azure subscription

**One-time setup on your machine:**
```powershell
# Install tools (Windows)
winget install --exact --id Microsoft.AzureCLI
winget install --exact --id Hashicorp.Terraform

# Close and reopen terminal, then verify
az version
terraform -version

# Allow PowerShell scripts
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned

# Disable Windows sign-in broker (if using VS Code)
az config set core.enable_broker_on_windows=false
```

---

## Getting Started

### 1. Clone This Repository
```bash
git clone https://github.com/yourusername/ntfs-lab-terraform.git
cd ntfs-lab-terraform
```

### 2. Set Up Remote State (One Time)
Terraform needs a storage account to hold state. Create it once:
```powershell
az group create --name rg-tfstate --location centralus
az storage account create --name tfstatelabs<YOUR-RANDOM> --resource-group rg-tfstate --sku Standard_LRS
az storage container create --name tfstate --account-name tfstatelabs<YOUR-RANDOM>
```

### 3. Configure Variables
```powershell
# Copy the example file
Copy-Item terraform/terraform.tfvars.example terraform/terraform.tfvars

# Edit terraform/terraform.tfvars:
#   - your public IP (get it: Invoke-RestMethod https://api.ipify.org)
#   - a globally unique Key Vault name
```

### 4. Set Admin Password (First Time Only)
```powershell
$env:TF_VAR_admin_password = "YourSecure!Password123"
```

### 5. Deploy Infrastructure
```powershell
cd terraform
terraform init
terraform plan      # Review the 18 resources
terraform apply
```

### 6. Build the Domain and File Server
```powershell
cd ../scripts
$labPwd = az keyvault secret show --vault-name <your-vault-name> --name vm-admin-password --query value -o tsv

# Run each script in order, verifying after each one
az vm run-command invoke -g RG-FileServerLab -n DC01 --command-id RunPowerShellScript --scripts "@00-promote-dc.ps1" --parameters "SafeModePassword=$labPwd"
# ... then run scripts 01-06 (see docs/DEPLOYMENT.md for full details)
```

---

## Repository Structure

```
ntfs-lab-terraform/
├── README.md                          ← You are here
├── .gitignore
├── .github/
│   └── ISSUE_TEMPLATE/
│       └── error-report.md
│
├── terraform/                         ← Infrastructure as Code
│   ├── versions.tf                    ← Provider versions
│   ├── backend.tf                     ← Remote state config
│   ├── variables.tf                   ← Input variables
│   ├── main.tf                        ← Network + 3 VMs
│   ├── keyvault.tf                    ← Key Vault + secret
│   ├── outputs.tf                     ← Output IPs
│   ├── terraform.tfvars.example       ← Safe template (commit this)
│   ├── terraform.tfvars               ← Real values (gitignored)
│   └── .terraform.lock.hcl            ← Provider version lock (commit this)
│
├── scripts/                           ← PowerShell configuration
│   ├── 00-promote-dc.ps1              ← Create lab.local domain
│   ├── 01-join-domain.ps1             ← Join FS01 & CLIENT01
│   ├── 02-create-ous.ps1              ← Build OU structure
│   ├── 03-create-groups.ps1           ← Create security groups
│   ├── 04-create-users.ps1            ← Add test users
│   ├── 05-create-share-and-permissions.ps1  ← NTFS permissions
│   └── 06-add-rdp-users.ps1           ← RDP access for users
│
└── docs/                              ← Documentation
    ├── DEPLOYMENT.md                  ← Step-by-step walkthrough
    ├── ERRORS.md                      ← Common errors & fixes
    ├── ARCHITECTURE.md                ← Design decisions
    └── LESSONS.md                     ← Key takeaways
```

---

## Test Users and Permissions

Four test users across two departments. Brian deliberately belongs to two Finance groups to demonstrate cumulative Allow permissions.

| User | Department | Groups | Finance Access | HR Access |
|---|---|---|---|---|
| alice.finance | Finance | GG-Finance-ReadOnly | Read only | Denied |
| brian.finance | Finance | GG-Finance-ReadOnly + GG-Finance-Modify | Read + Modify | Denied |
| carla.hr | HR | GG-HR-ReadOnly | Denied | Read only |
| david.hr | HR | GG-HR-FullControl | Denied | Full control |

---

## Operating Rules

1. **Always load the password from Key Vault before `terraform plan` or `terraform apply`**
   ```powershell
   $env:TF_VAR_admin_password = az keyvault secret show --vault-name <your-vault-name> --name vm-admin-password --query value -o tsv
   ```

2. **Read the plan, not the summary.** If you see `# forces replacement`, stop and debug before applying.

3. **Verify every step.** "Provisioning succeeded" only means the script launched. Always run the verification command.

4. **Commit `.terraform.lock.hcl`** — it pins exact provider versions so a clone reproduces the same build.

5. **Never commit `terraform.tfvars`** — it's gitignored. The `.example` file is safe to commit.

---

## Common Tasks

### Pause the Lab (Stop Billing)
```powershell
az vm deallocate --resource-group RG-FileServerLab --name DC01
az vm deallocate --resource-group RG-FileServerLab --name FS01
az vm deallocate --resource-group RG-FileServerLab --name CLIENT01
```

### Resume
```powershell
az vm start --resource-group RG-FileServerLab --name DC01
az vm start --resource-group RG-FileServerLab --name FS01
az vm start --resource-group RG-FileServerLab --name CLIENT01
```

### Tear Down Completely
```powershell
$env:TF_VAR_admin_password = az keyvault secret show --vault-name <your-vault-name> --name vm-admin-password --query value -o tsv
cd terraform
terraform destroy
```

### Move to a New Machine
State lives in Azure. On a new machine:
```powershell
git clone ...
cd terraform
terraform init
$env:TF_VAR_admin_password = az keyvault secret show --vault-name <your-vault-name> --name vm-admin-password --query value -o tsv
terraform plan      # should say: No changes
```

---

## Troubleshooting

**Quick diagnostics:**
- Did the Terraform deployment succeed? → `az group list --query "[].name" -o tsv`
- Is DC01 actually a domain controller? → See `docs/ERRORS.md`
- Can FS01 find the domain? → See `docs/ERRORS.md` — DNS diagnostics section
- Double extension on script files? → Run `dir` in scripts folder and look for `.ps1.ps1`

**Detailed troubleshooting:** See `docs/ERRORS.md` for the 20+ errors encountered during this build, their root causes, and fixes.

---

## Key Concepts

### Secret Management
The admin password lives **only** in Key Vault after initial deployment. It never touches your disk:
- Environment variable (temporary, per-session)
- Key Vault (permanent, encrypted)

### NTFS vs Share Permissions
- **Share permissions:** Everyone Full Control (wide open)
- **NTFS permissions:** Explicit grants per security group (where access is actually decided)
- **Result:** Most restrictive layer wins → NTFS is the single point of access control

### Verification Over Status
Azure reports "Provisioning succeeded" when a script *launches*, not when it *works*. Always verify:
```powershell
# After script runs, check the actual result:
Get-WindowsFeature AD-Domain-Services  # for DC01 promotion
(Get-CimInstance Win32_ComputerSystem).Domain  # for domain joins
Get-SmbShare -Name CompanyData  # for share creation
```

---

## Resources

- [Terraform Azure Provider Docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Azure AD Module Docs](https://learn.microsoft.com/en-us/powershell/module/activedirectory/)
- [az vm run-command Reference](https://learn.microsoft.com/en-us/cli/azure/vm/run-command)
- [NTFS Permissions Guide](https://learn.microsoft.com/en-us/windows/win32/fileio/file-security-and-access-rights)

---

## Contributing

This is a learning lab. If you find errors, have improvements, or encounter issues:

1. Check `docs/ERRORS.md` first
2. Open an issue with the error details
3. Include output from diagnostic commands (redact sensitive info)

---

## License

MIT — Use freely in your lab environment. See LICENSE file.

---

## Author Notes

This SOP was built by actually running through a complete deployment, documenting every command and every real error. Nothing here is theoretical. Every troubleshooting entry happened during this build.

Key habits this lab teaches:
- "It said it worked" ≠ "it works"
- Read plans, not summaries
- Secrets belong in vaults, not in files
- Grant to groups, not users
- Verify results before moving to the next step

**September 2026 — Verified Build Edition**
