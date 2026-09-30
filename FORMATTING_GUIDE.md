# How to Format GitHub Repositories for Cloud Infrastructure Projects

This guide teaches you **why and how** to format infrastructure repositories the way this project is formatted. These principles apply to any Terraform project, Ansible playbook, or infrastructure-as-code work.

---

## Principle 1: Separate Code by Type

**Why:** Each file type has different purposes and audiences.
- Terraform files describe infrastructure — read by operators and reviewed in planning
- Scripts describe behavior — read while troubleshooting
- Docs explain everything — read by beginners

**How:**
```
project/
├── terraform/          ← Infrastructure definitions only
├── scripts/            ← Configuration & setup scripts
├── docs/               ← Everything else (guides, troubleshooting, lessons)
└── .github/            ← GitHub-specific files (issue templates, workflows)
```

**Result:** Someone reading your repo immediately knows where to look for what.

---

## Principle 2: README as the Front Door

**Why:** Most people will see your README and nothing else. If it doesn't hook them, they won't dig deeper.

**Structure:**
```
1. One-sentence hook (What is this?)
2. What it creates (Bullet points)
3. Why it matters (Table of skills + context)
4. Prerequisites (Checklist)
5. Getting started (Copy-paste steps)
6. Folder structure (Tree view)
7. Key concepts (Short explanations)
8. Troubleshooting link
9. Contributing / License
```

**Example bad README:**
```markdown
# Infrastructure Lab

This lab builds stuff with Terraform.

See the docs for more info.
```

**Example good README:**
```markdown
# NTFS File Server Lab

Terraform + Azure AD infrastructure demonstrating file server permissions management 
in a production-like environment. Deploy in 90 minutes, tear down in 5 commands.

**What gets created:**
- Domain controller with lab.local forest
- File server with NTFS-secured shares
- Windows 11 workstation for testing

**Key learning:**
| Skill | Why |
|---|---|
| Infrastructure as Code | Rebuild identically, every time |
| Security groups | Scale without editing permissions |
| Secret management | Keep passwords in Key Vault, not files |

**Get started:** See "Getting Started" below

**Stuck?** See docs/ERRORS.md
```

The good one answers questions without making you read other files first.

---

## Principle 3: Modularize Terraform Files

**Why:** Each `.tf` file should have one logical responsibility. This makes changes atomic and reviewable.

**Don't do this:**
```
terraform/
└── everything.tf  ← 500 lines, mixing network, VMs, and security
```

**Do this:**
```
terraform/
├── versions.tf           ← Provider version constraints
├── backend.tf            ← Remote state config
├── variables.tf          ← Input variables & defaults
├── main.tf               ← Network, VMs, NSG (compute layer)
├── keyvault.tf           ← Key Vault & secrets (security layer)
├── outputs.tf            ← Exported values
├── terraform.tfvars.example
└── .terraform.lock.hcl
```

**Why this works:**
- Changes to compute (main.tf) won't accidentally change security (keyvault.tf)
- Code review is faster — reviewers see related changes together
- Collaboration is easier — two people can edit different layers without merge conflicts
- It's obvious where to add new resources — network stuff goes in main.tf

---

## Principle 4: Number Scripts in Order

**Why:** Execution order matters. Numbering makes it obvious.

**Don't do this:**
```
scripts/
├── promote-dc.ps1
├── join-domain.ps1
├── create-users.ps1
├── create-share.ps1
```

**Do this:**
```
scripts/
├── 00-promote-dc.ps1
├── 01-join-domain.ps1
├── 02-create-ous.ps1
├── 03-create-groups.ps1
├── 04-create-users.ps1
├── 05-create-share-and-permissions.ps1
└── 06-add-rdp-users.ps1
```

**Why:**
- A new reader sees the sequence immediately
- You can't accidentally run them out of order (domain controller must be first)
- Tools can sort them alphabetically and still get the right order
- In documentation, you can say "Run 01 after 00" instead of explaining dependencies

---

## Principle 5: Make Filenames Descriptive

**Naming:**
- ✅ `00-promote-dc.ps1` — Clear: creates domain controller
- ❌ `setup.ps1` — Ambiguous: sets up what?

- ✅ `terraform.tfvars.example` — Clear: example, safe to commit
- ❌ `tfvars` — Ambiguous: is this a file or a folder?

- ✅ `ERRORS.md` — Clear: all common errors
- ❌ `README.md` in docs/ — Ambiguous: readme for what?

---

