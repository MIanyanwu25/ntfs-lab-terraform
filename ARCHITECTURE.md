# Architecture: NTFS File Server Lab

**Complete technical design of the lab environment, why each component exists, and how data flows.**

---

## Overview Diagram

```
<img width="1050" height="591" alt="image" src="https://github.com/user-attachments/assets/6a5e2da4-fe8f-4764-870d-415237bd6a26" />

┌─────────────────────────────────────────────────────────────────────┐
│                          AZURE CLOUD                                │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │           Virtual Network: 10.0.0.0/16                       │   │
│  │           DNS Servers: 10.0.1.4 (DC01)                       │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐     │   │
│  │  │      Subnet: 10.0.1.0/24                            │     │   │
│  │  │  (All three VMs on same subnet)                     │     │   │
│  │  │                                                     │     │   │
│  │  │  ┌──────────────────────────────────────────────┐   │     │   │
│  │  │  │ DC01 (Domain Controller)                     │   │     │   │
│  │  │  │ • Private IP: 10.0.1.4 (STATIC)              │   │     │   │
│  │  │  │ • Public IP: <allocated>                     │   │     │   │
│  │  │  │ • Roles: AD DS, DNS, DHCP                    │   │     │   │
│  │  │  │ • Domain: lab.local (forest root)            │   │     │   │
│  │  │  └──────────────────────────────────────────────┘   │     │   │
│  │  │                                                     │     │   │
│  │  │  ┌──────────────────────────────────────────────┐   │     │   │
│  │  │  │ FS01 (File Server)                           │   │     │   │
│  │  │  │ • Private IP: 10.0.1.x (Dynamic)             │   │     │   │
│  │  │  │ • Public IP: <allocated>                     │   │     │   │
│  │  │  │ • Shares: CompanyData (Finance, HR folders)  │   │     │   │
│  │  │  │ • NTFS Permissions: per security group       │   │     │   │
│  │  │  └──────────────────────────────────────────────┘   │     │   │
│  │  │                                                     │     │   │
│  │  │  ┌──────────────────────────────────────────────┐   │     │   │
│  │  │  │ CLIENT01 (Windows 11 Workstation)            │   │     │   │
│  │  │  │ • Private IP: 10.0.1.x (Dynamic)             │   │     │   │
│  │  │  │ • Public IP: <allocated>                     │   │     │   │
│  │  │  │ • Test user logins: alice, brian, carla, david   │     │   │
│  │  │  └──────────────────────────────────────────────┘   │     │   │
│  │  │                                                     │     │   │
│  │  │              ┌────────────────────┐                 │     │   │
│  │  │              │   NSG (Security)   │                 │     │   │
│  │  │              │  RDP 3389 from     │                 │     │   │
│  │  │              │  your IP only      │                 │     │   │
│  │  │              │ Everything blocked │                 │     │   │
│  │  │              └────────────────────┘                 │     │   │
│  │  └─────────────────────────────────────────────────────┘     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────┐         ┌───────────────────────────┐     │
│  │   Key Vault          │         │  rg-tfstate (Storage)     │     │
│  │ (Secure storage)     │         │  Terraform state file     │     │
│  │                      │         │  (Lab 2 also uses this)   │     │
│  │ • VM admin password  │         └───────────────────────────┘     │
│  │ • Lab 2 secrets      │                                           │  
│  └──────────────────────┘                                           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────┐
│   Your Local Machine                │
│                                     │
│  Terraform (Infrastructure as Code) │
│  PowerShell (Configuration Scripts) │
│  Azure CLI (Control Plane)          │
│                                     │
│  • terraform/ folder                │
│  • scripts/ folder                  │
│  • Git history                      │
└─────────────────────────────────────┘
```

---

## Component Details

### DC01 — Domain Controller

**Role:** Creates and manages the lab.local Active Directory forest

**What it does:**
- Hosts Active Directory (AD DS)
- Runs DNS for lab.local domain
- Manages users and security groups
- Authenticates domain joins

**Critical design choice:**
- **Static IP (10.0.1.4)** — Every VM uses this IP for DNS. If it changed, domain resolution breaks.

**What lives on it:**
```
AD Structure:
OU=FileServerLab
├── OU=Groups
│   ├── GG-Finance-ReadOnly
│   ├── GG-Finance-Modify
│   ├── GG-HR-ReadOnly
│   └── GG-HR-FullControl
└── OU=Users
    ├── alice.finance (GG-Finance-ReadOnly)
    ├── brian.finance (GG-Finance-ReadOnly + GG-Finance-Modify)
    ├── carla.hr (GG-HR-ReadOnly)
    └── david.hr (GG-HR-FullControl)
```

