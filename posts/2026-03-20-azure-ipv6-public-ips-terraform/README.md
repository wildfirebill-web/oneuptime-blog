# How to Configure Azure IPv6 Public IPs with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Terraform, IPv6, Public IP, Networking, Infrastructure as Code

Description: A guide to creating and managing Azure Standard IPv6 Public IP addresses with Terraform for use with VMs, load balancers, and firewalls.

Azure Public IP addresses can be assigned either IPv4 or IPv6 addresses. IPv6 Public IPs are Standard SKU only and are assigned dynamically from Microsoft's public IPv6 ranges. They are used to expose VMs and load balancers to the internet over IPv6.

## Prerequisites

- Azure subscription with sufficient quota
- Terraform >= 1.3 with `hashicorp/azurerm` provider ~> 3.0
- A resource group

## Step 1: Create IPv6 Public IP Addresses

```hcl
# public-ips.tf - Azure Standard IPv6 Public IP addresses

locals {
  location = "East US"
  rg_name  = "rg-ipv6-demo"
}

# Standalone IPv6 public IP for a single resource
resource "azurerm_public_ip" "ipv6_primary" {
  name                = "pip-ipv6-primary"
  location            = local.location
  resource_group_name = local.rg_name

  # Must be Standard for IPv6 (Basic SKU does not support IPv6 standalone)
  sku = "Standard"

  # Specify IPv6 address version
  ip_version = "IPv6"

  # IPv6 public IPs must use Static allocation with Standard SKU
  allocation_method = "Static"

  # Optional: specify a DNS label for a AAAA record
  domain_name_label = "myapp-ipv6"

  tags = {
    Environment = "production"
  }
}

# Output the assigned IPv6 address (assigned after apply)
output "ipv6_address" {
  value = azurerm_public_ip.ipv6_primary.ip_address
}

output "ipv6_fqdn" {
  value = azurerm_public_ip.ipv6_primary.fqdn
}
```

## Step 2: Create Multiple IPv6 Public IPs with for_each

```hcl
# multiple-pips.tf - Create IPv6 public IPs for multiple regions or services
variable "services" {
  type    = list(string)
  default = ["web", "api", "gateway"]
}

resource "azurerm_public_ip" "ipv6_services" {
  for_each = toset(var.services)

  name                = "pip-ipv6-${each.key}"
  location            = local.location
  resource_group_name = local.rg_name
  sku                 = "Standard"
  ip_version          = "IPv6"
  allocation_method   = "Static"
  domain_name_label   = "myapp-${each.key}-ipv6"

  tags = {
    Service = each.key
  }
}

output "service_ipv6_addresses" {
  value = {
    for name, pip in azurerm_public_ip.ipv6_services :
    name => pip.ip_address
  }
}
```

## Step 3: Attach IPv6 Public IP to a VM NIC

```hcl
# nic.tf - Attach IPv6 public IP to a network interface
resource "azurerm_network_interface" "dual_stack" {
  name                = "nic-dual-stack"
  location            = local.location
  resource_group_name = local.rg_name

  # Primary IPv4 configuration
  ip_configuration {
    name                          = "ipv4"
    subnet_id                     = azurerm_subnet.main.id
    private_ip_address_allocation = "Dynamic"
    private_ip_address_version    = "IPv4"
    public_ip_address_id          = azurerm_public_ip.ipv4.id
    primary                       = true
  }

  # Secondary IPv6 configuration
  ip_configuration {
    name                          = "ipv6"
    subnet_id                     = azurerm_subnet.main.id
    private_ip_address_allocation = "Dynamic"
    private_ip_address_version    = "IPv6"
    # Attach the IPv6 public IP
    public_ip_address_id          = azurerm_public_ip.ipv6_primary.id
  }
}
```

## Step 4: Apply and Verify

```bash
terraform apply

# Check the assigned IPv6 address
az network public-ip show \
  --resource-group rg-ipv6-demo \
  --name pip-ipv6-primary \
  --query ipAddress

# Verify DNS resolution of the FQDN
dig AAAA myapp-ipv6.eastus.cloudapp.azure.com
```

## Important Constraints

- IPv6 public IPs require **Standard SKU**
- IPv6 public IPs require **Static** allocation
- A NIC can have at most one IPv6 public IP per IP configuration
- Azure Firewall and some other services require dual Public IPs (one IPv4 + one IPv6)

Azure IPv6 Public IPs are foundational for any Azure resource that needs to be directly reachable over IPv6 from the internet, including load balancers, application gateways, and directly exposed VMs.
