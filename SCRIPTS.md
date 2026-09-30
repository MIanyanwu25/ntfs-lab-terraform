# PowerShell Scripts for NTFS File Server Lab

This file contains all 7 configuration scripts (00-06) used to deploy the lab infrastructure. Each script is clearly partitioned and can be extracted as a standalone `.ps1` file.

**Usage:** These scripts are deployed via `az vm run-command` and run on the VMs themselves (not locally).

---

## 📋 Scripts Index

1. [00-promote-dc.ps1](#00-promote-dcps1) — Promote DC01 to domain controller
2. [01-join-domain.ps1](#01-join-domainps1) — Join FS01 & CLIENT01 to domain
3. [02-create-ous.ps1](#02-create-oups1) — Create OU structure
4. [03-create-groups.ps1](#03-create-groupsps1) — Create security groups
5. [04-create-users.ps1](#04-create-usersps1) — Create test users
6. [05-create-share-and-permissions.ps1](#05-create-share-and-permissionsps1) — Set up shares and NTFS
7. [06-add-rdp-users.ps1](#06-add-rdp-usersps1) — Grant RDP access

---

# 00-promote-dc.ps1

**Target:** DC01  
**Purpose:** Install Active Directory and create lab.local forest  
**Execution:** Script 00 must run first, before any other scripts  
**Notes:** The server reboots automatically. Wait 10 minutes before running Script 01.

```powershell
param(
    [Parameter(Mandatory = $true)]
    [string]$SafeModePassword,
    
    [string]$DomainName = "lab.local",
    [string]$DomainNetbiosName = "LAB"
)

# Convert plain text password to SecureString for AD DS cmdlets
$secureSafeModePassword = ConvertTo-SecureString $SafeModePassword -AsPlainText -Force

# Install the Active Directory Domain Services role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Import the ADDSDeployment module (becomes available after role installation)
Import-Module ADDSDeployment

# Promote this server to a domain controller
# Creates the lab.local forest, domain, and DNS service
Install-ADDSForest `
    -DomainName $DomainName `
    -DomainNetbiosName $DomainNetbiosName `
    -SafeModeAdministratorPassword $secureSafeModePassword `
    -InstallDns `
    -Force `
    -NoReboot

# Schedule an automatic reboot (gives time for DNS to initialize)
shutdown /r /t 120 /c "AD DS installation complete, rebooting in 2 minutes"
```

**What it does:**
1. Converts password to SecureString (required by AD DS)
2. Installs AD DS and DNS roles
3. Creates lab.local forest and domain
4. Reboots automatically after 2 minutes

**Verification (after reboot):**
```powershell
Get-WindowsFeature AD-Domain-Services
(Get-CimInstance Win32_ComputerSystem).Domain  # Should return: lab.local
Get-Service DNS  # Should show: Running
```

---

# 01-join-domain.ps1

**Target:** FS01, CLIENT01  
**Purpose:** Join FS01 and CLIENT01 to the lab.local domain  
**Dependencies:** Script 00 must complete and DNS must be ready (wait 5-10 min)  
**Execution:** Run for FS01, then for CLIENT01. Both reboot automatically.

```powershell
param(
    [Parameter(Mandatory = $true)]
    [string]$DomainName,
    
    [Parameter(Mandatory = $true)]
    [string]$AdminUsername,
    
    [Parameter(Mandatory = $true)]
    [string]$AdminPassword
)

# Convert password to SecureString for domain join cmdlet
$securePassword = ConvertTo-SecureString $AdminPassword -AsPlainText -Force

# Create PSCredential object for domain join
$credential = New-Object System.Management.Automation.PSCredential `
    -ArgumentList "$DomainName\$AdminUsername", $securePassword

# Join this computer to the domain
Add-Computer `
    -DomainName $DomainName `
    -Credential $credential `
    -Restart
```

**What it does:**
1. Converts password to SecureString
2. Creates credential object with domain\username format
3. Joins the machine to lab.local domain
4. Reboots automatically

**Verification (after reboot):**
```powershell
(Get-CimInstance Win32_ComputerSystem).Domain  # Should return: lab.local
(Get-CimInstance Win32_ComputerSystem).PartOfDomain  # Should return: True
```

---

# 02-create-ous.ps1

**Target:** DC01  
**Purpose:** Create the OU structure in Active Directory  
**Dependencies:** Script 01 must complete (domain joins)  
**Execution:** Run once on DC01

```powershell
param(
    [string]$DomainName = "lab.local"
)

# Extract the distinguished name from domain name
# lab.local becomes: DC=lab,DC=local
$dn = ($DomainName -split '\.') | ForEach-Object { "DC=$_" }
$baseDn = $dn -join ','

# Create the root OU for this lab
New-ADOrganizationalUnit `
    -Name "FileServerLab" `
    -Path $baseDn

# Create sub-OUs for groups and users
New-ADOrganizationalUnit `
    -Name "Groups" `
    -Path "OU=FileServerLab,$baseDn"

New-ADOrganizationalUnit `
    -Name "Users" `
    -Path "OU=FileServerLab,$baseDn"
```

**What it does:**
1. Calculates the distinguished name (DN) for the domain
2. Creates OU=FileServerLab at domain root
3. Creates OU=Groups under FileServerLab
4. Creates OU=Users under FileServerLab

**Verification:**
```powershell
Get-ADOrganizationalUnit -Filter "Name -eq 'FileServerLab'"
Get-ADOrganizationalUnit -Filter "Name -eq 'Groups'" -SearchBase "OU=FileServerLab,DC=lab,DC=local"
Get-ADOrganizationalUnit -Filter "Name -eq 'Users'" -SearchBase "OU=FileServerLab,DC=lab,DC=local"
```

---

# 03-create-groups.ps1

**Target:** DC01  
**Purpose:** Create security groups for permission management  
**Dependencies:** Script 02 must complete (OUs created)  
**Execution:** Run once on DC01

```powershell
param(
    [string]$DomainName = "lab.local"
)

# Distinguished name for the Groups OU
$groupsOuDn = "OU=Groups,OU=FileServerLab,DC=lab,DC=local"

# Create Finance groups
New-ADGroup `
    -Name "GG-Finance-ReadOnly" `
    -Path $groupsOuDn `
    -GroupScope Global `
    -GroupCategory Security `
    -Description "Finance department - Read only access to CompanyData/Finance"

New-ADGroup `
    -Name "GG-Finance-Modify" `
    -Path $groupsOuDn `
    -GroupScope Global `
    -GroupCategory Security `
    -Description "Finance department - Read and modify access to CompanyData/Finance"

# Create HR groups
New-ADGroup `
    -Name "GG-HR-ReadOnly" `
    -Path $groupsOuDn `
    -GroupScope Global `
    -GroupCategory Security `
    -Description "HR department - Read only access to CompanyData/HR"

New-ADGroup `
    -Name "GG-HR-FullControl" `
    -Path $groupsOuDn `
    -GroupScope Global `
    -GroupCategory Security `
    -Description "HR department - Full control of CompanyData/HR"
```

**What it does:**
1. Creates 4 security groups in OU=Groups
2. Two groups for Finance (ReadOnly, Modify)
3. Two groups for HR (ReadOnly, FullControl)
4. All groups are Global scope (can be used across domains)

**Verification:**
```powershell
Get-ADGroup -Filter "Name -like 'GG-*'" | Select-Object Name, Description
```

---

# 04-create-users.ps1

**Target:** DC01  
**Purpose:** Create test users and assign them to groups  
**Dependencies:** Script 03 must complete (groups created)  
**Execution:** Run once on DC01

```powershell
param(
    [Parameter(Mandatory = $true)]
    [string]$UserPassword
)

# Convert password to SecureString
$securePassword = ConvertTo-SecureString $UserPassword -AsPlainText -Force

# Distinguished name for the Users OU
$usersOuDn = "OU=Users,OU=FileServerLab,DC=lab,DC=local"

# Create Finance users
New-ADUser `
    -Name "alice.finance" `
    -SamAccountName "alice.finance" `
    -UserPrincipalName "alice.finance@lab.local" `
    -Path $usersOuDn `
    -AccountPassword $securePassword `
    -Enabled $true `
    -PasswordNotRequired $false

New-ADUser `
    -Name "brian.finance" `
    -SamAccountName "brian.finance" `
    -UserPrincipalName "brian.finance@lab.local" `
    -Path $usersOuDn `
    -AccountPassword $securePassword `
    -Enabled $true `
    -PasswordNotRequired $false

# Create HR users
New-ADUser `
    -Name "carla.hr" `
    -SamAccountName "carla.hr" `
    -UserPrincipalName "carla.hr@lab.local" `
    -Path $usersOuDn `
    -AccountPassword $securePassword `
    -Enabled $true `
    -PasswordNotRequired $false

New-ADUser `
    -Name "david.hr" `
    -SamAccountName "david.hr" `
    -UserPrincipalName "david.hr@lab.local" `
    -Path $usersOuDn `
    -AccountPassword $securePassword `
    -Enabled $true `
    -PasswordNotRequired $false

# Add users to groups
Add-ADGroupMember -Identity "GG-Finance-ReadOnly" -Members "alice.finance"
Add-ADGroupMember -Identity "GG-Finance-ReadOnly" -Members "brian.finance"
Add-ADGroupMember -Identity "GG-Finance-Modify" -Members "brian.finance"

Add-ADGroupMember -Identity "GG-HR-ReadOnly" -Members "carla.hr"
Add-ADGroupMember -Identity "GG-HR-FullControl" -Members "david.hr"
```

**What it does:**
1. Creates 4 test users (alice.finance, brian.finance, carla.hr, david.hr)
2. Assigns users to groups
3. **Note:** brian.finance is in TWO groups to test cumulative permissions

**Group membership:**
| User | Groups |
|---|---|
| alice.finance | GG-Finance-ReadOnly |
| brian.finance | GG-Finance-ReadOnly + GG-Finance-Modify |
| carla.hr | GG-HR-ReadOnly |
| david.hr | GG-HR-FullControl |

**Verification:**
```powershell
Get-ADUser -Filter "SamAccountName -like '*finance' -or SamAccountName -like '*hr'" | Select-Object Name, Enabled
Get-ADGroupMember "GG-Finance-ReadOnly" | Select-Object Name
Get-ADGroupMember "GG-Finance-Modify" | Select-Object Name
```

---

# 05-create-share-and-permissions.ps1

**Target:** FS01  
**Purpose:** Create the CompanyData share with NTFS permissions  
**Dependencies:** Script 04 must complete (users created)  
**Execution:** Run once on FS01

```powershell
param(
    [string]$SharePath = "C:\CompanyData"
)

# Create the root share folder if it doesn't exist
if (-not (Test-Path $SharePath)) {
    New-Item -ItemType Directory -Path $SharePath -Force | Out-Null
}

# Create subfolders
New-Item -ItemType Directory -Path "$SharePath\Finance" -Force | Out-Null
New-Item -ItemType Directory -Path "$SharePath\HR" -Force | Out-Null

# Create the SMB share (wide open on purpose; NTFS is the real gate)
New-SmbShare `
    -Name "CompanyData" `
    -Path $SharePath `
    -FullAccess "Everyone" `
    -Description "Shared folder for Finance and HR departments"

# ===== FINANCE FOLDER NTFS PERMISSIONS =====
# First, remove inherited permissions
$acl = Get-Acl "$SharePath\Finance"
$acl.Access | ForEach-Object { $acl.RemoveAccessRule($_) }

# Grant SYSTEM (required for Windows operations)
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "SYSTEM",
    "FullControl",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)
$acl.AddAccessRule($rule)

# Grant Domain Admins
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "LAB\Domain Admins",
    "FullControl",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)
$acl.AddAccessRule($rule)

# Grant GG-Finance-ReadOnly (Read + Execute)
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "LAB\GG-Finance-ReadOnly",
    "ReadAndExecute",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)
$acl.AddAccessRule($rule)

# Grant GG-Finance-Modify (Modify = Read + Write + Delete)
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "LAB\GG-Finance-Modify",
    "Modify",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)
$acl.AddAccessRule($rule)

Set-Acl "$SharePath\Finance" $acl

# ===== HR FOLDER NTFS PERMISSIONS =====
$acl = Get-Acl "$SharePath\HR"
$acl.Access | ForEach-Object { $acl.RemoveAccessRule($_) }

# Grant SYSTEM
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "SYSTEM",
    "FullControl",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)
$acl.AddAccessRule($rule)

# Grant Domain Admins
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "LAB\Domain Admins",
    "FullControl",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)
$acl.AddAccessRule($rule)

# Grant GG-HR-ReadOnly (Read + Execute)
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "LAB\GG-HR-ReadOnly",
    "ReadAndExecute",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)
$acl.AddAccessRule($rule)

# Grant GG-HR-FullControl (Full Control)
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "LAB\GG-HR-FullControl",
    "FullControl",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)
$acl.AddAccessRule($rule)

Set-Acl "$SharePath\HR" $acl
```

**What it does:**
1. Creates C:\CompanyData folder structure
   - C:\CompanyData\Finance
   - C:\CompanyData\HR
2. Creates SMB share (Everyone Full Control) — wide open
3. Strips inherited NTFS permissions
4. Applies explicit NTFS ACLs per group
5. NTFS is the real access control point

**Permission summary:**
```
Finance folder:
  SYSTEM: Full Control
  Domain Admins: Full Control
  GG-Finance-ReadOnly: Read + Execute
  GG-Finance-Modify: Modify (Read + Write + Delete)

HR folder:
  SYSTEM: Full Control
  Domain Admins: Full Control
  GG-HR-ReadOnly: Read + Execute
  GG-HR-FullControl: Full Control
```

**Verification:**
```powershell
Get-SmbShare -Name "CompanyData" | Select-Object Name, Path
icacls C:\CompanyData\Finance
icacls C:\CompanyData\HR
```

---

# 06-add-rdp-users.ps1

**Target:** CLIENT01  
**Purpose:** Add test users to Remote Desktop Users group  
**Dependencies:** Script 04 must complete (users created), users must be able to log in  
**Execution:** Run once on CLIENT01

```powershell
# Add test users to the local Remote Desktop Users group
# This allows them to RDP into CLIENT01

# Get the RDP group (name varies by OS language, so use SID)
$rdpGroupSid = "S-1-5-32-555"
$rdpGroup = [System.Security.Principal.SecurityIdentifier]::new($rdpGroupSid)
$rdpGroupName = $rdpGroup.Translate([System.Security.Principal.NTAccount]).Value

# Create a directory entry for the group
$localGroup = [ADSI]"WinNT://$env:COMPUTERNAME/Remote Desktop Users,group"

# Add each test user
$users = @(
    "LAB\alice.finance",
    "LAB\brian.finance",
    "LAB\carla.hr",
    "LAB\david.hr"
)

foreach ($user in $users) {
    try {
        $userPath = "WinNT://lab/$($user.Split('\')[1])"
        $localGroup.Add($userPath)
        Write-Output "Added $user to Remote Desktop Users"
    }
    catch {
        Write-Output "Error adding $user : $_"
    }
}

# Verify
Write-Output "`nCurrent Remote Desktop Users:"
Get-LocalGroupMember -Group "Remote Desktop Users" | Select-Object Name, ObjectClass
```

**What it does:**
1. Gets the local Remote Desktop Users group (SID-based lookup)
2. Adds all 4 test users to the group
3. Verifies the additions
4. Handles errors gracefully

**After this script:**
- Users can RDP into CLIENT01
- They can test access to \\FS01\CompanyData

**Verification:**
```powershell
Get-LocalGroupMember -Group "Remote Desktop Users"
# Should show all 4 users from LAB domain
```

---

## Deployment Order (Critical)

```
1. Script 00: promote-dc.ps1 (DC01)
   └─ Wait 10 minutes for DNS to be ready

2. Script 01: join-domain.ps1 (FS01)
   └─ Wait for reboot

3. Script 01: join-domain.ps1 (CLIENT01)
   └─ Wait for reboot

4. Script 02: create-ous.ps1 (DC01)

5. Script 03: create-groups.ps1 (DC01)

6. Script 04: create-users.ps1 (DC01)

7. Script 05: create-share-and-permissions.ps1 (FS01)

8. Script 06: add-rdp-users.ps1 (CLIENT01)

9. Verification: RDP into CLIENT01 and test as each user
```

---

## How to Extract Individual Scripts

Each script can be copied from this file and saved as a standalone `.ps1` file:

```powershell
# For example, to extract Script 00:
1. Find the "# 00-promote-dc.ps1" section
2. Copy everything from the opening ``` to the closing ```
3. Paste into a new file named "00-promote-dc.ps1"
4. Save in your scripts/ folder
```

Or use this PowerShell to extract all scripts:

```powershell
# This creates individual .ps1 files from the markdown
$markdownPath = "SCRIPTS.md"
$scriptFolder = "scripts"

# PowerShell would parse the markdown and extract code blocks
# (Implementation left as exercise)
```

---

## Quick Reference: What Each Script Does

| # | Script | Target | Action |
|---|---|---|---|
| 00 | promote-dc.ps1 | DC01 | Install AD DS, create forest |
| 01 | join-domain.ps1 | FS01, CLIENT01 | Join to lab.local domain |
| 02 | create-ous.ps1 | DC01 | Create OU structure |
| 03 | create-groups.ps1 | DC01 | Create security groups |
| 04 | create-users.ps1 | DC01 | Create users & group membership |
| 05 | create-share-and-permissions.ps1 | FS01 | Create share & NTFS permissions |
| 06 | add-rdp-users.ps1 | CLIENT01 | Add users to Remote Desktop group |

