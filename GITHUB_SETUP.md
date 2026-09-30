# How to Create This GitHub Repository from Scratch

This guide teaches you how to recreate the `ntfs-lab-terraform` repository structure on GitHub, formatted exactly as shown in this project.

---

## Part 1: Create the Repository on GitHub

### Step 1 — Create a GitHub Account (if you don't have one)
Go to [github.com](https://github.com) and sign up. Free tier is fine for this.

### Step 2 — Create a New Repository
1. Click the **+** icon in the top right → **New repository**
2. **Repository name:** `ntfs-lab-terraform`
3. **Description:** "Infrastructure as Code for Azure NTFS file server lab with Active Directory, security groups, and Terraform"
4. **Public** or **Private** (your choice — public is better for learning)
5. **Initialize with:**
   - ✅ Add a README.md
   - ✅ Add .gitignore → Choose template **Terraform**
   - ✅ Add a license → Choose **MIT**
6. Click **Create repository**

---

## Part 2: Set Up Your Local Folder Structure

On your machine, you'll mirror the GitHub repo structure exactly.

### Step 1 — Clone the Repository
```powershell
cd $HOME\Projects
git clone https://github.com/yourusername/ntfs-lab-terraform.git
cd ntfs-lab-terraform
```

### Step 2 — Create the Folder Structure
```powershell
# Create main directories
New-Item -ItemType Directory -Path "terraform" -Force
New-Item -ItemType Directory -Path "scripts" -Force
New-Item -ItemType Directory -Path "docs" -Force
New-Item -ItemType Directory -Path ".github/ISSUE_TEMPLATE" -Force
```

**Result:**
```
ntfs-lab-terraform/
├── .github/
│   └── ISSUE_TEMPLATE/
├── terraform/
├── scripts/
├── docs/
├── .gitignore         (auto-created by GitHub)
├── LICENSE            (auto-created by GitHub)
└── README.md          (auto-created by GitHub)
```

---

## Part 3: Add Your Files

### Step 1 — Replace the Main README
Delete the auto-generated README.md and replace it with the one from this project:

1. Open the `README.md` from this documentation
2. Copy the entire content
3. Paste into `ntfs-lab-terraform/README.md`
4. Save

### Step 2 — Add Terraform Files
Create each file in the `terraform/` folder:

**terraform/versions.tf**
```hcl
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
    time = {
      source  = "hashicorp/time"
      version = "~> 0.11"
    }
  }
}
```

(Copy the rest from the original SOP document into the remaining terraform files: `backend.tf`, `variables.tf`, `main.tf`, `keyvault.tf`, `outputs.tf`, `terraform.tfvars.example`)

### Step 3 — Add PowerShell Scripts
Create each file in the `scripts/` folder:

**scripts/00-promote-dc.ps1** → Copy from the original SOP
**scripts/01-join-domain.ps1** → Copy from the original SOP
(Continue for scripts 02–06)

### Step 4 — Add Documentation Files
Create files in the `docs/` folder:

**docs/DEPLOYMENT.md** — Step-by-step walkthrough
**docs/ERRORS.md** — Use the one from this project
**docs/ARCHITECTURE.md** — Deep dive on design decisions
**docs/LESSONS.md** — Key takeaways

### Step 5 — Enhance .gitignore
The Terraform template is good, but add these to be explicit:

```gitignore
# Terraform
terraform.tfvars
*.tfstate
*.tfstate.*
.terraform/
*.tfplan
crash.log
crash.*.log

# Environment
.env
.envrc

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Security
*.pem
*.key
secrets.*
```

### Step 6 — Add GitHub Issue Template
Create `.github/ISSUE_TEMPLATE/error-report.md`:

```markdown
---
name: Error or Issue Report
about: Something went wrong during deployment
title: '[ERROR] Brief description'
labels: 'bug'
---

## What happened?
(Describe the error)

## Error message
```
(Paste the full error here)
```

## Command you ran
```powershell
(Paste the command)
```

## Diagnostic output
```
(Run relevant commands from docs/ERRORS.md and paste output)
```

## What did you expect?
(What should have happened)

## Environment
- Terraform version: (run `terraform -version`)
- Azure CLI version: (run `az version`)
- OS: Windows / macOS / Linux
```

---

## Part 4: Organize Your README Structure

A good GitHub README is **self-documenting**. Here's how to structure it:

```
1. Title + Badge
2. Quick summary (1-2 sentences)
3. What gets created (bullet points)
4. What You'll Learn (table)
5. Prerequisites (checklist)
6. Getting Started (numbered steps)
7. Repository Structure (tree view)
8. Common Tasks (code examples)
9. Troubleshooting (link to docs/ERRORS.md)
10. Key Concepts (short explanations)
11. Contributing
12. License
```

This keeps readers from scrolling endlessly. Each section should answer a specific question the reader might have.

---

## Part 5: Commit and Push to GitHub

```powershell
cd ntfs-lab-terraform

# See what changed
git status

# Add everything
git add .

# Commit with a clear message
git commit -m "Initial commit: terraform files, scripts, and documentation"

# Push to GitHub
git push origin main
```

If `git push` fails with "permission denied", you likely need to set up authentication:

```powershell
# If you haven't configured Git yet:
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Then retry:
git push origin main
```

---

## Part 6: Markdown Formatting Tips

### Headers
```markdown
# Main title (H1 — use once at top)
## Section (H2)
### Subsection (H3)
#### Details (H4)
```

### Lists
```markdown
# Bulleted
- First item
- Second item
  - Nested
  - Item

# Numbered
1. First
2. Second
3. Third
```

### Code Blocks
**PowerShell:**
```markdown
​```powershell
$env:TF_VAR_admin_password = "YourPassword"
terraform plan
​```
```

**HCL (Terraform):**
```markdown
​```hcl
resource "azurerm_resource_group" "example" {
  name     = "my-rg"
  location = "centralus"
}
​```
```

**Plain text/output:**
```markdown
​```
$ terraform apply
Plan: 18 to add
​```
```

### Tables
```markdown
| Column 1 | Column 2 | Column 3 |
|---|---|---|
| Row 1A | Row 1B | Row 1C |
| Row 2A | Row 2B | Row 2C |
```

### Links
```markdown
[Link text](https://example.com)
[Internal link](./docs/ERRORS.md)
```

### Emphasis
```markdown
**bold text** or __bold text__
*italic text* or _italic text_
~~strikethrough~~
`inline code`
```

### Horizontal line
```markdown
---
```

---

## Part 7: File Naming Conventions

Follow these conventions for consistency:

| File Type | Convention | Example |
|---|---|---|
| Markdown docs | UPPERCASE | README.md, ERRORS.md, DEPLOYMENT.md |
| Terraform files | lowercase.tf | main.tf, variables.tf, keyvault.tf |
| PowerShell scripts | 00-name.ps1 | 00-promote-dc.ps1, 01-join-domain.ps1 |
| Example configs | name.example | terraform.tfvars.example |

---

## Part 8: Add Badges (Optional)

Badges at the top of README look professional and convey information:

```markdown
![Terraform](https://img.shields.io/badge/Terraform-1.5.0+-blue)
![PowerShell](https://img.shields.io/badge/PowerShell-5+-blue)
![Azure](https://img.shields.io/badge/Azure-Cloud-blue)
![License](https://img.shields.io/badge/License-MIT-green)
```

These render as small colored badges. Find more at [shields.io](https://shields.io).

---

## Part 9: Continuous Updates

As you refine the lab or discover new errors:

```powershell
# Edit a file
# ...make changes...

# Stage and commit
git add docs/ERRORS.md
git commit -m "Add: New error case for double .ps1 extensions"

# Push
git push origin main
```

GitHub automatically shows your commit history. Future users can see what changed and when.

---

## Part 10: README Best Practices

✅ **DO:**
- Start with a clear sentence explaining what this is
- Use headers to break up sections
- Add code examples (most readers skim, then copy-paste)
- Link to detailed docs in the `docs/` folder
- Include a troubleshooting section or link
- Make the Getting Started section copy-pasteable

❌ **DON'T:**
- Make the README too long (keep it under 1000 lines)
- Repeat content — link to `docs/` instead
- Include sensitive data (IPs, passwords, IDs)
- Leave placeholder text ("TODO: add this later")
- Use only prose — people need code examples

---

## Part 11: Repository Settings (Optional)

On GitHub, click **Settings**:

1. **Description:** Add a short tagline
2. **Topics:** Add tags like `terraform`, `azure`, `active-directory`, `lab`
3. **Require status checks before merging:** Not needed for a solo project, but good practice
4. **Automatically delete head branches:** Keeps the branch list clean

---

## Part 12: Create a .gitignore Deep Dive

You already have Terraform's template, but here's why each line matters:

```gitignore
# Local .terraform directories
.terraform/              # ← Downloaded provider binaries (huge, regenerated by init)

# .tfstate files
*.tfstate               # ← Current state (contains sensitive values)
*.tfstate.*             # ← State backups
*.tfvars                # ← Variable overrides (passwords, IPs, etc.)
!terraform.tfvars.example  # ← EXCEPT the example (safe template)

# Crash logs
crash.log
crash.*.log

# Exclude all .tfvars files, which might contain sensitive data
terraform.tfvars
terraform.tfvars.json

# Ignore plan files
*.tfplan

# IDE
.vscode/
.idea/
*.swp

# Mac
.DS_Store
```

The key: **Prevent state files and passwords from reaching GitHub.**

---

## Quick Reference: Repository Structure

This is the final layout:

```
ntfs-lab-terraform/
│
├── README.md                           ← Start here (overview + quick start)
├── LICENSE                             ← MIT license
├── .gitignore                          ← Prevents secrets leaking
│
├── terraform/
│   ├── versions.tf                     ← Provider versions
│   ├── backend.tf                      ← Remote state config
│   ├── variables.tf                    ← Input variables
│   ├── main.tf                         ← Network + 3 VMs
│   ├── keyvault.tf                     ← Key Vault + secret
│   ├── outputs.tf                      ← Output IPs
│   ├── terraform.tfvars.example        ← Safe template
│   ├── terraform.tfvars                ← Your values (gitignored)
│   └── .terraform.lock.hcl             ← Provider lock (commit this)
│
├── scripts/
│   ├── 00-promote-dc.ps1               ← Domain controller setup
│   ├── 01-join-domain.ps1              ← Domain joining
│   ├── 02-create-ous.ps1               ← OU structure
│   ├── 03-create-groups.ps1            ← Security groups
│   ├── 04-create-users.ps1             ← Test users
│   ├── 05-create-share-and-permissions.ps1  ← NTFS setup
│   └── 06-add-rdp-users.ps1            ← RDP access
│
├── docs/
│   ├── DEPLOYMENT.md                   ← Step-by-step walkthrough
│   ├── ERRORS.md                       ← All errors & fixes
│   ├── ARCHITECTURE.md                 ← Design decisions
│   └── LESSONS.md                      ← Key takeaways
│
└── .github/
    └── ISSUE_TEMPLATE/
        └── error-report.md             ← GitHub issue template
```

---

## Verification Checklist

Before considering your repo "done":

- ✅ README.md is clear and has links to docs/
- ✅ All Terraform files are in `terraform/` folder
- ✅ All PowerShell scripts are in `scripts/` folder, numbered 00-06
- ✅ Documentation is in `docs/` folder
- ✅ .gitignore prevents `terraform.tfvars` and `*.tfstate` from committing
- ✅ `terraform.tfvars.example` IS committed (it's safe)
- ✅ `.terraform.lock.hcl` IS committed (it pins provider versions)
- ✅ No passwords, IPs, or sensitive data in any file
- ✅ All markdown files have proper headers and code examples
- ✅ At least one commit message per section
- ✅ GitHub shows your commits in the commit history

---

## Next Steps

1. **Invite collaborators** (optional): Settings → Collaborators
2. **Enable branch protection** (optional): Settings → Branches → Add rule
3. **Enable GitHub Pages** (optional): Makes a website from your README
4. **Monitor issues**: People can report errors using the issue template

---

## One More Thing: Keep It Updated

As you run this lab and find new errors or improvements:

```powershell
# Update your local repo
git pull origin main

# Make changes
# ...edit files...

# Commit
git add .
git commit -m "Update: Add error case for double .ps1 extensions"
git push origin main
```

Your repo becomes a living document. Future you (or someone learning from you) will thank you for detailed commit messages and clear documentation.

---

## Resources

- [GitHub Markdown Guide](https://guides.github.com/features/mastering-markdown/)
- [Terraform Style Guide](https://developer.hashicorp.com/terraform/language/style)
- [Git Cheat Sheet](https://github.github.com/training-kit/downloads/github-git-cheat-sheet.pdf)
- [GitHub Issues Best Practices](https://guides.github.com/features/issues/)

---

**You now know how to structure a professional infrastructure repository on GitHub.** This same pattern works for any Terraform project, any size.
