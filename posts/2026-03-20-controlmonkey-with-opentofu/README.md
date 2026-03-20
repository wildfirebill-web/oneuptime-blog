# How to Use ControlMonkey with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, ControlMonkey, Drift Detection, Infrastructure as Code, Governance, DevOps

Description: Learn how to integrate ControlMonkey with OpenTofu to gain continuous drift detection, self-service infrastructure provisioning, and automated policy enforcement across your cloud estate.

## Introduction

ControlMonkey is a cloud infrastructure automation platform that connects to your OpenTofu configurations and continuously reconciles them with the actual state of your cloud environment. Its primary differentiators are autonomous drift remediation and a self-service catalog that lets developers provision approved infrastructure without writing code.

## Connecting ControlMonkey to a Repository

ControlMonkey manages stacks through its provider:

```hcl
# controlmonkey.tf

terraform {
  required_providers {
    controlmonkey = {
      source  = "control-monkey/controlmonkey"
      version = "~> 1.0"
    }
  }
}

provider "controlmonkey" {
  # Token set via CONTROLMONKEY_TOKEN environment variable
}

# Create a namespace (maps to a team or business unit)
resource "controlmonkey_namespace" "platform" {
  name = "platform-team"
}

# Create a stack backed by OpenTofu
resource "controlmonkey_stack" "networking" {
  name         = "aws-networking"
  namespace_id = controlmonkey_namespace.platform.id

  iac_config {
    iac_type       = "openTofu"
    opentofu_version = "1.9.0"
  }

  vcs_info {
    provider_id  = "vcs-github-connection-id"
    repo_name    = "my-org/infra-repo"
    branch       = "main"
    working_directory = "stacks/networking"
  }
}
```

## Enabling Drift Detection

ControlMonkey continuously monitors deployed resources and compares them to the OpenTofu state. Configure remediation behavior per stack:

```hcl
resource "controlmonkey_stack" "networking" {
  name         = "aws-networking"
  namespace_id = controlmonkey_namespace.platform.id

  iac_config {
    iac_type         = "openTofu"
    opentofu_version = "1.9.0"
  }

  drift_detection {
    # How often to check for drift (in minutes)
    cron            = "0 */6 * * *"
    # Automatically create a plan when drift is detected
    auto_remediate  = false
  }

  deployment_approval {
    # Require a human to approve before applying drift remediation
    require_approval = true
  }
}
```

## Self-Service Blueprint Example

A blueprint is a pre-approved, parameterized OpenTofu module that developers can deploy without infrastructure knowledge:

```hcl
resource "controlmonkey_blueprint" "web_app" {
  name        = "Standard Web Application"
  description = "Deploys an EC2 instance, ALB, and RDS in a standard configuration"

  iac_config {
    iac_type         = "openTofu"
    opentofu_version = "1.9.0"
    working_directory = "blueprints/web-app"
    repo_name        = "my-org/infra-blueprints"
    branch           = "main"
  }

  # Variables exposed to developers in the self-service form
  variable_overrides = [
    {
      name          = "environment"
      is_overridable = true
    },
    {
      name          = "instance_type"
      is_overridable = true
      allowed_values = ["t3.micro", "t3.small", "t3.medium"]
    }
  ]
}
```

## Policy as Code Integration

ControlMonkey evaluates OPA policies during plan:

```hcl
resource "controlmonkey_policy_group" "security_baseline" {
  name = "security-baseline"

  policy {
    name              = "require-encryption"
    rego_code         = file("policies/require_encryption.rego")
    enforcement_level = "MANDATORY"
  }
}

resource "controlmonkey_policy_group_assignment" "networking_policies" {
  policy_group_id = controlmonkey_policy_group.security_baseline.id
  scope_type      = "NAMESPACE"
  scope_id        = controlmonkey_namespace.platform.id
}
```

## Importing Existing Resources

ControlMonkey can scan your cloud account and generate OpenTofu code for unmanaged resources:

```bash
# From the ControlMonkey CLI
controlmonkey scan --cloud aws --region us-east-1 --output ./imported

# Review generated code, then import into a stack
controlmonkey import --stack-id stack-abc123 --path ./imported
```

## Conclusion

ControlMonkey extends OpenTofu with continuous drift detection, self-service infrastructure blueprints, and OPA policy enforcement. Teams that want to move beyond reactive drift remediation toward proactive governance - where drift is detected and triaged automatically - will find ControlMonkey a compelling complement to a standard OpenTofu workflow.
