# How to Configure Azure VM Extensions with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, VM Extension, Custom Scripts, Diagnostic, Monitoring, Infrastructure as Code

Description: Learn how to configure Azure VM extensions with OpenTofu to automate post-deployment configuration, monitoring, and management tasks on Linux and Windows VMs.

## Introduction

Azure VM extensions are small applications that run on VMs to provide post-deployment configuration, automation, and management. Common extensions include Custom Script Extension (run scripts), Azure Monitor Agent (collect metrics and logs), Microsoft Antimalware, AAD Login (use Azure AD credentials), and Disk Encryption. Extensions run as privileged processes and are managed by the Azure VM Agent that runs on every Azure VM.

## Prerequisites

- OpenTofu v1.6+
- An existing Azure Linux or Windows VM
- Azure credentials with VM Contributor permissions

## Step 1: Custom Script Extension (Linux)

```hcl
resource "azurerm_virtual_machine_extension" "setup_script" {
  name                 = "setup-script"
  virtual_machine_id   = azurerm_linux_virtual_machine.main.id
  publisher            = "Microsoft.Azure.Extensions"
  type                 = "CustomScript"
  type_handler_version = "2.1"

  settings = jsonencode({
    script = base64encode(<<-EOT
      #!/bin/bash
      apt-get update -y
      apt-get install -y nginx
      systemctl enable nginx
      systemctl start nginx
      echo "Configured by VM Extension" > /var/www/html/index.html
    EOT
    )
  })

  tags = {
    Name = "${var.project_name}-setup"
  }
}
```

## Step 2: Azure Monitor Agent

```hcl
resource "azurerm_virtual_machine_extension" "ama" {
  name                       = "AzureMonitorLinuxAgent"
  virtual_machine_id         = azurerm_linux_virtual_machine.main.id
  publisher                  = "Microsoft.Azure.Monitor"
  type                       = "AzureMonitorLinuxAgent"
  type_handler_version       = "1.0"
  automatic_upgrade_enabled  = true
  auto_upgrade_minor_version = true

  settings = jsonencode({
    authentication = {
      managedIdentity = {
        identifier-name  = "mi_res_id"
        identifier-value = azurerm_linux_virtual_machine.main.id
      }
    }
  })
}
```

## Step 3: AAD SSH Login Extension

```hcl
# Allow Azure AD users to SSH into Linux VMs

resource "azurerm_virtual_machine_extension" "aad_login" {
  name                 = "AADSSHLoginForLinux"
  virtual_machine_id   = azurerm_linux_virtual_machine.main.id
  publisher            = "Microsoft.Azure.ActiveDirectory"
  type                 = "AADSSHLoginForLinux"
  type_handler_version = "1.0"

  depends_on = [azurerm_linux_virtual_machine.main]
}

# Grant VM Login role to a user
resource "azurerm_role_assignment" "vm_login" {
  scope                = azurerm_linux_virtual_machine.main.id
  role_definition_name = "Virtual Machine User Login"
  principal_id         = var.user_principal_id
}
```

## Step 4: Disk Encryption Extension

```hcl
resource "azurerm_virtual_machine_extension" "disk_encryption" {
  name                 = "AzureDiskEncryption"
  virtual_machine_id   = azurerm_linux_virtual_machine.main.id
  publisher            = "Microsoft.Azure.Security"
  type                 = "AzureDiskEncryption"
  type_handler_version = "2.2"

  settings = jsonencode({
    EncryptionOperation = "EnableEncryption"
    KeyVaultURL         = var.key_vault_uri
    KeyVaultResourceId  = var.key_vault_id
    VolumeType          = "All"  # OS, Data, or All
  })
}
```

## Step 5: Windows Antimalware Extension

```hcl
resource "azurerm_virtual_machine_extension" "antimalware" {
  name                 = "IaaSAntimalware"
  virtual_machine_id   = azurerm_windows_virtual_machine.main.id
  publisher            = "Microsoft.Azure.Security"
  type                 = "IaaSAntimalware"
  type_handler_version = "1.3"

  settings = jsonencode({
    AntimalwareEnabled = true
    RealtimeProtectionEnabled = true
    ScheduledScanSettings = {
      isEnabled = true
      day       = "1"     # Sunday
      time      = "120"   # Minutes from midnight
      scanType  = "Quick"
    }
    Exclusions = {
      Extensions = ".log;.bak"
      Paths      = "D:\\Data"
      Processes  = "application.exe"
    }
  })
}
```

## Step 6: Deploy

```bash
tofu init
tofu plan
tofu apply

# Check extension status
az vm extension show \
  --resource-group <rg> \
  --vm-name <vm-name> \
  --name CustomScript

# View extension logs (Linux)
# /var/log/azure/custom-script/handler.log
```

## Conclusion

Order extension deployment carefully using `depends_on` when extensions have dependencies. The Azure Monitor Agent extension requires a managed identity on the VM (`identity { type = "SystemAssigned" }`) and a Data Collection Rule association to start collecting metrics and logs. Use `automatic_upgrade_enabled = true` on monitoring extensions to stay current with agent updates. For Custom Script Extension, prefer referencing scripts from Azure Blob Storage over base64-encoded inline scripts to keep configurations readable and maintainable.
