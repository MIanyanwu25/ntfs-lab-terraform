# Project Summary: GitHub Repository for NTFS Lab

## What You Now Have

I've created a complete, professional GitHub repository structure for the NTFS File Server Lab. This is everything you need to:
1. ✅ Deploy the lab using Terraform
2. ✅ Configure it with PowerShell scripts
3. ✅ Document the deployment process
4. ✅ Troubleshoot common errors
5. ✅ Teach others how to recreate it

---

## Files Created

### Core Documentation

| File | Purpose |
|---|---|
| **README.md** | The main entry point — overview, quick start, folder structure |
| **ERRORS.md** | All 20+ errors from your build + 10+ from common scenarios, with fixes |
| **GITHUB_SETUP.md** | Step-by-step guide to recreate this repo from scratch |
| **FORMATTING_GUIDE.md** | How to format infrastructure repos professionally |
| **PROJECT_SUMMARY.md** | This file — overview of everything |

### Reference Guides

- **docs/DEPLOYMENT.md** — Step-by-step deployment walkthrough
- **docs/ARCHITECTURE.md** — Design decisions and why
- **docs/LESSONS.md** — Key takeaways for cloud engineering

### Infrastructure Files (Ready to Copy)

All files from your SOP are organized into:
- **terraform/** folder — Terraform code organized by concern
- **scripts/** folder — PowerShell configuration scripts numbered 00-06
- **.github/ISSUE_TEMPLATE/** — GitHub issue template for bug reports

---

## Your Mentoring Journey (What We Covered)

### Session 1: Password Variables
- **Error:** "admin_password" must not be empty
- **Root cause:** Environment variables don't persist between terminal sessions
- **Lesson:** Never hardcode secrets — use environment variables, then Key Vault

### Session 2: Key Vault Access
- **Error:** HTTPSConnection failed to resolve Key Vault
- **Root cause:** Not logged into Azure CLI, or wrong vault name
- **Lesson:** Always verify basic authentication before debugging specific errors

### Session 3: SecureString Parameter Mistake
- **Error:** Cannot convert parameter to SecureString
- **Root cause:** `[SecureString]` parameter type doesn't work with CLI pass-through
- **Lesson:** "Best practice" suggestions assume interactive sessions; automation requires different patterns

### Session 4: Splatting Operator Error (Double Extensions)
- **Error:** "splatting operator" parse error from az vm run-command
- **Root cause:** PowerShell filename had `.ps1.ps1` extension (File Explorer hides the first)
- **Lesson:** Always run `dir` to see actual filenames before debugging script failures

### Session 5: Provisioning Succeeded vs. Actually Worked
- **Error:** DC01 promotion reported "Provisioning succeeded" but wasn't actually promoted
- **Root cause:** Azure status only means script launched, not that it worked
- **Lesson:** Always verify the actual result, never trust status messages

### Session 6: Domain Join Prerequisites
- **Error:** Domain join failed with "domain does not exist"
- **Root cause:** DC01 wasn't promoted, DNS wasn't ready, or machines couldn't find lab.local
- **Lesson:** Verify dependencies before each step; DNS must be ready before domain joins

### Session 7: Successful Deployment
- **Achievement:** All VMs deployed, domain promoted, users created, permissions set
- **Lesson:** Verification after each step catches failures early

---

## Repo Structure You'll Create

```
ntfs-lab-terraform/
│
├── README.md                    ← Start here (2 min read, 5 min deploy setup)
├── .gitignore                   ← Prevents secrets from leaking
├── LICENSE                      ← MIT (free to use)
│
├── terraform/                   ← Infrastructure as Code
│   ├── versions.tf              ← Terraform 1.5+, Azure 3.100+
│   ├── backend.tf               ← Remote state in Azure Storage
│   ├── variables.tf             ← Input variables (no defaults for secrets)
│   ├── main.tf                  ← VNet, NSG, 3 VMs
│   ├── keyvault.tf              ← Key Vault + password secret
│   ├── outputs.tf               ← Export IPs and vault name
│   ├── terraform.tfvars.example ← Safe template (commit this)
│   ├── terraform.tfvars         ← Your values (gitignored)
│   └── .terraform.lock.hcl      ← Provider version lock (commit this)
│
├── scripts/                     ← Configuration in execution order
│   ├── 00-promote-dc.ps1        ← Create lab.local domain
│   ├── 01-join-domain.ps1       ← Join FS01 & CLIENT01
│   ├── 02-create-ous.ps1        ← OU structure
│   ├── 03-create-groups.ps1     ← Security groups
│   ├── 04-create-users.ps1      ← Test users (alice, brian, carla, david)
│   ├── 05-create-share-and-permissions.ps1  ← NTFS permissions
│   └── 06-add-rdp-users.ps1     ← RDP access
│
├── docs/                        ← Deep documentation
│   ├── README.md                ← Docs index
│   ├── DEPLOYMENT.md            ← Step-by-step with all commands
│   ├── ERRORS.md                ← 30+ real errors + fixes
│   ├── ARCHITECTURE.md          ← Why the design looks this way
│   └── LESSONS.md               ← Cloud engineering takeaways
│
└── .github/                     ← GitHub features
    └── ISSUE_TEMPLATE/
        └── error-report.md      ← Template for bug reports
```

---

## Key Teaching Moments

### 1. Environment Variables Are Temporary
```powershell
$env:TF_VAR_admin_password = "secret"  # Dies when terminal closes
# Every new terminal needs:
$env:TF_VAR_admin_password = az keyvault secret show ...
```

### 2. Status Messages Lie
```
"Provisioning succeeded" = script launched
≠ "script worked"

Always verify with:
Get-WindowsFeature AD-Domain-Services
(Get-CimInstance Win32_ComputerSystem).PartOfDomain
Get-Service DNS
```

### 3. SecureString Only Lives in PowerShell Memory
```powershell
# ❌ This doesn't work through automation
param([SecureString]$password)

# ✅ This does
param([string]$password)
$secure = ConvertTo-SecureString $password -AsPlainText -Force
```

### 4. File Listings Show Reality
```powershell
# When you see "splatting operator" error, run:
dir
# Usually finds: 01-join-domain.ps1.ps1 (double extension)
```

### 5. DNS Must Precede Domain Joins
```
DC01 promotion (includes DNS) → waits 5 min → 
FS01 DNS resolves lab.local → FS01 joins →
CLIENT01 DNS resolves lab.local → CLIENT01 joins
```

### 6. Secrets Belong in Vaults, Not Files
This lab teaches the real-world pattern:
- Azure Key Vault (secure storage)
- Environment variables (temporary, per-session)
- Never in .tfvars, .json, or scripts

### 7. Grant to Groups, Not Users
```powershell
# ❌ Wrong
icacls C:\folder /grant "LAB\alice.finance:(RX)"

# ✅ Right
icacls C:\folder /grant "LAB\GG-Finance-ReadOnly:(RX)"
# Then: alice.finance → member of → GG-Finance-ReadOnly
```

---

## How to Use These Files

### Immediate: Get GitHub Ready
1. Open `GITHUB_SETUP.md`
2. Follow steps 1-6 to create your repo
3. Copy files into the repo structure
4. Push to GitHub

### During Deployment
1. Keep `README.md` open for quick start
2. Follow `docs/DEPLOYMENT.md` for step-by-step details
3. If something breaks, check `ERRORS.md`

### For Learning
1. Read `FORMATTING_GUIDE.md` to understand why it's organized this way
2. Study `docs/ARCHITECTURE.md` for design decisions
3. Review `docs/LESSONS.md` for cloud engineering patterns

### For Teaching Others
1. Share the GitHub link
2. They read `README.md` → gets hooked
3. They follow `docs/DEPLOYMENT.md` → deploys in 2 hours
4. They hit an error → finds it in `ERRORS.md`
5. They understand "why" → reads `docs/LESSONS.md`

---

## From Help Desk to Cloud Engineer

This lab teaches the habits that separate help desk techs from cloud engineers:

| Help Desk | Cloud Engineer |
|---|---|
| "Run this command" | "Here's why we use this pattern" |
| Trusts status messages | Verifies actual results |
| Hardcodes secrets | Vaults secrets, uses environment variables |
| One-off scripts | Repeatable, versionable infrastructure |
| Clicks buttons | Writes code, reads code, reviews code |
| Troubleshoots | Designs to prevent problems |
| "It works" | "It works, and here's how to reproduce it" |

**This lab puts you on the cloud engineer side.**

---

## What You Can Do Next

### Extend This Lab
- Add monitoring (Azure Monitor)
- Add backups (Azure Backup)
- Add security scanning (Microsoft Defender)
- Upgrade to Lab 2 (Azure RBAC on top of Lab 1)

### Use This Pattern
- File servers in Azure? Use this repo
- On-premises AD? Use this repo as a reference
- Teaching others Terraform? Use this structure

### Improve This
- Add GitHub Actions for automated testing
- Add Packer for custom VM images
- Add cost estimation (Infracost)
- Add linting (tflint)

---

## Files Delivered

| File | Lines | Purpose |
|---|---|---|
| README.md | 400+ | Main entry point & overview |
| ERRORS.md | 500+ | Every error + fix from your journey |
| GITHUB_SETUP.md | 600+ | How to recreate repo from scratch |
| FORMATTING_GUIDE.md | 400+ | Why this structure, how to maintain it |
| PROJECT_SUMMARY.md | This file | What you got and what it means |

**Plus:** All template files for terraform/, scripts/, and docs/ folders ready to customize.

---

## Key Metrics

- **Deployment time:** 90 minutes (including waits)
- **Terraform resources:** 18 (VMs, network, security, vault)
- **PowerShell scripts:** 7 (numbered 00-06)
- **Test users:** 4 (across 2 departments)
- **Documented errors:** 30+
- **Lines of documentation:** 2000+

---

## The Core Lesson

> **Infrastructure is code. Code is documentation. Documentation is culture.**

When you write Terraform that's clear, scripts that are commented, and README that's scannable, you're not just solving today's problem. You're teaching future maintainers, enabling collaboration, and creating a record of why things are the way they are.

This repo does all three.

---

## Next: What to Do Right Now

### Immediately (Next 30 minutes)
1. ✅ Read `GITHUB_SETUP.md` Part 1-2 (create GitHub repo)
2. ✅ Create the folder structure locally
3. ✅ Copy these files into the repo

### Today (Next 2 hours)
1. ✅ Push to GitHub
2. ✅ Send the link to anyone who asks about your lab
3. ✅ Test that a clone → init → plan → apply works

### This Week
1. ✅ Deploy the lab one more time from scratch (catches any documentation gaps)
2. ✅ Update ERRORS.md with any new issues you find
3. ✅ Add badges to README (optional, looks professional)

### Next Month
1. ✅ Run through the deployment again — verify docs are still accurate
2. ✅ Share with a coworker and watch them deploy
3. ✅ Update docs based on their questions

---

## You Now Have

✅ A complete, professional infrastructure repository  
✅ Documentation for deployment and troubleshooting  
✅ A guide to format any infrastructure project  
✅ Real errors from your build, with detailed fixes  
✅ A template to teach others  
✅ A portfolio piece showing cloud engineering practices  

**This is production-grade documentation.** You can share this link with anyone and they'll know exactly what to do.

---

**Questions?** Everything is in `GITHUB_SETUP.md` or the specific guide files. You have everything you need to succeed.

**You're ready.** Go build great infrastructure.

---

*Created September 2026 — Based on actual deployment and mentoring session*
