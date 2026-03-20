# How to Write Your First OpenTofu Configuration File

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Beginner, HCL, DevOps, Getting Started

Description: A beginner's guide to writing your first OpenTofu configuration file using HCL syntax, covering providers, resources, variables, and outputs.

---

OpenTofu configuration files use HashiCorp Configuration Language (HCL) — a declarative language designed to be human-readable while being machine-parseable. Your first configuration file teaches the core building blocks: providers, resources, variables, and outputs. This guide walks through each one.

---

## Create Your First Configuration File

OpenTofu files use the `.tf` extension. By convention, a simple project starts with `main.tf`.

```hcl
# main.tf — your first OpenTofu configuration file

# 1. The terraform block: declares OpenTofu version requirements and providers
terraform {
  required_version = ">= 1.8.0"

  required_providers {
    # Use the local provider — it works without credentials for learning
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

# 2. Configure the provider
provider "local" {
  # The local provider needs no configuration
}

# 3. Declare a resource — this creates a file on your local filesystem
resource "local_file" "hello" {
  # The file content
  content  = "Hello from OpenTofu!"

  # Where to create the file
  filename = "${path.module}/hello.txt"
}

# 4. Declare an output — displays information after apply
output "file_path" {
  description = "The path of the created file"
  value       = local_file.hello.filename
}
```

---

## Initialize and Apply

```bash
# Initialize the working directory (downloads providers)
tofu init

# Preview what will be created
tofu plan

# Create the infrastructure (the file in this case)
tofu apply

# Confirm by typing 'yes' when prompted
# Output: Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

# Check the output value
tofu output file_path
# "./hello.txt"

# Verify the file was created
cat hello.txt
# Hello from OpenTofu!
```

---

## Adding a Variable

Make the configuration reusable by adding an input variable.

```hcl
# variables.tf — input variables
variable "greeting" {
  type        = string
  description = "The greeting message to write to the file"
  default     = "Hello from OpenTofu!"
}

variable "output_filename" {
  type        = string
  description = "Name of the output file"
  default     = "hello.txt"
}
```

```hcl
# main.tf — updated to use variables
resource "local_file" "hello" {
  content  = var.greeting
  filename = "${path.module}/${var.output_filename}"
}
```

---

## The Complete Project Structure

```
my-first-project/
├── main.tf           # resources and providers
├── variables.tf      # input variable declarations
├── outputs.tf        # output value declarations
├── versions.tf       # version constraints (optional but recommended)
└── .terraform/       # created by tofu init (don't commit this)
```

---

## Summary

A minimal OpenTofu configuration has four parts: a `terraform` block declaring the required version and providers, a `provider` block configuring the provider, one or more `resource` blocks defining the infrastructure to create, and optionally `output` blocks to display results. Start with the `local` provider to learn the workflow before moving to cloud providers.
