# How to Deploy Serverless Functions Across Multiple Clouds with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Serverless, Multi-Cloud, AWS Lambda, Azure Functions, Infrastructure as Code

Description: Learn how to use OpenTofu to deploy and manage serverless functions across AWS, Azure, and GCP from a single configuration using modules and provider aliases.

## Introduction

Deploying serverless functions across multiple cloud providers can be complex when done manually. OpenTofu's provider model and module system make it possible to manage AWS Lambda, Azure Functions, and Google Cloud Functions from a single codebase.

## Project Structure

```text
serverless-multi-cloud/
├── main.tf
├── variables.tf
├── outputs.tf
├── modules/
│   ├── aws-lambda/
│   │   └── main.tf
│   ├── azure-function/
│   │   └── main.tf
│   └── gcp-function/
│       └── main.tf
```

## Provider Configuration

```hcl
terraform {
  required_providers {
    aws   = { source = "hashicorp/aws" }
    azurerm = { source = "hashicorp/azurerm" }
    google  = { source = "hashicorp/google" }
  }
}

provider "aws"    { region = var.aws_region }
provider "azurerm" { features {} }
provider "google"  { project = var.gcp_project; region = var.gcp_region }
```

## AWS Lambda Module

```hcl
# modules/aws-lambda/main.tf

resource "aws_lambda_function" "fn" {
  function_name = var.function_name
  runtime       = "python3.11"
  handler       = "handler.lambda_handler"
  role          = aws_iam_role.lambda_exec.arn
  filename      = var.zip_path

  environment {
    variables = var.env_vars
  }
}
```

## Azure Function Module

```hcl
# modules/azure-function/main.tf
resource "azurerm_linux_function_app" "fn" {
  name                = var.function_name
  resource_group_name = var.resource_group
  location            = var.location
  service_plan_id     = azurerm_service_plan.plan.id

  storage_account_name       = azurerm_storage_account.sa.name
  storage_account_access_key = azurerm_storage_account.sa.primary_access_key

  site_config {
    application_stack {
      python_version = "3.11"
    }
  }
}
```

## Calling the Modules in main.tf

```hcl
module "aws_hello" {
  source        = "./modules/aws-lambda"
  function_name = "hello-world-aws"
  zip_path      = "function.zip"
  env_vars      = { ENV = "production" }
}

module "azure_hello" {
  source         = "./modules/azure-function"
  function_name  = "hello-world-azure"
  resource_group = "my-rg"
  location       = "East US"
}
```

## Deploying

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

OpenTofu's multi-provider support and module system make it practical to manage serverless functions across cloud providers with consistent patterns, reducing operational overhead and keeping all configurations in one place.