**Deployment:**
1. Terraform creates the VM
2. Script 00 promotes it to DC
3. DC reboots automatically
4. DNS service starts serving lab.local

---

### FS01 — File Server

**Role:** Hosts shared folders with NTFS-controlled access

**What it does:**
- Shares `CompanyData` folder (SMB share, wide open)
- Creates subfolders: Finance and HR
- Applies NTFS permissions per security group
- Serves test files for permission verification

**Permission Model:**

```
Share Level (SMB):
CompanyData → Everyone: Full Control
              (wide open on purpose)
                ↓
NTFS Level (File System):
Finance folder → GG-Finance-ReadOnly: (RX) — Read + Execute
              → GG-Finance-Modify: (M) — Modify
              → SYSTEM: (F) — Full Control
              → Domain Admins: (F) — Full Control

HR folder    → GG-HR-ReadOnly: (RX) — Read + Execute
              → GG-HR-FullControl: (F) — Full Control
              → SYSTEM: (F) — Full Control
              → Domain Admins: (F) — Full Control
```

**Why this design:**
- Share permissions are intentionally wide
- NTFS is the single gate that actually controls access
- "Most restrictive wins" rule makes NTFS the enforcement point
- If you need to change access, you only touch one place: NTFS

---

### CLIENT01 — Windows 11 Workstation

**Role:** Test client for verifying permissions work correctly

**What it does:**
- Joins lab.local domain
- Allows test users to RDP in
- Test users access \\FS01\CompanyData
- Verify each user gets exactly their expected access

**Test sequence:**
```
Sign in as alice.finance
  → Can access Finance (read only)
  → Cannot access HR (Access Denied)
  
Sign in as brian.finance
  → Can access Finance (read + modify)
  → Cannot access HR (Access Denied)
  
Sign in as carla.hr
  → Cannot access Finance (Access Denied)
  → Can access HR (read only)
  
Sign in as david.hr
  → Cannot access Finance (Access Denied)
  → Can access HR (full control, including permissions)
```

---

## Network Flow Diagram

```
┌─────────────────────────────────────┐
│   Your Computer                      │
│   (Outside Azure)                    │
└───────────────┬──────────────────────┘
                │
                │ RDP Port 3389
                │ (from your public IP only)
                │
                ▼
        ┌───────────────┐
        │     NSG       │
        │  (Firewall)   │
        └───────┬───────┘
                │
                ├─────────────────────┬───────────────────┐
                │                     │                   │
                ▼                     ▼                   ▼
        ┌───────────────┐   ┌──────────────┐   ┌────────────────┐
        │     DC01      │   │     FS01     │   │   CLIENT01     │
        │  10.0.1.4     │   │  10.0.1.x    │   │   10.0.1.x     │
        │   (DNS)       │   │ (File Server)│   │ (Workstation)  │
        └───────┬───────┘   └──────┬───────┘   └────────┬───────┘
                │                  │                    │
                └──────────────────┬────────────────────┘
                                   │
                         Lab.local VNet
                         10.0.0.0/16
```

**Access paths:**

1. **RDP into CLIENT01:**
   - Your IP → NSG (allowed) → CLIENT01 (RDP port 3389)
   - Only RDP allowed; everything else blocked

2. **Domain authentication:**
   - CLIENT01 → DNS query → DC01 (10.0.1.4)
   - DC01 resolves lab.local names
   - CLIENT01 joins lab.local domain

3. **File access:**
   - CLIENT01 user → SMB share \\FS01\CompanyData
   - Share check: Everyone allowed (wide open)
   - NTFS check: User's groups determine actual access
   - Access grant/deny based on NTFS ACLs

4. **Lateral communication:**
   - DC01 ↔ FS01: Domain authentication, replication
   - DC01 ↔ CLIENT01: Logon, group policy
   - FS01 ↔ CLIENT01: SMB file access
   - All within VNet (fast, no public internet)

---

## Data Flow Diagram

