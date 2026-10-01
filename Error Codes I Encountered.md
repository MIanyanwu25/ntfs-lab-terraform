# Common Errors and Fixes

**This page documents real errors from lab deployments.** Every entry below happened during actual builds.

---

## Terraform & Environment

### Error: "admin_password" must not be empty

**Symptom:**
```
│ Error: "admin_password" must not be empty
│
│   with azurerm_windows_virtual_machine.dc01,
│   on main.tf line 74, in resource "azurerm_windows_virtual_machine" "dc01":
│ 74:   admin_password                    = var.admin_password
```

**Cause:**
The `admin_password` environment variable is not set in your terminal session.

**Fix — First deployment:**
```powershell
$env:TF_VAR_admin_password = "YourSecure!Password123"  # 12+ chars, 3 types
terraform plan
```

**Fix — Returning session:**
```powershell
$env:TF_VAR_admin_password = az keyvault secret show --vault-name <your-vault-name> --name vm-admin-password --query value -o tsv
terraform plan
```

**Key point:** Environment variables die when the terminal closes. Every new terminal needs the password reloaded **before** any plan or apply.

---

### Error: HTTPSConnection — Failed to resolve Key Vault

**Symptom:**
```
ERROR: HTTPSConnection(host='kv-fslab-6fd52c4e.vault.azure.net', port=443): Failed to resolve 'kv-fslab-6fd52c4e.vault.azure.net' ([Errno 11001] getaddrinfo failed)
```

