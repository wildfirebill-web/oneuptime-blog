# How to Deploy Portainer on Azure VM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, Virtual Machine, Docker, Self-Hosted, Infrastructure

Description: Learn how to provision an Azure VM and deploy Portainer CE to manage Docker containers on Azure infrastructure.

---

Portainer can be deployed on Azure Virtual Machines as a Docker container. This guide uses OpenTofu to provision the VM and cloud-init to install Docker and Portainer automatically.

---

## Create the Azure VM with OpenTofu

```hcl
resource "azurerm_resource_group" "portainer" {
  name     = "portainer-rg"
  location = "eastus"
}

resource "azurerm_linux_virtual_machine" "portainer" {
  name                = "portainer-vm"
  resource_group_name = azurerm_resource_group.portainer.name
  location            = azurerm_resource_group.portainer.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"

  network_interface_ids = [azurerm_network_interface.portainer.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  custom_data = base64encode(<<-EOF
    #cloud-config
    packages:
      - apt-transport-https
      - ca-certificates
      - curl

    runcmd:
      - curl -fsSL https://get.docker.com | sh
      - docker volume create portainer_data
      - docker run -d --name portainer --restart=always
          -p 9443:9443
          -v /var/run/docker.sock:/var/run/docker.sock
          -v portainer_data:/data
          portainer/portainer-ce:latest
  EOF
  )
}
```

---

## Network Security Group

```hcl
resource "azurerm_network_security_group" "portainer" {
  name                = "portainer-nsg"
  location            = azurerm_resource_group.portainer.location
  resource_group_name = azurerm_resource_group.portainer.name

  security_rule {
    name                       = "allow-portainer"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = var.admin_ip
    destination_address_prefix = "*"
    source_port_range          = "*"
    destination_port_range     = "9443"
  }
}
```

---

## Output the Public IP

```hcl
output "portainer_url" {
  value = "https://${azurerm_public_ip.portainer.ip_address}:9443"
}
```

---

## Summary

Use `custom_data` with a cloud-config script to install Docker and launch Portainer on first boot. Restrict the NSG to allow port 9443 only from your admin IP. Apply with `tofu apply` and access Portainer at the output URL within a few minutes after the VM boots.
