# How to Create Your First OpenTofu Module - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Modules

Description: Learn how to create your first reusable OpenTofu module by extracting infrastructure code into a composable, configurable unit.

## Introduction

An OpenTofu module is a collection of `.tf` files in a directory that define infrastructure resources. Modules encapsulate resource configurations, accept input variables, and expose output values, enabling code reuse across projects and teams.

## What Makes a Module

A module is simply a directory containing:
- `main.tf` - resource definitions
- `variables.tf` - input variable declarations
- `outputs.tf` - output value declarations
- `README.md` - documentation (recommended)

## Step-by-Step: Creating a Simple EC2 Module

### Step 1: Create the Module Directory

```bash
mkdir -p modules/ec2-instance
```

### Step 2: Define Variables (`variables.tf`)

```hcl
# modules/ec2-instance/variables.tf

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}

variable "name" {
  type        = string
  description = "Name tag for the instance"
}

variable "subnet_id" {
  type        = string
  description = "Subnet ID to launch the instance in"
}

variable "security_group_ids" {
  type        = list(string)
  description = "List of security group IDs"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod."
  }
}

variable "tags" {
  type        = map(string)
  description = "Additional tags to apply"
  default     = {}
}
```

### Step 3: Define Resources (`main.tf`)

```hcl
# modules/ec2-instance/main.tf

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "this" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = merge(
    {
      Name        = var.name
      Environment = var.environment
      ManagedBy   = "OpenTofu"
    },
    var.tags
  )
}
```

### Step 4: Define Outputs (`outputs.tf`)

```hcl
# modules/ec2-instance/outputs.tf

output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.this.id
}

output "private_ip" {
  description = "Private IP address of the instance"
  value       = aws_instance.this.private_ip
}

output "public_ip" {
  description = "Public IP address (if assigned)"
  value       = aws_instance.this.public_ip
}
```

### Step 5: Call the Module

```hcl
# root/main.tf

module "web_server" {
  source = "./modules/ec2-instance"

  name               = "web-server"
  instance_type      = "t3.medium"
  subnet_id          = module.vpc.public_subnet_ids[0]
  security_group_ids = [aws_security_group.web.id]
  environment        = "prod"

  tags = {
    Team    = "platform"
    Service = "web"
  }
}

output "web_server_ip" {
  value = module.web_server.private_ip
}
```

## Module Invocation Syntax

```hcl
module "<label>" {
  source = "<path-or-registry-address>"

  # Input variables
  var1 = value1
  var2 = value2
}
```

## Key Benefits of Modules

1. **DRY** - Define once, use many times
2. **Encapsulation** - Hide complexity, expose clean interfaces
3. **Versioning** - Pin module versions for stability
4. **Testing** - Test modules independently
5. **Composition** - Combine modules to build complex systems

## Conclusion

Creating your first OpenTofu module is straightforward: define variables, write resources, declare outputs, and reference the module with a `module` block. Start by extracting repeated infrastructure patterns from your root module into reusable modules, and your infrastructure code will become significantly more maintainable and consistent.
