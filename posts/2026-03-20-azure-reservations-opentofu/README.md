# How to Manage Azure Reservations with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Reservations, Cost Optimization, Infrastructure as Code

Description: Learn how to purchase and manage Azure Reservations with OpenTofu to reduce costs on VMs, SQL databases, and other Azure services with 1 or 3-year commitments.

Azure Reservations offer up to 72% savings on Azure services compared to pay-as-you-go pricing. Managing reservations in OpenTofu keeps financial commitments documented and reviewable alongside the infrastructure they support.

## Provider Configuration

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    azapi = {
      source  = "Azure/azapi"
      version = "~> 1.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

## VM Reservation

```hcl
# Azure reservations are managed via the azapi provider or Azure CLI
# The azurerm provider does not yet support purchasing reservations directly

resource "azapi_resource" "vm_reservation" {
  type      = "Microsoft.Capacity/reservationOrders@2022-11-01"
  name      = "production-vm-reservation"
  location  = "global"
  parent_id = "/subscriptions/${var.subscription_id}"

  body = jsonencode({
    sku = {
      name = "Standard_D4s_v5"
    }
    properties = {
      reservedResourceType = "VirtualMachines"
      term                 = "P1Y"  # P1Y = 1 year, P3Y = 3 years
      billingPlan          = "Monthly"  # Monthly or Upfront
      quantity             = 5  # Number of instances
      appliedScopes        = ["Shared"]  # Shared or specific subscription/resource group
      appliedScopeType     = "Shared"
      location             = "eastus"

      reservedResourceProperties = {
        instanceFlexibility = "On"  # Instance size flexibility
      }
    }
  })
}
```

## SQL Database Reservation

```hcl
resource "azapi_resource" "sql_reservation" {
  type      = "Microsoft.Capacity/reservationOrders@2022-11-01"
  name      = "production-sql-reservation"
  location  = "global"
  parent_id = "/subscriptions/${var.subscription_id}"

  body = jsonencode({
    sku = {
      name = "GP_Gen5"
    }
    properties = {
      reservedResourceType = "SqlDatabases"
      term                 = "P3Y"  # 3 year for maximum savings
      billingPlan          = "Monthly"
      quantity             = 2
      appliedScopes        = ["Shared"]
      appliedScopeType     = "Shared"
      location             = "eastus"

      reservedResourceProperties = {
        productType = "SQLAzureGPGen5"
        vcoresCount = 8
      }
    }
  })
}
```

## Using Azure CLI for Reservation Purchase

```bash
# Get available reservation offers
az reservations catalog show \
  --subscription-id <sub-id> \
  --reserved-resource-type VirtualMachines \
  --location eastus

# Purchase a reservation
az reservations reservation-order purchase \
  --reservation-order-name "prod-vm-reservation" \
  --reserved-resource-type VirtualMachines \
  --sku "Standard_D4s_v5" \
  --applied-scope-type Shared \
  --billing-scope "/subscriptions/<sub-id>" \
  --quantity 5 \
  --term P1Y \
  --billing-plan Monthly \
  --location eastus
```

## Cost Budget for Reservation Coverage

```hcl
resource "azurerm_consumption_budget_subscription" "vm_budget" {
  name            = "vm-cost-budget"
  subscription_id = var.subscription_id

  amount     = 10000
  time_grain = "Monthly"

  time_period {
    start_date = "2024-01-01T00:00:00Z"
  }

  filter {
    resource_type {
      operator = "In"
      values   = ["Microsoft.Compute/virtualMachines"]
    }
  }

  notification {
    enabled        = true
    threshold      = 90
    operator       = "GreaterThan"
    threshold_type = "Actual"

    contact_emails = ["finops@example.com"]
  }
}
```

## Reserved Instance Exchange and Return

Azure allows exchanging reservations for different sizes within the same family (if Instance Size Flexibility is enabled). Manage this through the Azure portal or CLI, not OpenTofu, as it involves stateful financial transactions.

## Conclusion

Azure Reservations in OpenTofu provide documented, reviewable financial commitments. Use the azapi provider for reservation purchases until native azurerm support is available, enable instance size flexibility to automatically apply savings to different VM sizes within the same family, and monitor reservation utilization in the Azure Cost Management portal. Share reservations at the subscription or billing account level for maximum utilization.