**Causes (in order of likelihood):**
1. Not logged into Azure CLI
2. Wrong Key Vault name (name doesn't match what was deployed)
3. Network/firewall blocking Azure DNS resolution

**Fix — Step 1: Confirm you're logged in**
```powershell
az account show
# If error, run:
az login
```

**Fix — Step 2: Find your real vault name**
```powershell
az keyvault list --query "[].name" -o tsv
# Compare output to the name in your command
```

**Fix — Step 3: Check DNS resolution**
```powershell
Resolve-DnsName vault.azure.net
# If timeout, you have a network/firewall issue
```

---

### Error: Plan wants to destroy all 3 VMs

**Symptom:**
```
Plan: 3 to add, 1 to change, 3 to destroy

~ admin_password = (sensitive value) # forces replacement
```

**Cause:**
The password in your current variable doesn't match what's stored in Terraform state. Since `admin_password` is ForceNew on VMs, Terraform's only way to "fix" the mismatch is to destroy and rebuild the VMs.

**Fix:**
**DO NOT APPLY.** First:
```powershell
$env:TF_VAR_admin_password = az keyvault secret show --vault-name <your-vault-name> --name vm-admin-password --query value -o tsv
terraform plan
# Should now say: No changes
```

The `# forces replacement` line is a red flag—always read plans carefully. One missed line can blow up your entire lab.

---

### Error: No Key Vault found

**Symptom:**
```powershell
az keyvault list --query "[].name" -o tsv
# (returns nothing)
```

**Cause:**
Either:
1. Terraform never deployed successfully
2. You're on the wrong Azure subscription

**Fix — Step 1: Check the subscription**
```powershell
az account show --query "{subscription:name, id:id}" -o table
# Wrong one? Switch it:
az account set --subscription "<name or id>"
```

**Fix — Step 2: Check if infrastructure exists**
```powershell
az group list --query "[].name" -o tsv
# Should show: RG-FileServerLab and rg-tfstate
```

If nothing appears, `terraform apply` from Step 5 never completed or failed silently. Re-run it.

---

## PowerShell Scripts & Azure VM Agent

### Error: "splatting operator" parse error in run-command

**Symptom:**
```
At C:\...\Downloads\script0.ps1:1 char:9
+ @01-join-domain.ps1
+         ~
You must provide a value expression following the '-join' operator...
```

**Cause:**
The `@filename.ps1` file wasn't found in your **local** current directory. Azure CLI doesn't report the missing file—it just sends the literal text `@01-join-domain.ps1` to the VM as if it were PowerShell code. PowerShell then tries to parse `@01-join-domain` as the splatting operator and fails.

**Fix:**
```powershell
# Confirm you're in the scripts folder
cd "$HOME\Projects\ntfs-lab-terraform\scripts"

# Check what's actually there
dir

# Look for the file name. Watch for:
# ❌ Double extensions: 01-join-domain.ps1.ps1
# ❌ Wrong case: 01-Join-Domain.ps1 (if you're case-sensitive)
# ❌ Typos: 01-join_domain.ps1

# If found with wrong name, rename it:
Rename-Item "01-join-domain.ps1.ps1" "01-join-domain.ps1"
```

Then re-run the command.

**Key lesson:** When you see "splatting operator" error, **run `dir` first.** The file almost always has a typo or double extension, especially if File Explorer was hiding extensions.

---

### Error: Cannot convert parameter to SecureString

**Symptom:**
```
Cannot process argument transformation on parameter 'SafeModePassword'. 
Cannot convert the "YourStrongPassword!" value of type "System.String" 
to type "System.Security.SecureString".
```

**Cause:**
The PowerShell parameter was typed `[SecureString]` instead of `[string]`. PowerShell deliberately refuses to auto-convert plain text to SecureString (for security reasons), even though it looks like a helpful suggestion.

**Fix:**
Change the parameter from:
```powershell
param([SecureString]$SafeModePassword)
```

to:
```powershell
param([string]$SafeModePassword)
```

Then immediately convert inside the script:
```powershell
$secureSafeModePassword = ConvertTo-SecureString $SafeModePassword -AsPlainText -Force
```

**Why:** `az vm run-command` passes parameters as plain strings over the wire. SecureString only exists inside PowerShell memory and can't be serialized. Always receive as string, convert manually inside the script.

---

### Error: "Provisioning succeeded" but domain not joined

**Symptom:**
```
"displayStatus": "Provisioning succeeded"

# But later, verification shows:
(Get-CimInstance Win32_ComputerSystem).Domain
# Returns: WORKGROUP  ← NOT lab.local
```

**Cause:**
Azure's run-command status only means the script *launched*, not that it *worked*. A script can launch successfully and still fail to do what you intended. DC01 might not have finished promotion, or a machine's DNS might not be pointing at DC01 yet.

**Fix:**
Always run the verification command after every script. Don't trust the status:
```powershell
# After 00-promote-dc.ps1:
az vm run-command invoke -g RG-FileServerLab -n DC01 --command-id RunPowerShellScript --scripts "Get-WindowsFeature AD-Domain-Services; (Get-CimInstance Win32_ComputerSystem).Domain; (Get-CimInstance Win32_ComputerSystem).PartOfDomain; Get-Service DNS"

# Expected: Installed, lab.local, True, Running
```

Verify before moving to the next step. This catches real failures early.

---

## Domain and DNS

### Error: Domain join fails with "domain does not exist"

**Symptom:**
```
Join-Computer : Unable to find a DC for the domain
```

**Cause:**
Either:
1. DC01 wasn't actually promoted (check Gotcha above)
2. FS01/CLIENT01 isn't using DC01 as DNS (10.0.1.4)
3. DNS is up but the domain hasn't propagated yet

**Fix — Step 1: Confirm DC01 promotion**
```powershell
az vm run-command invoke -g RG-FileServerLab -n DC01 --command-id RunPowerShellScript --scripts "(Get-CimInstance Win32_ComputerSystem).PartOfDomain; Get-Service DNS"
# Want: True, Running
```

**Fix — Step 2: Check DNS on the joining machine**
```powershell
az vm run-command invoke -g RG-FileServerLab -n FS01 --command-id RunPowerShellScript --scripts "Get-DnsClientServerAddress -AddressFamily IPv4 | Format-Table -AutoSize"
# Want: DNS server is 10.0.1.4
```

**Fix — Step 3: Test domain discovery**
```powershell
az vm run-command invoke -g RG-FileServerLab -n FS01 --command-id RunPowerShellScript --scripts "Resolve-DnsName -Name _ldap._tcp.dc._msdcs.lab.local -Type SRV"
# Want: SRV record pointing to dc01.lab.local at 10.0.1.4
```

If the SRV lookup times out, DC01's DNS isn't ready yet. Wait 5–10 minutes and retry.

---

### Error: SRV lookup times out

**Symptom:**
```powershell
Resolve-DnsName -Name _ldap._tcp.dc._msdcs.lab.local -Type SRV
# Hangs or times out
```

**Cause:**
DC01's DNS hasn't finished initializing, or DC01 promotion failed silently.

**Fix:**
```powershell
# Confirm DC01 is running
az vm list -d -g RG-FileServerLab --query "[].{name:name, powerState:powerState}" -o table

# Confirm promotion really worked
az vm run-command invoke -g RG-FileServerLab -n DC01 --command-id RunPowerShellScript --scripts "Get-Service DNS; (Get-CimInstance Win32_ComputerSystem).PartOfDomain"

# If both are good, just wait. DNS propagation takes time.
# Try again in 5 minutes:
Start-Sleep -Seconds 300
```

---

## Network & Access

### Error: RDP times out or won't connect

**Symptom:**
```
mstsc /v:40.123.45.67
# Hangs or "connection refused"
```

**Causes:**
1. VM is deallocated
2. Public IP changed but NSG rule still uses the old IP
3. VMs still booting

**Fix:**
```powershell
# Check VM state
az vm list -d -g RG-FileServerLab --query "[].{name:name, powerState:powerState, ip:publicIps}" -o table

# If deallocated, start it:
az vm start --resource-group RG-FileServerLab --name CLIENT01

# If IP changed, update the NSG rule in terraform/variables.tf:
# allowed_rdp_source_ip = "your-new-ip"
# Then:
terraform apply
```

---

### Error: OSProvisioningTimedOut on CLIENT01

**Symptom:**
```
OSProvisioningTimedOut: OS provisioning for VM CLIENT01 failed
```

**Cause:**
Windows 11's guest agent was slow to boot and report ready before Azure's timeout expired.

**Fix:**
```powershell
cd terraform
terraform apply
# It will successfully create CLIENT01 on the retry
```

This usually only needs one retry.

---

## Terraform File Issues

### Error: Operator mismatch in versions.tf

**Symptom:**
```
Error: Invalid version constraint: "~> 1.5.0"
```

**Cause:**
You used the wrong version operator on `required_version`. Remember:
- `required_version` (Terraform CLI) uses `>=` (stays backward compatible across 1.x)
- `required_providers` (providers) use `~>` (pessimistic, blocks breaking changes)

**Fix:**
In `versions.tf`:
```hcl
terraform {
  required_version = ">= 1.5.0"  # ← Greater-than-or-equal
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"       # ← Pessimistic (tile)
    }
  }
}
```

---

### Error: "Duplicate resource ... configuration"

**Symptom:**
```
Error: Duplicate resource "azurerm_network_interface" "fs01"
```

**Cause:**
A resource block was copy-pasted twice by accident.

**Fix:**
Look through your `.tf` files and delete the duplicate resource block.

---

### Error: PrivateIPAddressIsAllocated on DC01's NIC

**Symptom:**
```
Error creating Network Interface: ...PrivateIPAddressIsAllocated...
```

**Cause:**
Terraform tried to create the dynamic NICs (FS01, CLIENT01) before DC01's static NIC could reserve 10.0.1.4. The dynamic allocation grabbed it first.

**Fix (for future):**
The `depends_on` clauses in `main.tf` exist to prevent this. If you hit it anyway:

```powershell
# Delete the conflicting resources
terraform destroy -target azurerm_windows_virtual_machine.fs01
terraform destroy -target azurerm_network_interface.fs01

# Then apply again
terraform apply
```

---

### Error: Plan reports permanent drift on vm_agent_platform_updates_enabled

**Symptom:**
```
~ vm_agent_platform_updates_enabled = true → false
# ...on every plan, forever
```

**Cause:**
Azure enables this setting server-side; azurerm 3.x provider versions before 3.100 read it incorrectly. (Fixed in azurerm 4.26.0+.)

**Fix:**
Set the property explicitly on all three VMs in `main.tf`:
```hcl
vm_agent_platform_updates_enabled = true
```

This is already in the current `main.tf`. If you upgraded from an older version, make sure this line is on every `azurerm_windows_virtual_machine` resource.

---

## File System & Permissions

### Error: NTFS permissions not taking effect

**Symptom:**
```
icacls C:\CompanyData\Finance
# Shows inherited permissions still present
```

**Cause:**
The script didn't strip inherited permissions, or you forgot to grant SYSTEM and Domain Admins back after stripping.

**Fix:**
Re-run script 05 and verify:
```powershell
az vm run-command invoke -g RG-FileServerLab -n FS01 --command-id RunPowerShellScript --scripts "@05-create-share-and-permissions.ps1"

# Verify exactly 4 entries per folder:
az vm run-command invoke -g RG-FileServerLab -n FS01 --command-id RunPowerShellScript --scripts "icacls C:\CompanyData\Finance; icacls C:\CompanyData\HR"
```

Expected for Finance:
```
SYSTEM:(F)
LAB\Domain Admins:(F)
LAB\GG-Finance-ReadOnly:(RX)
LAB\GG-Finance-Modify:(M)
```

---

### Error: Access denied on share from test user

**Symptom:**
```
# Signed in as alice.finance
# Opening \\FS01\CompanyData works
# Opening Finance subfolder: Access Denied
```

**Causes (in order):**
1. User not in the correct group
2. NTFS permissions not set correctly
3. User signed in but group membership hasn't cached yet (sign out and back in)

**Fix:**
```powershell
# Verify user is in the right group:
az vm run-command invoke -g RG-FileServerLab -n DC01 --command-id RunPowerShellScript --scripts "Get-ADGroupMember -Identity 'GG-Finance-ReadOnly' | Select-Object Name"

# If alice.finance is missing, re-run script 04:
az vm run-command invoke -g RG-FileServerLab -n DC01 --command-id RunPowerShellScript --scripts "@04-create-users.ps1" --parameters "UserPassword=$labPwd"

# Then sign out and back in on CLIENT01 so group membership refreshes
```

---

## Azure CLI Issues

### Error: az login hangs after choosing account

**Symptom:**
```
# VS Code terminal
az login
# (hangs at browser login confirmation)
```

**Cause:**
Windows sign-in broker stalls when called from VS Code terminal.

**Fix:**
```powershell
az config set core.enable_broker_on_windows=false
az login
# Will open browser instead of trying broker
```

---

### Error: "az" command not found after installation

**Symptom:**
```powershell
az version
# The term 'az' is not recognized...
```

**Cause:**
The terminal was open *before* you installed Azure CLI. Your terminal's PATH hasn't been updated.

**Fix:**
Close the terminal completely and open a new one. The new terminal will see the updated PATH.

---

## Getting Help

**Checklist before debugging:**
1. ✅ Run `terraform plan` — does it show changes you expect?
2. ✅ Read the full error, not just the summary line
3. ✅ Check `docs/ERRORS.md` (this file) for the exact error
4. ✅ Run the relevant verification command (don't trust status messages)
5. ✅ Check your `.gitignore` — is sensitive data leaking?

**Common diagnostics:**
```powershell
# What infrastructure exists?
az group list --query "[].name" -o tsv
az resource list -g RG-FileServerLab --query "[].{name:name, type:type}" -o table

# Are the VMs running?
az vm list -d -g RG-FileServerLab --query "[].{name:name, power:powerState, ip:publicIps}" -o table

# Is Terraform state in sync?
terraform plan
# Should say: No changes (or show only what you intended to change)

# Is the domain healthy?
az vm run-command invoke -g RG-FileServerLab -n DC01 --command-id RunPowerShellScript --scripts "(Get-CimInstance Win32_ComputerSystem).PartOfDomain; Get-Service DNS"
```

---

## Still Stuck?

1. Capture the exact error message (copy-paste, not a screenshot)
2. Include the command you ran
3. Include output from one or two diagnostic commands above
4. Open an issue on GitHub with a clear title
5. Don't include passwords or sensitive IPs

---

**Last updated:** September 2026 — Verified Build Edition
