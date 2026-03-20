# How to Create Network Security Groups with OpenTofu on Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, NSG, Network Security, Infrastructure as Code

Description: Learn how to create Azure Network Security Groups with OpenTofu to control inbound and outbound traffic for subnets and network interfaces.

## Introduction

Azure Network Security Groups (NSGs) filter network traffic with inbound and outbound security rules. They can be associated with subnets or individual network interfaces. OpenTofu manages NSGs with both inline and separate security rules.

## NSG with Inline Rules

```hcl
resource "azurerm_network_security_group" "web" {
  name                = "nsg-web-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name

  security_rule {
    name                       = "allow-https"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "Internet"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "allow-http"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "Internet"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "deny-all-inbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = { Environment = var.environment }
}
```

## Separate Security Rule Resources

For modular management, use separate `azurerm_network_security_rule` resources:

```hcl
resource "azurerm_network_security_group" "app" {
  name                = "nsg-app-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
}

resource "azurerm_network_security_rule" "app_from_alb" {
  name                        = "allow-from-load-balancer"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "8080"
  source_address_prefix       = "AzureLoadBalancer"  # Azure service tag
  destination_address_prefix  = "*"
  resource_group_name         = var.resource_group_name
  network_security_group_name = azurerm_network_security_group.app.name
}
```

## NSG Association with Subnet

```hcl
resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.web.id
  network_security_group_id = azurerm_network_security_group.web.id
}
```

## Dynamic Rules from Variable

```hcl
variable "inbound_rules" {
  type = list(object({
    name           = string
    priority       = number
    port           = string
    source_prefix  = string
  }))
}

resource "azurerm_network_security_rule" "dynamic" {
  for_each = { for rule in var.inbound_rules : rule.name => rule }

  name                        = each.value.name
  priority                    = each.value.priority
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = each.value.port
  source_address_prefix       = each.value.source_prefix
  destination_address_prefix  = "*"
  resource_group_name         = var.resource_group_name
  network_security_group_name = azurerm_network_security_group.app.name
}
```

## Conclusion

NSGs in Azure are stateful firewalls—allow one direction and the return traffic is automatically permitted. Use Azure Service Tags (like `AzureLoadBalancer`, `Internet`, `VirtualNetwork`) instead of CIDRs where possible, as they automatically update when Azure IP ranges change.
