# How to Use Data Sources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Data Source, HCL, Infrastructure as Code, DevOps, Read-Only

Description: Learn how to use data sources in OpenTofu to read existing infrastructure information and reference it in your resource configurations.

---

Data sources let you read information from external systems without creating or managing infrastructure. They query existing resources - like fetching the latest AMI, looking up a VPC ID, or reading a secret - and make that data available for use in your resources and locals.

---

## Basic Data Source Syntax

```hcl
# data.<provider_type>.<local_name>

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Reference data source attributes
resource "aws_instance" "web" {
  # Use the AMI ID from the data source
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}
```

---

## Common Data Source Patterns

### Look Up an Existing Resource by Name

```hcl
# Find a VPC by name tag
data "aws_vpc" "production" {
  tags = {
    Name        = "production-vpc"
    Environment = "production"
  }
}

resource "aws_subnet" "new_service" {
  vpc_id     = data.aws_vpc.production.id   # use the found VPC
  cidr_block = "10.0.50.0/24"
}
```

### Get Current Account/Region Information

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

resource "aws_iam_role" "app" {
  name = "app-role"
  assume_role_policy = jsonencode({
    Statement = [{
      Action    = "sts:AssumeRole"
      Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
    }]
  })
}
```

### Read an AWS Secret

```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/database/password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
  # ...
}
```

---

## Data Sources vs Resources

| | Resource | Data Source |
|---|---|---|
| Keyword | `resource` | `data` |
| Creates infrastructure | Yes | No |
| Manages lifecycle | Yes | Read-only |
| Reference syntax | `<type>.<name>.<attr>` | `data.<type>.<name>.<attr>` |

---

## Filtering Data Sources

```hcl
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [aws_vpc.main.id]
  }

  tags = {
    Type = "private"
  }
}

resource "aws_lb" "internal" {
  internal        = true
  subnets         = data.aws_subnets.private.ids   # list of subnet IDs
  load_balancer_type = "application"
}
```

---

## When Data Sources Run

Data sources are evaluated during `tofu plan`. If a data source query fails (resource not found, permissions error), the plan fails. Add `depends_on` if the data source queries a resource that must exist first.

---

## Summary

Data sources read information from existing infrastructure and external systems without creating or managing resources. Use `data.<type>.<name>` to define a data source and `data.<type>.<name>.<attribute>` to reference its values. Data sources are perfect for looking up AMIs, existing VPCs, secrets, account IDs, and any other information your resources need but don't create.
