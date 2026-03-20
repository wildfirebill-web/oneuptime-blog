# How to Create AWS Accounts with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS Organizations, Account Management, Multi-Account, Infrastructure as Code

Description: Learn how to create and configure new AWS accounts within AWS Organizations using OpenTofu for automated account provisioning.

Creating AWS accounts programmatically through AWS Organizations enables consistent, repeatable account provisioning. OpenTofu lets you define account properties, OU placement, and initial IAM roles as code.

## Creating an AWS Account

```hcl
resource "aws_organizations_account" "production" {
  name      = "mycompany-production"
  email     = "aws+production@mycompany.com"

  # Place in the Production OU immediately
  parent_id = aws_organizations_organizational_unit.production.id

  # Role name for cross-account access from the management account
  role_name = "OrganizationAccountAccessRole"

  # Prevent accidental account closure
  close_on_deletion = false

  tags = {
    Environment = "production"
    ManagedBy   = "opentofu"
    CostCenter  = "engineering"
  }
}

output "production_account_id" {
  value = aws_organizations_account.production.id
}
```

## Creating Multiple Accounts

```hcl
variable "accounts" {
  type = map(object({
    email     = string
    parent_ou = string
  }))
  default = {
    "security" = {
      email     = "aws+security@mycompany.com"
      parent_ou = "Security"
    }
    "logging" = {
      email     = "aws+logging@mycompany.com"
      parent_ou = "Infrastructure"
    }
    "shared-services" = {
      email     = "aws+shared@mycompany.com"
      parent_ou = "Infrastructure"
    }
  }
}

locals {
  ou_ids = {
    "Security"       = aws_organizations_organizational_unit.security.id
    "Infrastructure" = aws_organizations_organizational_unit.infrastructure.id
  }
}

resource "aws_organizations_account" "accounts" {
  for_each  = var.accounts
  name      = "mycompany-${each.key}"
  email     = each.value.email
  parent_id = local.ou_ids[each.value.parent_ou]
  role_name = "OrganizationAccountAccessRole"
}
```

## Assuming the Access Role in a New Account

After creating an account, use its access role to provision resources:

```hcl
provider "aws" {
  alias  = "production"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::${aws_organizations_account.production.id}:role/OrganizationAccountAccessRole"
  }
}

# Create a baseline VPC in the new account
resource "aws_vpc" "production_baseline" {
  provider   = aws.production
  cidr_block = "10.1.0.0/16"
  tags = { Name = "production-baseline" }
}
```

## Moving an Account to a Different OU

Update the `parent_id` attribute and apply:

```hcl
resource "aws_organizations_account" "production" {
  parent_id = aws_organizations_organizational_unit.production.id  # Changed from staging OU
  # ...
}
```

## Importing Existing Accounts

If accounts already exist, import them into state:

```bash
tofu import aws_organizations_account.existing 123456789012
```

## Conclusion

Creating AWS accounts with OpenTofu automates the account provisioning pipeline. Use `aws_organizations_account` to create and place accounts in the correct OU, use the `OrganizationAccountAccessRole` to bootstrap resources in new accounts via provider `assume_role`, and manage account lifecycle through state. Account email addresses must be globally unique.
