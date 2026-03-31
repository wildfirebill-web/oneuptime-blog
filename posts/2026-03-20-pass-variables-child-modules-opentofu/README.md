# How to Pass Variables to Child Modules in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Module, Variable, HCL, Infrastructure as Code, DevOps

Description: Learn how to pass input variables from a root module to child modules and how to expose child module outputs back to the caller.

---

OpenTofu modules communicate through input variables (passing values in) and output values (exposing results out). When you call a module, you pass values for its input variables as module arguments. This guide covers how to structure and pass variables to child modules effectively.

---

## Module Structure

```text
infrastructure/
├── main.tf           # root module - calls child modules
├── variables.tf      # root module variables
├── outputs.tf        # root module outputs
└── modules/
    └── networking/
        ├── main.tf
        ├── variables.tf   # child module variable declarations
        └── outputs.tf     # child module output declarations
```

---

## Declaring Variables in the Child Module

```hcl
# modules/networking/variables.tf - child module inputs

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  default     = "development"
}

variable "availability_zones" {
  type        = list(string)
  description = "Availability zones to use"
}
```

---

## Calling the Child Module with Variables

```hcl
# main.tf - root module calling the child module

variable "environment" {
  type = string
}

module "networking" {
  source = "./modules/networking"

  # Pass variables to the child module as arguments
  vpc_cidr           = "10.0.0.0/16"
  environment        = var.environment   # pass root variable to child
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
}
```

---

## Passing Complex Types

```hcl
module "application" {
  source = "./modules/application"

  # Pass a map
  tags = {
    Environment = var.environment
    ManagedBy   = "opentofu"
  }

  # Pass a list derived from another module's output
  subnet_ids = module.networking.private_subnet_ids

  # Pass an object
  server_config = {
    instance_type = var.environment == "production" ? "m5.large" : "t3.micro"
    disk_size_gb  = 50
  }
}
```

---

## Accessing Child Module Outputs

```hcl
# modules/networking/outputs.tf - child module exports

output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

```hcl
# main.tf - root module uses child module outputs
resource "aws_eks_cluster" "main" {
  name = "my-cluster"

  vpc_config {
    # Reference child module output
    subnet_ids = module.networking.private_subnet_ids
    vpc_id     = module.networking.vpc_id
  }
}
```

---

## Forwarding Root Module Variables to Multiple Modules

```hcl
# main.tf - pass the same variable to multiple modules

module "networking" {
  source      = "./modules/networking"
  environment = var.environment  # forwarded from root
  region      = var.region       # forwarded from root
}

module "database" {
  source      = "./modules/database"
  environment = var.environment  # same variable, different module
  vpc_id      = module.networking.vpc_id
}

module "application" {
  source      = "./modules/application"
  environment = var.environment
  db_endpoint = module.database.endpoint
  vpc_id      = module.networking.vpc_id
}
```

---

## Summary

Pass variables to child modules as named arguments in the `module` block - each argument corresponds to a declared `variable` in the child module. Use `var.<name>` to forward root module variables to children, and use `module.<name>.<output>` to pass a child module's output as input to another module. Required variables (no default) in the child module must always be explicitly provided.
