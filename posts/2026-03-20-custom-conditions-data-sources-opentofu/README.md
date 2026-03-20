# How to Add Custom Conditions to Data Sources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Data Source, Preconditions, Postconditions, Validation, HCL, Infrastructure as Code

Description: Learn how to add custom conditions - preconditions and postconditions - to data sources in OpenTofu to validate infrastructure before it is referenced in your configuration.

---

OpenTofu allows you to add `precondition` and `postcondition` blocks inside a data source's `lifecycle` block. These act as guardrails: preconditions validate inputs before the query runs, and postconditions validate the discovered data meets your requirements.

---

## Basic Postcondition: Verify What You Found

```hcl
data "aws_security_group" "app" {
  name   = "app-security-group"
  vpc_id = var.vpc_id

  lifecycle {
    postcondition {
      # Ensure the security group actually belongs to our expected VPC
      condition     = self.vpc_id == var.vpc_id
      error_message = "Found security group does not belong to the expected VPC (${var.vpc_id})."
    }
  }
}
```

---

## Precondition: Validate Before Querying

```hcl
variable "aws_region" {
  type = string
}

data "aws_ami" "base" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  lifecycle {
    precondition {
      # Only proceed if the region is one we support
      condition     = contains(["us-east-1", "us-west-2", "eu-west-1"], var.aws_region)
      error_message = "AMI lookup is only supported in us-east-1, us-west-2, and eu-west-1."
    }
  }
}
```

---

## Validate Infrastructure State

```hcl
data "aws_db_instance" "primary" {
  db_instance_identifier = "production-postgres"

  lifecycle {
    postcondition {
      condition     = self.db_instance_status == "available"
      error_message = "The production database is not in 'available' state. Current: ${self.db_instance_status}."
    }

    postcondition {
      condition     = self.multi_az == true
      error_message = "Production RDS must have Multi-AZ enabled for high availability."
    }

    postcondition {
      condition     = self.deletion_protection == true
      error_message = "Production RDS must have deletion protection enabled."
    }
  }
}
```

---

## Validate TLS Certificate Status

```hcl
data "aws_acm_certificate" "api" {
  domain   = "api.${var.domain_name}"
  statuses = ["ISSUED"]

  lifecycle {
    postcondition {
      condition     = self.status == "ISSUED"
      error_message = "TLS certificate for api.${var.domain_name} is not ISSUED. Status: ${self.status}."
    }

    postcondition {
      # Ensure the certificate covers both apex and wildcard
      condition = contains(self.subject_alternative_names, "*.${var.domain_name}")
      error_message = "Certificate must include wildcard SAN *.${var.domain_name}."
    }
  }
}
```

---

## Validate IAM Role Exists and is Assumable

```hcl
data "aws_iam_role" "execution" {
  name = "ecs-task-execution-role"

  lifecycle {
    postcondition {
      # Verify the role exists and has the expected path
      condition     = startswith(self.arn, "arn:aws:iam::")
      error_message = "ECS execution role ARN is invalid: ${self.arn}."
    }
  }
}
```

---

## Multiple Conditions

You can add multiple `precondition` and `postcondition` blocks to a single data source:

```hcl
data "aws_vpc" "production" {
  id = var.production_vpc_id

  lifecycle {
    precondition {
      condition     = length(var.production_vpc_id) > 0
      error_message = "production_vpc_id variable must not be empty."
    }

    precondition {
      condition     = startswith(var.production_vpc_id, "vpc-")
      error_message = "production_vpc_id must be a valid VPC ID (starts with 'vpc-')."
    }

    postcondition {
      condition     = self.state == "available"
      error_message = "Production VPC must be in 'available' state."
    }

    postcondition {
      condition     = self.enable_dns_hostnames == true
      error_message = "Production VPC must have DNS hostnames enabled."
    }
  }
}
```

---

## When Conditions Are Evaluated

- **Preconditions** run before the data source is read, during planning
- **Postconditions** run after the data source is read - if the data source read fails (not found), the postcondition won't run; if it succeeds, postconditions run before the result is used

---

## Summary

Custom conditions on data sources let you validate both inputs and results at plan time. Use `precondition` to validate variables and configuration before querying, and `postcondition` to enforce that discovered infrastructure meets your standards - correct state, required settings, expected properties. The `self` reference inside `postcondition` gives access to all the data source's attributes.
