# How to Use Postconditions on Resources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Validation

Description: Learn how to use postconditions in OpenTofu lifecycle blocks to validate that resources are in the expected state after creation or update.

## Introduction

Postconditions are assertions that must be true after a resource is created or updated. If a postcondition fails, OpenTofu marks the apply as failed, even if the resource was provisioned. This prevents dependent resources from proceeding in an invalid state. Use postconditions to verify that the cloud provider provisioned resources with the expected attributes.

## Basic Postcondition

```hcl
resource "aws_s3_bucket" "data" {
  bucket = var.bucket_name

  lifecycle {
    postcondition {
      condition     = self.bucket == var.bucket_name
      error_message = "Bucket was created with name '${self.bucket}' but expected '${var.bucket_name}'."
    }
  }
}
```

Note: postconditions use `self` to reference the resource's own attributes.

## Validate Computed Attributes

```hcl
resource "aws_lb" "app" {
  name               = "acme-app-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = var.public_subnet_ids

  lifecycle {
    postcondition {
      # Verify the ALB was assigned a DNS name
      condition     = self.dns_name != ""
      error_message = "Load balancer was not assigned a DNS name."
    }
  }
}
```

## Validate Provider-Assigned Values

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  lifecycle {
    postcondition {
      # Ensure the instance got a private IP (it always should, but verify)
      condition     = self.private_ip != ""
      error_message = "EC2 instance was not assigned a private IP address."
    }

    postcondition {
      # Verify instance launched in the expected AZ
      condition     = startswith(self.availability_zone, var.region)
      error_message = "Instance launched in unexpected AZ: ${self.availability_zone}"
    }
  }
}
```

## Postcondition on Data Source

```hcl
data "aws_s3_object" "config" {
  bucket = aws_s3_bucket.config.bucket
  key    = "app/config.json"

  lifecycle {
    postcondition {
      condition     = self.content_type == "application/json"
      error_message = "Config file must be JSON. Got content-type: ${self.content_type}"
    }
  }
}
```

## Multiple Postconditions

```hcl
resource "aws_db_instance" "main" {
  identifier     = var.db_identifier
  engine         = "postgres"
  instance_class = var.db_instance_class
  # ...

  lifecycle {
    postcondition {
      condition     = self.status == "available"
      error_message = "RDS instance is not available. Status: ${self.status}"
    }

    postcondition {
      condition     = self.storage_encrypted == true
      error_message = "RDS storage encryption was not enabled."
    }

    postcondition {
      condition     = self.multi_az == true || var.environment != "production"
      error_message = "Production RDS must be Multi-AZ."
    }
  }
}
```

## Postconditions Protect Downstream Resources

When a postcondition fails, dependent resources are not created:

```hcl
resource "aws_db_instance" "main" {
  identifier = "production-db"

  lifecycle {
    postcondition {
      condition     = self.endpoint != ""
      error_message = "Database endpoint was not provisioned."
    }
  }
}

# This resource will NOT be created if the postcondition above fails

resource "aws_ssm_parameter" "db_endpoint" {
  name  = "/production/db_endpoint"
  value = aws_db_instance.main.endpoint  # References the postconditioned resource
}
```

## EKS Cluster Postcondition

```hcl
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks.arn
  version  = var.kubernetes_version

  lifecycle {
    postcondition {
      condition     = self.status == "ACTIVE"
      error_message = "EKS cluster did not reach ACTIVE status. Current status: ${self.status}"
    }

    postcondition {
      condition     = self.kubernetes_network_config[0].service_ipv4_cidr != ""
      error_message = "EKS cluster was not assigned a service CIDR."
    }
  }
}
```

## Postcondition vs Check Block

```text
Postcondition:
- Runs after the specific resource is created/updated
- BLOCKS apply if it fails
- Uses 'self' to reference the resource
- Prevents dependent resources from being created

Check block:
- Runs after ALL resources are applied
- WARNS but does NOT block apply
- References any resource in configuration
- For continuous validation
```

## Conclusion

Postconditions validate that the cloud provider provisioned resources as expected. Use `self` to reference the resource's computed attributes and check values like endpoint URLs, assigned IPs, or provisioning status. Postconditions block the apply on failure, preventing resources in invalid states from serving as inputs to downstream resources. For non-critical assertions that should warn without blocking, use check blocks instead.