```
Step 1: Terraform Deployment
┌─────────────────────────────────────┐
│  terraform apply                    │
│  (with $env:TF_VAR_admin_password)  │
└──────────────┬──────────────────────┘
               │
               ▼
       ┌───────────────┐
       │ Azure API     │
       │ Creates VMs   │
       │ Creates VNet  │
       │ Creates NSG   │
       │ Creates KV    │
       └───────┬───────┘
               │
               ▼
       ┌───────────────────────────┐
       │ Three VMs deployed        │
       │ Networking configured     │
       │ Key Vault stores password │
       └───────┬───────────────────┘

Step 2: PowerShell Configuration
┌─────────────────────────────────────┐
│  az vm run-command                  │
│  (passes script + password)         │
└──────────────┬──────────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  Azure VM Agent           │
       │  (runs inside VM)         │
       └───────┬───────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  PowerShell Script        │
       │  (00-promote-dc.ps1)      │
       └───────┬───────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  DC01 promoted to DC      │
       │  lab.local forest created │
       │  DNS service running      │
       └───────────────────────────┘

Step 3: Domain Joins
┌─────────────────────────────────────┐
│  script 01-join-domain.ps1          │
└──────────────┬──────────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  FS01 & CLIENT01          │
       │  Resolve DNS (10.0.1.4)   │
       │  Find lab.local domain    │
       └───────┬───────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  Join to lab.local        │
       │  Reboot automatically     │
       │  Sync with DC01           │
       └───────────────────────────┘

Step 4: Security Groups & Users
┌─────────────────────────────────────┐
│  scripts 02-04                      │
│  (OUs, groups, users)               │
└──────────────┬──────────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  Active Directory         │
       │  populated with:          │
       │  • OUs                    │
       │  • Security groups        │
       │  • Test users             │
       │  • Group memberships      │
       └───────────────────────────┘

Step 5: NTFS Permissions
┌─────────────────────────────────────┐
│  script 05-create-share-and-...     │
└──────────────┬──────────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  FS01                     │
       │  Create shares            │
       │  Apply NTFS ACLs          │
       │  Grant to security groups │
       └───────┬───────────────────┘

Step 6: RDP Access
┌─────────────────────────────────────┐
│  script 06-add-rdp-users.ps1        │
└──────────────┬──────────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  CLIENT01                 │
       │  Add users to local       │
       │  Remote Desktop Users     │
       │  group                    │
       └───────────────────────────┘

Step 7: Verification
┌─────────────────────────────────────┐
│  RDP into CLIENT01                  │
│  Sign in as test user               │
│  Access \\FS01\CompanyData          │
└──────────────┬──────────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  Test user's groups       │
       │  checked against NTFS ACL │
       └───────┬───────────────────┘
               │
               ▼
       ┌───────────────────────────┐
       │  Access granted or denied │
       │  based on permissions     │
       └───────────────────────────┘
```

---

## Dependency Chain

**What must happen in order:**

```
1. Terraform deployment
   ├─ Resource group created
   ├─ VNet created with DNS pointing at 10.0.1.4 (DC01)
   ├─ NSG created (RDP only from your IP)
   ├─ DC01 created with static NIC (10.0.1.4)
   ├─ FS01 & CLIENT01 created with depends_on: [DC01's NIC]
   │  └─ Why depends_on? Prevents race condition where dynamic NIC grabs 10.0.1.4
   └─ Key Vault created, password stored
        │
2. Script 00: Promote DC01
   │  DC01 installs AD DS
   │  DC01 creates lab.local forest
   │  DC01 starts DNS service
   │  DC01 reboots
   │  └─ Wait 5-10 minutes for DNS to be ready
        │
3. Script 01: Join FS01 & CLIENT01
   │  Machines look up DNS (10.0.1.4)
   │  Machines find lab.local domain
   │  Machines join domain
   │  Machines reboot
        │
4. Script 02: Create OUs
   │  DC01 creates: OU=FileServerLab, OU=Groups, OU=Users
        │
5. Script 03: Create Groups
   │  DC01 creates: GG-Finance-ReadOnly, GG-Finance-Modify, GG-HR-ReadOnly, GG-HR-FullControl
        │
6. Script 04: Create Users
   │  DC01 creates: alice.finance, brian.finance, carla.hr, david.hr
   │  Users added to groups
        │
7. Script 05: Create Share & NTFS Permissions
   │  FS01 creates: C:\CompanyData\Finance, C:\CompanyData\HR
   │  FS01 creates SMB share (Everyone Full Control)
   │  FS01 applies NTFS ACLs (groups only)
        │
8. Script 06: Add RDP Users
   │  CLIENT01 adds users to Remote Desktop Users group
        │
9. Verification
   └─ RDP into CLIENT01 as each test user
      └─ Verify access matches expectations
```

**Critical points:**
- DNS must be ready before domain joins
- Domain must exist before adding users
- Groups must exist before assigning permissions
- NTFS permissions must be set before testing access
- Each step is verified before moving to the next

---

## Security Model

