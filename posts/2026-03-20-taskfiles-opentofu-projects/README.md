# How to Create Taskfiles for OpenTofu Projects

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Taskfile, Task Runner, Automation, DevOps, Infrastructure as Code

Description: Learn how to use Taskfile (go-task) as a modern alternative to Makefiles for organizing and running OpenTofu workflows with better cross-platform support.

## Introduction

Taskfile is a modern task runner written in Go that provides YAML-based task definitions with built-in variable handling, task dependencies, and cross-platform compatibility. It is an excellent alternative to Makefiles for OpenTofu projects.

## Installing Task

```bash
# macOS

brew install go-task

# Linux
sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b ~/.local/bin

# Windows (scoop)
scoop install task
```

## Basic Taskfile

```yaml
# Taskfile.yml
version: '3'

vars:
  ENVIRONMENT:
    sh: echo "${ENVIRONMENT:-dev}"
  REGION:
    sh: echo "${AWS_REGION:-us-east-1}"
  PLAN_FILE: "{{.ENVIRONMENT}}.tfplan"
  VARS_FILE: "environments/{{.ENVIRONMENT}}.tfvars"

env:
  TF_CLI_ARGS_plan: "-var-file={{.VARS_FILE}}"
  TF_CLI_ARGS_apply: ""

tasks:
  default:
    desc: Show available tasks
    cmds:
      - task --list

  init:
    desc: Initialize OpenTofu for the specified environment
    cmds:
      - |
        tofu init \
          -backend-config="bucket=my-state-{{.ENVIRONMENT}}" \
          -backend-config="key={{.ENVIRONMENT}}/terraform.tfstate" \
          -backend-config="region={{.REGION}}" \
          -upgrade

  validate:
    desc: Validate configuration syntax
    cmds:
      - tofu validate

  fmt:
    desc: Format all Terraform files
    cmds:
      - tofu fmt -recursive

  fmt-check:
    desc: Check formatting (for CI)
    cmds:
      - tofu fmt -recursive -check -diff

  plan:
    desc: Run OpenTofu plan and save to file
    deps: [validate]
    cmds:
      - tofu plan -var-file={{.VARS_FILE}} -out={{.PLAN_FILE}}
    generates:
      - "{{.PLAN_FILE}}"

  apply:
    desc: Apply the saved plan
    preconditions:
      - sh: test -f {{.PLAN_FILE}}
        msg: "Plan file not found. Run 'task plan' first."
    cmds:
      - tofu apply {{.PLAN_FILE}}
      - rm -f {{.PLAN_FILE}}

  destroy:
    desc: Destroy all resources (with confirmation)
    prompt: "This will destroy ALL resources in {{.ENVIRONMENT}}. Continue?"
    cmds:
      - tofu destroy -var-file={{.VARS_FILE}} -auto-approve

  output:
    desc: Show all outputs as JSON
    cmds:
      - tofu output -json | jq .

  state-list:
    desc: List all resources in state
    cmds:
      - tofu state list

  lint:
    desc: Run tflint
    cmds:
      - tflint --recursive

  security-scan:
    desc: Run tfsec security scan
    cmds:
      - tfsec .

  ci:
    desc: Full CI pipeline (fmt-check, validate, plan)
    cmds:
      - task: fmt-check
      - task: validate
      - task: plan
```

## Running Tasks

```bash
# List all tasks
task

# Initialize for staging
ENVIRONMENT=staging task init

# Plan and apply
ENVIRONMENT=staging task plan
ENVIRONMENT=staging task apply

# Run full CI pipeline
ENVIRONMENT=staging task ci
```

## Advanced Features

```yaml
# Taskfile.yml additions

  # Task with multiple environments using includes
includes:
  dev:
    taskfile: ./Taskfile.yml
    dir: .
    vars:
      ENVIRONMENT: dev
  staging:
    taskfile: ./Taskfile.yml
    dir: .
    vars:
      ENVIRONMENT: staging

  # Watch for file changes and revalidate
  watch-validate:
    desc: Validate on file changes
    watch: true
    sources:
      - "**/*.tf"
      - "**/*.tofu"
    cmds:
      - task: validate
```

## Calling Specific Environment Tasks

```bash
# With includes, call environment-specific tasks
task dev:plan
task staging:apply
task staging:destroy
```

## Summary

Taskfile provides a modern, YAML-based alternative to Makefiles for OpenTofu project automation. With built-in variable handling, preconditions, interactive prompts, and cross-platform support, Taskfile makes OpenTofu workflows accessible, consistent, and easy to maintain.
