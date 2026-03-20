# How to Use ignore_changes Lifecycle in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Resources, Lifecycle, ignore_changes, Infrastructure as Code, DevOps

Description: A guide to using ignore_changes lifecycle in OpenTofu to prevent specific resource attributes from being managed by OpenTofu after initial creation.

## Introduction

The `ignore_changes` lifecycle setting tells OpenTofu to ignore differences for specific attributes when planning changes. This is useful when attributes are managed externally (by autoscalers, external processes, or manual operations) and should not be reverted by OpenTofu.

## Basic ignore_changes

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = [
      ami,  # Don't modify AMI if changed externally
      tags  # Ignore tag changes made outside OpenTofu
    ]
  }
}
```

## Common Use Cases

### Ignoring Auto Scaling Changes

```hcl
resource "aws_autoscaling_group" "web" {
  name             = "web-asg"
  min_size         = 2
  max_size         = 10
  desired_capacity = 2

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  lifecycle {
    # desired_capacity is managed by Auto Scaling policies
    ignore_changes = [desired_capacity]
  }
}
```

### Ignoring External Tag Changes

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name        = "web-server"
    Environment = var.environment
    # CostCenter is added by your tagging automation - ignore it
  }

  lifecycle {
    # Don't overwrite tags added by cost allocation automation
    ignore_changes = [tags["CostCenter"], tags["AutoScalingGroupName"]]
  }
}
```

### Ignoring Password Changes

```hcl
resource "aws_db_instance" "main" {
  identifier     = "production-db"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  username       = "admin"
  password       = var.initial_password  # Only used at creation

  lifecycle {
    # After creation, passwords are rotated externally (e.g., via Secrets Manager rotation)
    ignore_changes = [password]
  }
}
```

### Ignoring Computed Values

```hcl
resource "aws_ecs_task_definition" "app" {
  family                   = "app"
  container_definitions    = jsonencode([...])

  lifecycle {
    # ECS updates the revision counter - ignore it
    ignore_changes = [container_definitions]
  }
}
```

## ignore_changes = all (Use with Caution)

```hcl
# Ignore ALL changes - OpenTofu only manages creation and deletion
resource "aws_instance" "manually_managed" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = all  # Extreme - avoid unless necessary
  }
}
```

## Nested Attribute Ignoring

```hcl
resource "aws_autoscaling_group" "web" {
  # ...

  lifecycle {
    # Ignore specific nested attributes
    ignore_changes = [
      tag,             # Ignore all tags added by AWS
      load_balancers,  # Ignore load balancer attachments managed externally
    ]
  }
}
```

## When NOT to Use ignore_changes

```hcl
# AVOID: Using ignore_changes to suppress errors
resource "aws_s3_bucket" "main" {
  bucket = "my-bucket"

  lifecycle {
    # BAD: Hiding configuration drift instead of fixing it
    ignore_changes = all
  }
}

# BETTER: Fix the underlying configuration drift
# Or use data sources to read external state instead of managing it
```

## Combining with Other Lifecycle Settings

```hcl
resource "aws_eks_cluster" "main" {
  name     = "production"
  version  = var.kubernetes_version

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes = [
      # EKS can update this during maintenance
      kubernetes_network_config,
    ]
  }
}
```

## Conclusion

`ignore_changes` bridges the gap between OpenTofu-managed infrastructure and externally-modified attributes. It's particularly valuable for autoscaling groups (managed capacity), password rotation, and resources with attributes modified by other automation. Use it judiciously — overusing it can cause OpenTofu to become unaware of significant drift, undermining the benefits of infrastructure as code. Document why each `ignore_changes` entry exists to help future engineers understand the intent.