## Principle 6: Document Before, During, After

**Before deployment:**
```
docs/
├── ARCHITECTURE.md       ← Why the design looks this way
└── PREREQUISITES.md      ← What you need beforehand
```

**During deployment:**
```
README.md → "Getting Started" section
  (copy-paste commands, reference docs/DEPLOYMENT.md for details)
```

**After deployment:**
```
docs/
├── ERRORS.md            ← What went wrong and why
├── LESSONS.md           ← What to take forward
└── VERIFICATION.md      ← How to test it works
```

**Why this structure:**
- Readers know what to expect upfront (ARCHITECTURE)
- Readers can follow along step-by-step (DEPLOYMENT)
- Readers have a lifeline if something breaks (ERRORS)

---

## Principle 7: Comment Your Code

**Terraform comments:**
```hcl
# A short description of what this resource does
resource "azurerm_resource_group" "lab" {
  name     = var.resource_group_name
  location = var.location
}

# Why this has to be static IP — DNS resolution depends on it
resource "azurerm_network_interface" "dc01" {
  ...
  ip_configuration {
    private_ip_address_allocation = "Static"
    private_ip_address            = var.dc01_private_ip  # ← Must not change
  }
}
```

**PowerShell comments:**
```powershell
# Step 1: Install the AD DS role — it doesn't exist on fresh Windows
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Step 2: Now the promotion cmdlets are available
Import-Module ADDSDeployment

# Step 3: Create the forest, domain, and DNS in one operation
Install-ADDSForest `
    -DomainName $DomainName `
    ...
```

**Why:**
- Six months from now, you'll wonder "why is this static?"
- Comments explain intention, not syntax
- New readers get context immediately

---

## Principle 8: Make README Scannable

**Use subheadings:** Readers skim first. Subheadings guide them to the section they need.

**Use tables:** Tabular data is faster than prose.

```markdown
# Bad — paragraph
This lab teaches Terraform, which is an infrastructure-as-code tool.
Active Directory is important because it manages permissions and authentication.
NTFS is how Windows secures files and folders.

# Good — table
| Concept | What It Is | Why It Matters |
|---|---|---|
| Terraform | IaC tool | Rebuild infrastructure identically |
| Active Directory | Identity service | Manage permissions for users and groups |
| NTFS | File security | Control access at folder level |
```

**Use code blocks:** Readers expect to copy-paste from docs. Make it easy:
```markdown
# Good — copy-paste ready
terraform init
terraform plan
terraform apply

# Bad — embedded in prose
To deploy, run terraform init to initialize the working directory, 
then terraform plan to see what will change, then terraform apply to deploy it.
```

**Use links:**
```markdown
# Good — readers can jump
See docs/ERRORS.md for 20+ common errors and fixes.

# Bad — readers have to search
There's a troubleshooting section that lists errors.
```

---

## Principle 9: Version Lock Your Dependencies

**Why:** Terraform and PowerShell versions change. Code that works today might break tomorrow.

**Do this:**
```hcl
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
  }
}
```

**Commit .terraform.lock.hcl:**
```gitignore
# No — lock file is regenerated every time
# Yes — lock file pins exact provider versions for team consistency
.terraform.lock.hcl

# Also add this to .gitignore:
.terraform/     # Downloaded binaries, not needed in repo
```

**Why:**
- New team member clones → runs `terraform init` → gets exact same provider versions
- You rebuild in 6 months → exact same versions → no surprises
- CI/CD pipeline runs → exact same versions → reproducible

---

## Principle 10: Docs Folder Has a Clear Index

**Don't:**
```
docs/
├── something.md
├── info.md
├── details.md
```

**Do:**
```
docs/
├── README.md             ← Index: "Start here for docs overview"
├── DEPLOYMENT.md         ← How to deploy step-by-step
├── ERRORS.md             ← Common errors & fixes
├── ARCHITECTURE.md       ← Design decisions & why
├── VERIFICATION.md       ← How to test it works
└── LESSONS.md            ← Key takeaways
```

**The docs README:**
```markdown
# Documentation Index

**New to this lab?** Start here:
1. Read [ARCHITECTURE.md](./ARCHITECTURE.md) — understand the design
2. Follow [DEPLOYMENT.md](./DEPLOYMENT.md) — deploy step-by-step
3. Check [ERRORS.md](./ERRORS.md) — when something breaks

**Experienced?**
- [VERIFICATION.md](./VERIFICATION.md) — test each step
- [LESSONS.md](./LESSONS.md) — key concepts

**Specific error?** Search [ERRORS.md](./ERRORS.md)
```

