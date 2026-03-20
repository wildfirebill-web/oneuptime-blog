# How to Use Output Values in Remote State Data Sources

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Outputs, Remote State, Data Sources, Infrastructure as Code, DevOps

Description: A guide to using terraform_remote_state data source to access output values from other OpenTofu configurations.

## Introduction

The `terraform_remote_state` data source allows one OpenTofu configuration to access the output values of another configuration's state file. This is how separate infrastructure projects communicate — the networking project exposes VPC IDs as outputs, and the application project reads them as remote state data.

## Setting Up Remote State Output

```hcl
# infrastructure/networking/outputs.tf
# These outputs will be consumed by other configurations

output "vpc_id" {
  description = "VPC ID for use by other infrastructure stacks"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "bastion_security_group_id" {
  description = "Security group ID for bastion host access"
  value       = aws_security_group.bastion.id
}
```

## Reading Remote State

```hcl
# infrastructure/application/main.tf

# Read outputs from the networking stack
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "my-company-tofu-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use the outputs from networking stack
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Use VPC ID from remote state
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]

  vpc_security_group_ids = [
    aws_security_group.app.id,
    data.terraform_remote_state.networking.outputs.bastion_security_group_id
  ]

  tags = {
    Name = "application-server"
  }
}

resource "aws_security_group" "app" {
  vpc_id = data.terraform_remote_state.networking.outputs.vpc_id
  name   = "app-sg"

  # Allow traffic from bastion
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [data.terraform_remote_state.networking.outputs.bastion_security_group_id]
  }
}
```

## Multiple Remote State Sources

```hcl
# Reading from multiple remote states
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-tofu-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

data "terraform_remote_state" "security" {
  backend = "s3"
  config = {
    bucket = "my-tofu-state"
    key    = "security/terraform.tfstate"
    region = "us-east-1"
  }
}

data "terraform_remote_state" "shared_services" {
  backend = "s3"
  config = {
    bucket = "my-tofu-state"
    key    = "shared-services/terraform.tfstate"
    region = "us-east-1"
  }
}

# Combine outputs from multiple states
resource "aws_ecs_task_definition" "app" {
  family = "my-app"

  network_mode = "awsvpc"

  # Container gets IP from networking state
  # Uses log group from shared services state

  container_definitions = jsonencode([{
    name  = "app"
    image = "my-app:latest"
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group  = data.terraform_remote_state.shared_services.outputs.log_group_name
        awslogs-region = var.aws_region
      }
    }
  }])
}
```

## Dynamic Backend Configuration

```hcl
# Use variables for environment-specific remote state
variable "environment" {
  type = string
}

data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-tofu-state"
    key    = "${var.environment}/networking/terraform.tfstate"
    region = "us-east-1"
  }
}
```

## Alternative: Using SSM Parameter Store

Instead of remote state, you can use SSM Parameter Store to share values:

```hcl
# networking stack writes values to SSM
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/${var.environment}/networking/vpc_id"
  type  = "String"
  value = aws_vpc.main.id
}

# application stack reads from SSM (no remote state data source needed)
data "aws_ssm_parameter" "vpc_id" {
  name = "/${var.environment}/networking/vpc_id"
}

resource "aws_security_group" "app" {
  vpc_id = data.aws_ssm_parameter.vpc_id.value
}
```

## Conclusion

The `terraform_remote_state` data source enables loose coupling between infrastructure stacks while sharing essential identifiers. Design your remote state outputs to be a stable API that consuming stacks depend on, and be careful about breaking changes. For very large organizations, alternatives like SSM Parameter Store or dedicated service registries can provide more decoupled sharing mechanisms without requiring cross-state access permissions.
