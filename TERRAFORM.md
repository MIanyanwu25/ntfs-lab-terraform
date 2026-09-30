# Terraform Configuration for NTFS File Server Lab

This file contains all Terraform configuration files needed to deploy the lab infrastructure in Azure. Each file is clearly partitioned and can be extracted as a standalone `.tf` file.

**Usage:** Place these files in the `terraform/` folder, then run `terraform init`, `terraform plan`, and `terraform apply`.

---

## 📋 Terraform Files Index

1. [versions.tf](#versionstf) — Terraform and provider versions
2. [backend.tf](#backendtf) — Remote state configuration
3. [variables.tf](#variablestf) — Input variables
4. [main.tf](#maintf) — VNet, NSG, VMs
5. [keyvault.tf](#keyvaulttf) — Key Vault and secret storage
6. [outputs.tf](#outputstf) — Exported values
7. [terraform.tfvars.example](#terraformtfvarsexample) — Example variables (safe to commit)

---

# versions.tf

**Purpose:** Define Terraform version requirements and provider versions  
**Keep:** This file should be committed to Git (`.terraform.lock.hcl` locks exact versions)

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

# Configure the Azure Provider
provider "azurerm" {
  features {}
  skip_provider_registration = false
}

provider "random" {}
provider "time" {}
```

**Why these versions:**
- **Terraform >= 1.5.0**: Supports dynamic blocks and other modern features
- **azurerm ~> 3.100**: Pessimistic constraint (allows 3.100-3.999, blocks 4.0+)
- **random & time**: Utility providers for generating values and delays

---

# backend.tf

**Purpose:** Configure remote state storage in Azure Storage Account  
**Important:** This file is not committed. Values are provided via `-backend-config` flags.

```hcl
terraform {
  backend "azurerm" {
    # These values are provided via terraform init -backend-config=...
    # NOT hardcoded here (prevents secrets in Git)
  }
}
```

**How to initialize with backend config:**

```powershell
terraform init `
  -backend-config="resource_group_name=rg-tfstate" `
  -backend-config="storage_account_name=tfstatelabs<YOUR-RANDOM>" `
  -backend-config="container_name=tfstate" `
  -backend-config="key=lab.tfstate"
```

**Why remote state:**
- Terraform state lives in Azure Storage (encrypted, backed up)
- Team members share the same state (no conflicts)
- Prevents accidental overwrites
- State contains sensitive data (passwords) — never commit it locally

---

# variables.tf

**Purpose:** Define input variables for the deployment  
**Note:** No defaults for secrets (passwords must be supplied at runtime)

```hcl
# General
variable "location" {
  description = "Azure region for all resources"
  type        = string
  default     = "centralus"
}

variable "resource_group_name" {
  description = "Name of the resource group"
  type        = string
  default     = "RG-FileServerLab"
}

variable "environment" {
  description = "Environment name (used in resource naming)"
  type        = string
  default     = "lab"
}

# Networking
variable "vnet_address_space" {
  description = "Address space for the VNet"
  type        = list(string)
  default     = ["10.0.0.0/16"]
}

variable "subnet_address_prefix" {
  description = "Address prefix for the subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "allowed_rdp_source_ip" {
  description = "Your public IP (for RDP access) — get it from: Invoke-RestMethod https://api.ipify.org"
  type        = string
}

# Domain Controller
variable "dc_vm_name" {
  description = "Name of the domain controller VM"
  type        = string
  default     = "DC01"
}

variable "dc_private_ip" {
  description = "Static private IP for DC01 (DNS depends on this)"
  type        = string
  default     = "10.0.1.4"
}

# File Server
variable "fs_vm_name" {
  description = "Name of the file server VM"
  type        = string
  default     = "FS01"
}

# Client
variable "client_vm_name" {
  description = "Name of the client VM"
  type        = string
  default     = "CLIENT01"
}

# Credentials
variable "admin_username" {
  description = "Local admin username for all VMs"
  type        = string
  default     = "labadmin"
}

variable "admin_password" {
  description = "Local admin password (set via $env:TF_VAR_admin_password)"
  type        = string
  sensitive   = true
  # No default — must be supplied at runtime
}

# Key Vault
variable "key_vault_name" {
  description = "Name of the Key Vault (must be globally unique)"
  type        = string
  # Example: "kv-fslab-<your-random-string>"
  # Get a unique suffix: -join (1..8 | ForEach-Object { [char](97 + (Get-Random -Maximum 26)) })
}

variable "key_vault_sku" {
  description = "SKU for Key Vault"
  type        = string
  default     = "standard"
}

# VM Configuration
variable "vm_size" {
  description = "Size of the VMs"
  type        = string
  default     = "Standard_B2s"  # 2 vCPU, 4 GB RAM (adequate for lab)
}

variable "os_disk_size_gb" {
  description = "OS disk size in GB"
  type        = number
  default     = 128
}

# Image
variable "publisher" {
  description = "Image publisher"
  type        = string
  default     = "MicrosoftWindowsServer"
}

variable "offer" {
  description = "Image offer"
  type        = string
  default     = "WindowsServer"
}

variable "sku" {
  description = "Image SKU"
  type        = string
  default     = "2022-datacenter"  # Windows Server 2022
}

variable "image_version" {
  description = "Image version"
  type        = string
  default     = "latest"
}

# Tags
variable "tags" {
  description = "Tags for all resources"
  type        = map(string)
  default = {
    Project     = "NTFS-FileServer-Lab"
    Environment = "Lab"
    CreatedBy   = "Terraform"
  }
}
```

**Critical variables (no defaults):**
- `allowed_rdp_source_ip` — Your public IP
- `admin_password` — Via environment variable
- `key_vault_name` — Must be globally unique

**How to provide values:**

```powershell
# Set environment variable for password
$env:TF_VAR_admin_password = "YourSecure!Password123"

# Create/edit terraform.tfvars (gitignored)
allowed_rdp_source_ip = "203.0.113.42"
key_vault_name        = "kv-fslab-abc123xyz"

# Then run:
terraform plan
terraform apply
```

---

# main.tf

**Purpose:** Define the core infrastructure (VNet, NSG, VMs)  
**Size:** ~200 lines

```hcl
# Get current context (for outputs)
data "azurerm_client_config" "current" {}

# Create resource group
resource "azurerm_resource_group" "lab" {
  name     = var.resource_group_name
  location = var.location
  tags     = var.tags
}

# ===== NETWORKING =====

# Create Virtual Network
resource "azurerm_virtual_network" "lab" {
  name                = "vnet-${var.environment}"
  address_space       = var.vnet_address_space
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name

  tags = var.tags
}

# Create Subnet
resource "azurerm_subnet" "lab" {
  name                 = "subnet-${var.environment}"
  resource_group_name  = azurerm_resource_group.lab.name
  virtual_network_name = azurerm_virtual_network.lab.name
  address_prefixes     = [var.subnet_address_prefix]
}

# Create Network Security Group
resource "azurerm_network_security_group" "lab" {
  name                = "nsg-${var.environment}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name

  # Allow RDP from your IP only
  security_rule {
    name                       = "AllowRDP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "3389"
    source_address_prefix      = var.allowed_rdp_source_ip
    destination_address_prefix = "*"
  }

  # Deny everything else
  security_rule {
    name                       = "DenyAllInbound"
    priority                   = 200
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = var.tags
}

# ===== DC01 (DOMAIN CONTROLLER) =====

# Network Interface for DC01 (STATIC IP)
resource "azurerm_network_interface" "dc01" {
  name                = "nic-${var.dc_vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name

  ip_configuration {
    name                          = "ipconfig1"
    subnet_id                     = azurerm_subnet.lab.id
    private_ip_address_allocation = "Static"
    private_ip_address            = var.dc_private_ip  # MUST NOT CHANGE
    public_ip_address_id          = azurerm_public_ip.dc01.id
  }

  tags = var.tags
}

# Public IP for DC01
resource "azurerm_public_ip" "dc01" {
  name                = "pip-${var.dc_vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  allocation_method   = "Static"

  tags = var.tags
}

# Associate NSG with DC01's NIC
resource "azurerm_network_interface_security_group_association" "dc01" {
  network_interface_id      = azurerm_network_interface.dc01.id
  network_security_group_id = azurerm_network_security_group.lab.id
}

# DC01 Virtual Machine
resource "azurerm_windows_virtual_machine" "dc01" {
  name                = var.dc_vm_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  admin_username      = var.admin_username
  admin_password      = var.admin_password
  size                = var.vm_size

  network_interface_ids = [azurerm_network_interface.dc01.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
    disk_size_gb         = var.os_disk_size_gb
  }

  source_image_reference {
    publisher = var.publisher
    offer     = var.offer
    sku       = var.sku
    version   = var.image_version
  }

  tags = var.tags
}

# ===== FS01 (FILE SERVER) =====

# Network Interface for FS01 (DYNAMIC IP)
resource "azurerm_network_interface" "fs01" {
  name                = "nic-${var.fs_vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name

  ip_configuration {
    name                          = "ipconfig1"
    subnet_id                     = azurerm_subnet.lab.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.fs01.id
  }

  # Depends on DC01's NIC to prevent race condition
  depends_on = [azurerm_network_interface.dc01]

  tags = var.tags
}

# Public IP for FS01
resource "azurerm_public_ip" "fs01" {
  name                = "pip-${var.fs_vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  allocation_method   = "Static"

  tags = var.tags
}

# Associate NSG with FS01's NIC
resource "azurerm_network_interface_security_group_association" "fs01" {
  network_interface_id      = azurerm_network_interface.fs01.id
  network_security_group_id = azurerm_network_security_group.lab.id
}

# FS01 Virtual Machine
resource "azurerm_windows_virtual_machine" "fs01" {
  name                = var.fs_vm_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  admin_username      = var.admin_username
  admin_password      = var.admin_password
  size                = var.vm_size

  network_interface_ids = [azurerm_network_interface.fs01.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
    disk_size_gb         = var.os_disk_size_gb
  }

  source_image_reference {
    publisher = var.publisher
    offer     = var.offer
    sku       = var.sku
    version   = var.image_version
  }

  tags = var.tags
}

# ===== CLIENT01 (WORKSTATION) =====

# Network Interface for CLIENT01 (DYNAMIC IP)
resource "azurerm_network_interface" "client01" {
  name                = "nic-${var.client_vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name

  ip_configuration {
    name                          = "ipconfig1"
    subnet_id                     = azurerm_subnet.lab.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.client01.id
  }

  # Depends on DC01's NIC to prevent race condition
  depends_on = [azurerm_network_interface.dc01]

  tags = var.tags
}

# Public IP for CLIENT01
resource "azurerm_public_ip" "client01" {
  name                = "pip-${var.client_vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  allocation_method   = "Static"

  tags = var.tags
}

# Associate NSG with CLIENT01's NIC
resource "azurerm_network_interface_security_group_association" "client01" {
  network_interface_id      = azurerm_network_interface.client01.id
  network_security_group_id = azurerm_network_security_group.lab.id
}

# CLIENT01 Virtual Machine (Windows 11 would use "win11-" sku)
resource "azurerm_windows_virtual_machine" "client01" {
  name                = var.client_vm_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  admin_username      = var.admin_username
  admin_password      = var.admin_password
  size                = var.vm_size

  network_interface_ids = [azurerm_network_interface.client01.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
    disk_size_gb         = var.os_disk_size_gb
  }

  source_image_reference {
    publisher = var.publisher
    offer     = var.offer
    sku       = var.sku
    version   = var.image_version
  }

  tags = var.tags
}
```

**Key design decisions:**

1. **DC01 static IP (10.0.1.4)** — DNS depends on this never changing
2. **depends_on clauses** — Prevent race condition where dynamic NICs grab DC01's IP
3. **NSG RDP rule** — Allow from your IP only, deny everything else
4. **Public IPs** — Needed for RDP access; could be removed post-deployment
5. **All in one subnet** — Simplifies this lab; production would use multiple subnets

---

# keyvault.tf

**Purpose:** Create Azure Key Vault and store the admin password securely

```hcl
# Generate a random suffix for Key Vault name (must be globally unique)
resource "random_string" "keyvault_suffix" {
  length  = 8
  special = false
  upper   = false
}

# Create the Key Vault
resource "azurerm_key_vault" "lab" {
  name                = var.key_vault_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = var.key_vault_sku

  # Allow current user to access the vault
  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = data.azurerm_client_config.current.object_id

    secret_permissions = [
      "Get",
      "List",
      "Set",
      "Delete",
      "Recover",
      "Backup",
      "Restore"
    ]
  }

  soft_delete_retention_days = 7
  purge_protection_enabled   = false

  tags = var.tags
}

# Store the admin password in Key Vault
resource "azurerm_key_vault_secret" "admin_password" {
  name            = "vm-admin-password"
  value           = var.admin_password
  key_vault_id    = azurerm_key_vault.lab.id
  content_type    = "password"
  expiration_date = null  # Never expires (lab setting)

  tags = var.tags
}

# Wait 30 seconds after Key Vault creation (Azure propagation delay)
resource "time_sleep" "wait_for_keyvault" {
  depends_on = [azurerm_key_vault.lab]

  create_duration = "30s"
}
```

**Why Key Vault:**
- Password encrypted at rest
- Access controlled via Azure RBAC
- Audit trail of who accessed it
- Can be used by multiple labs/deployments
- Secrets never in files or Git history

**After deployment, retrieve password:**

```powershell
az keyvault secret show `
  --vault-name kv-fslab-abc123xyz `
  --name vm-admin-password `
  --query value -o tsv
```

---

# outputs.tf

**Purpose:** Export values that users need after deployment

```hcl
output "resource_group_name" {
  value       = azurerm_resource_group.lab.name
  description = "Name of the resource group"
}

output "vnet_id" {
  value       = azurerm_virtual_network.lab.id
  description = "ID of the virtual network"
}

output "dc01_private_ip" {
  value       = azurerm_network_interface.dc01.private_ip_address
  description = "Private IP of DC01 (DNS server)"
}

output "dc01_public_ip" {
  value       = azurerm_public_ip.dc01.ip_address
  description = "Public IP of DC01 (for RDP)"
}

output "dc01_vm_id" {
  value       = azurerm_windows_virtual_machine.dc01.id
  description = "Azure resource ID of DC01"
}

output "fs01_private_ip" {
  value       = azurerm_network_interface.fs01.private_ip_address
  description = "Private IP of FS01 (file server)"
}

output "fs01_public_ip" {
  value       = azurerm_public_ip.fs01.ip_address
  description = "Public IP of FS01 (for RDP)"
}

output "fs01_vm_id" {
  value       = azurerm_windows_virtual_machine.fs01.id
  description = "Azure resource ID of FS01"
}

output "client01_private_ip" {
  value       = azurerm_network_interface.client01.private_ip_address
  description = "Private IP of CLIENT01"
}

output "client01_public_ip" {
  value       = azurerm_public_ip.client01.ip_address
  description = "Public IP of CLIENT01 (for RDP)"
}

output "client01_vm_id" {
  value       = azurerm_windows_virtual_machine.client01.id
  description = "Azure resource ID of CLIENT01"
}

output "key_vault_id" {
  value       = azurerm_key_vault.lab.id
  description = "Azure resource ID of Key Vault"
}

output "key_vault_name" {
  value       = azurerm_key_vault.lab.name
  description = "Name of the Key Vault (use with: az keyvault secret show --vault-name)"
}

output "nsg_id" {
  value       = azurerm_network_security_group.lab.id
  description = "Azure resource ID of the NSG"
}

output "deployment_complete" {
  value       = "Infrastructure deployed. Next: Run scripts 00-06 in order."
  description = "Deployment status"
}
```

**After terraform apply, you'll see:**

```
Outputs:

dc01_private_ip = "10.0.1.4"
dc01_public_ip = "40.123.45.67"
fs01_public_ip = "40.123.45.68"
client01_public_ip = "40.123.45.69"
key_vault_name = "kv-fslab-abc123xyz"
deployment_complete = "Infrastructure deployed. Next: Run scripts 00-06 in order."
```

---

# terraform.tfvars.example

**Purpose:** Safe template for variable values (COMMIT THIS, not terraform.tfvars)  
**Note:** This is an EXAMPLE. Copy to terraform.tfvars and fill in your values.

```hcl
# Networking
allowed_rdp_source_ip = "203.0.113.42"  # YOUR public IP (get from: Invoke-RestMethod https://api.ipify.org)

# Key Vault (must be globally unique)
key_vault_name = "kv-fslab-abc123xyz"  # Change "abc123xyz" to something unique

# Optional overrides (defaults usually work)
location              = "centralus"      # Change if you prefer different region
resource_group_name   = "RG-FileServerLab"
vm_size               = "Standard_B2s"   # 2 vCPU, 4GB RAM (adequate for lab)
```

**Setup:**

```powershell
# Copy example to real file
Copy-Item terraform.tfvars.example terraform.tfvars

# Edit terraform.tfvars with your values
code terraform.tfvars

# Set password as environment variable
$env:TF_VAR_admin_password = "YourSecure!Password123"

# Deploy
terraform init
terraform plan
terraform apply
```

---

## Deployment Workflow

```powershell
# 1. Set up folder structure
mkdir ntfs-lab-terraform\terraform
cd ntfs-lab-terraform\terraform

# 2. Extract all .tf files from this markdown
# (Save each section into its own .tf file)

# 3. Set up backend state
terraform init `
  -backend-config="resource_group_name=rg-tfstate" `
  -backend-config="storage_account_name=tfstate<your-random>" `
  -backend-config="container_name=tfstate" `
  -backend-config="key=lab.tfstate"

# 4. Create terraform.tfvars
Copy-Item terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values

# 5. Set password
$env:TF_VAR_admin_password = "YourSecure!Password123"

# 6. Preview deployment
terraform plan

# 7. Deploy
terraform apply

# 8. Run PowerShell scripts (see SCRIPTS.md)
# 9. Test access
# 10. Cleanup (when done)
# terraform destroy
```

---

## File Organization Quick Reference

```
terraform/
├── versions.tf                    ✅ Commit this
├── backend.tf                     ✅ Commit this
├── variables.tf                   ✅ Commit this
├── main.tf                        ✅ Commit this
├── keyvault.tf                    ✅ Commit this
├── outputs.tf                     ✅ Commit this
├── terraform.tfvars.example       ✅ Commit this (safe template)
├── terraform.tfvars               ❌ DO NOT commit (.gitignored)
├── .terraform/                    ❌ DO NOT commit (provider binaries)
└── .terraform.lock.hcl            ✅ Commit this (version lock)
```

---

## Why This Structure

| File | Purpose | Commit? |
|---|---|---|
| versions.tf | Define versions | ✅ Yes |
| backend.tf | State location | ✅ Yes |
| variables.tf | Input parameters | ✅ Yes |
| main.tf | Infrastructure | ✅ Yes |
| keyvault.tf | Secrets | ✅ Yes |
| outputs.tf | Export values | ✅ Yes |
| terraform.tfvars.example | Safe template | ✅ Yes |
| terraform.tfvars | Your values (passwords!) | ❌ NO |
| .terraform/ | Downloaded binaries | ❌ NO |
| .terraform.lock.hcl | Version lock | ✅ Yes |

**Never commit terraform.tfvars** — it contains your admin password.

---

## Troubleshooting

### Error: "admin_password" must not be empty
```powershell
$env:TF_VAR_admin_password = "YourPassword"
terraform plan
```

### Error: Key Vault name not unique
```powershell
# Change key_vault_name in terraform.tfvars
# Add random suffix: -join (1..8 | ForEach-Object { [char](97 + (Get-Random -Maximum 26)) })
```

### Plan wants to destroy VMs
```powershell
# Password doesn't match state
$env:TF_VAR_admin_password = az keyvault secret show --vault-name <your-vault> --name vm-admin-password --query value -o tsv
terraform plan
# Should now show: No changes
```