```
┌──────────────────────────────────────────────────┐
│          NTFS File Server Security               │
│                                                  │
│  Layer 1: Network                                │
│  ├─ NSG allows RDP from your IP only            │
│  ├─ Everything else blocked by default          │
│  └─ VMs can't be accessed from internet         │
│                                                  │
│  Layer 2: Authentication                         │
│  ├─ All machines joined to lab.local domain     │
│  ├─ Users authenticate against DC01             │
│  └─ Passwords stored in Key Vault (encrypted)   │
│                                                  │
│  Layer 3: Authorization                         │
│  ├─ Share permissions: Everyone (wide open)     │
│  ├─ NTFS permissions: Security groups only      │
│  └─ Most restrictive wins                       │
│                                                  │
│  Layer 4: Access Control                         │
│  ├─ User signs in                               │
│  ├─ Token includes group memberships            │
│  ├─ Access to folder checked against NTFS ACL   │
│  └─ Grant/Deny based on group membership        │
│                                                  │
└──────────────────────────────────────────────────┘

Example: alice.finance accessing Finance folder

1. alice.finance RDPs to CLIENT01
   └─ DC01 authenticates password

2. alice.finance opens \\FS01\CompanyData\Finance
   └─ SMB check: Everyone allowed (pass)
   └─ NTFS check: Is alice in GG-Finance-ReadOnly? (yes)
   └─ Result: Access granted (read-only)

3. alice.finance tries to open \\FS01\CompanyData\HR
   └─ SMB check: Everyone allowed (pass)
   └─ NTFS check: Is alice in GG-HR-ReadOnly or GG-HR-FullControl? (no)
   └─ Result: Access denied
```

---

## Infrastructure as Code Benefits

**This design shows why Terraform + Terraform + PowerShell matters:**

```
Manual Clicking:                  Infrastructure as Code (This Lab):
├─ Create VNet → 1 hour          ├─ terraform apply → Automated
├─ Create VMs → 1 hour           ├─ Scripts run → Automated
├─ Install AD → 30 min            ├─ Full domain → Reproducible
├─ Create users → 20 min          ├─ Test access → Verified
└─ Hope it's right                └─ Redeploy in 2 hours identically

Error recovery:
├─ "Something broke"              ├─ terraform destroy → Clean up
├─ Manual troubleshooting         ├─ terraform apply → Redeploy
├─ Might take days                ├─ Scripts re-run → Full recovery
└─ Might not get it right         └─ 90 minutes later, same state
```

**Benefits:**
- Repeatable: Deploy the same thing 100 times identically
- Reviewable: Read the Terraform and PowerShell to understand design
- Versionable: Git history shows what changed and why
- Testable: Run test scripts to verify each step
- Scalable: Use same pattern for 3 VMs or 300

---

## Key Design Decisions

| Decision | What | Why |
|---|---|---|
| Static IP for DC01 | 10.0.1.4 never changes | DNS resolution depends on this address |
| DNS on VNet | All VMs point at DC01 | Without it, domain joins fail |
| depends_on on dynamic NICs | FS01 & CLIENT01 wait for DC01 NIC | Prevents race condition, DC01 keeps 10.0.1.4 |
| Share permissions wide open | Everyone Full Control | NTFS is the single gate, simpler to manage |
| NTFS to groups, not users | Permissions on GG-Finance-ReadOnly | When alice changes jobs, move to different group |
| Key Vault for password | No hardcoding anywhere | Secrets never in files, terminal, or Git history |
| az vm run-command | Run scripts through Azure agent | NSG can stay locked down, no open ports needed |
| Numbered scripts (00-06) | Clear execution order | Can't accidentally run them out of order |

---

## Scalability Considerations

**This lab is intentionally small, but scales these ways:**

```
For 10 users:
├─ Add to groups in script 04
└─ Permissions already set in script 05 (no changes)

For 100 users:
├─ Sync from AD (script 04 becomes import)
├─ Share already handles 100 users
└─ NTFS permissions unchanged (group-based)

For multiple file servers:
├─ Replicate FS01 setup to FS02, FS03...
├─ Each joins same lab.local domain
├─ Create group for each server: GG-FS01-Admins, GG-FS02-Admins...
└─ Permissions per server + per folder

For production:
├─ Use multiple DCs for redundancy
├─ Add Azure AD Connect for hybrid identity
├─ Add storage redundancy (geo-replication)
└─ Add monitoring (Azure Monitor)
```

---

## Summary

This architecture demonstrates:

✅ **Layered security:** Network → Authentication → Authorization → Access Control  
✅ **Repeatable infrastructure:** Terraform + PowerShell makes it reproducible  
✅ **Scalable permissions:** Groups, not users, survive organizational changes  
✅ **Clear dependencies:** Order matters, and that order is enforced  
✅ **Verification at every step:** Status messages don't count; verify results  
✅ **Single point of access control:** NTFS is where access is actually decided  

This is a production-grade pattern. The principles shown here apply to any file server, any cloud, any organization.