---

## Principle 11: Make Examples Copy-Paste Ready

**Bad example:**
```
To deploy, use these commands:
  terraform init
  terraform plan
  terraform apply
```

**Good example:**
```powershell
terraform init
terraform plan
terraform apply
```

**Why:** Readers copy your code. Make it literally copy-paste-able — no translation needed.

---

## Principle 12: .gitignore Prevents Accidents

**Critical items to exclude:**
```gitignore
# Secrets
terraform.tfvars          # Your passwords, IP addresses
*.pem                     # Private keys
.env                      # Environment variables
secrets.*

# State files (contain sensitive values)
*.tfstate
*.tfstate.*

# Downloaded provider binaries
.terraform/

# IDE clutter
.vscode/
.idea/
```

**Safe to commit:**
```gitignore
# EXCEPT these — they're safe and needed
!terraform.tfvars.example   # Safe template
.terraform.lock.hcl         # Version lock (commit this)
```

**Pro tip:** Use a separate `.env` file for local-only values, then `.gitignore` it:
```bash
# .env (not in git)
export AZURE_SUBSCRIPTION_ID="..."

# Then source it when you need it
source .env
```

---

## Principle 13: Use Consistent Formatting

### Markdown
```markdown
# H1 — once, at the top
## H2 — sections
### H3 — subsections

- Bullet points
  - Nested
  - Items

1. Numbered
2. Lists
3. For sequences

| Table | Format | Looks | Professional |
|---|---|---|---|
```

### Terraform
```hcl
# Use 2-space indentation (Terraform standard)
# Use descriptive resource names
resource "azurerm_resource_group" "lab" {
  name     = var.resource_group_name
  location = var.location
}

# Align equals signs for readability
variable "admin_password" {
  description = "Local admin password for every VM"
  type        = string
  sensitive   = true
}
```

### PowerShell
```powershell
# Use PascalCase for function/cmdlet names
# Use camelCase for variables
# Backtick for line continuation
Install-ADDSForest `
    -DomainName $DomainName `
    -SafeModeAdministratorPassword $secureSafeModePassword `
    -InstallDns `
    -Force
```

---

## Principle 14: README Maintenance Checklist

Every time you update your repo:

- ✅ Is the README still accurate?
- ✅ Do all links still work?
- ✅ Are code examples still correct?
- ✅ Have prerequisites changed?
- ✅ Does the folder structure diagram match reality?

---

## Quick Checklist: Is Your Repo Well-Formatted?

- ✅ README has sections, not just prose
- ✅ README has copy-paste code examples
- ✅ README links to detailed docs
- ✅ Terraform files are split by layer (versions, backend, variables, main, security, outputs)
- ✅ Scripts are numbered in execution order
- ✅ Docs folder has an index
- ✅ All files have comments explaining "why"
- ✅ .gitignore prevents secrets from leaking
- ✅ Filenames are descriptive
- ✅ Code is consistently formatted

---

## The Meta-Principle

> **Format your repository for someone who's never seen it before.**

That someone is:
- A new team member who needs to deploy tomorrow
- You, six months from now, forgetting what you did
- Someone learning from your code

Format for them. Use clear structure, links, and examples. They'll thank you, and so will future-you.

---

## Real-World Patterns

### Pattern 1: Simple Lab (like this one)
```
project/
├── README.md          ← Step-by-step for first-timers
├── terraform/         ← All infrastructure
├── scripts/           ← Setup scripts
└── docs/              ← Deep dives and troubleshooting
```

### Pattern 2: Multi-Environment
```
project/
├── README.md
├── terraform/
│   ├── dev/           ← Development environment
│   ├── prod/          ← Production environment
│   └── shared/        ← Shared modules
└── docs/
```

### Pattern 3: Monorepo (Many Projects)
```
monorepo/
├── README.md          ← Index of all projects
├── project-a/         ← Each has its own structure
├── project-b/
└── docs/              ← Shared documentation
```

---

## Tools to Help With Formatting

- **Markdown preview:** VS Code with Markdown Preview Enhanced
- **Terraform formatting:** `terraform fmt` (built-in)
- **Linting:** `terraform validate` and `tflint`
- **Diagram makers:** https://mermaid.live (embed diagrams in markdown)

---

**Remember:** Your repository is documentation. Make it clear, scannable, and complete.

The best infrastructure code is code that the next person can understand and use immediately.

