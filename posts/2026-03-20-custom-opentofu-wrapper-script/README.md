# How to Build a Custom OpenTofu Wrapper Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Shell Script, Automation, DevOps, CLI, Infrastructure as Code

Description: Learn how to build a custom wrapper script around the OpenTofu CLI to enforce conventions, add safety checks, and simplify common workflows.

## Introduction

A wrapper script around the `tofu` CLI enforces team conventions, adds safety checks, handles authentication, and provides a consistent interface for common operations. It reduces errors and makes the IaC workflow accessible to team members unfamiliar with OpenTofu internals.

## Basic Wrapper Structure

```bash
#!/usr/bin/env bash
# bin/tofu-wrapper.sh
# Usage: ./bin/tofu-wrapper.sh <command> [options]
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"

# ─── Configuration ───────────────────────────────────────────────────────────
TOFU_VERSION="${TOFU_VERSION:-1.9.0}"
STATE_BUCKET="${STATE_BUCKET:-my-tofu-state}"
REGION="${AWS_REGION:-us-east-1}"
ENVIRONMENT="${ENVIRONMENT:-dev}"

# ─── Colors ──────────────────────────────────────────────────────────────────
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'
info()  { echo -e "${GREEN}[INFO]${NC} $*"; }
warn()  { echo -e "${YELLOW}[WARN]${NC} $*"; }
error() { echo -e "${RED}[ERROR]${NC} $*" >&2; exit 1; }

# ─── Validation ──────────────────────────────────────────────────────────────
check_dependencies() {
  for cmd in tofu aws jq; do
    command -v "$cmd" &>/dev/null || error "Required command not found: $cmd"
  done

  # Verify tofu version
  local actual_version
  actual_version=$(tofu version -json | jq -r '.terraform_version')
  if [[ "$actual_version" != "$TOFU_VERSION" ]]; then
    warn "Expected tofu ${TOFU_VERSION}, found ${actual_version}. Consider updating."
  fi
}

check_aws_auth() {
  aws sts get-caller-identity &>/dev/null || error "AWS authentication failed. Run: aws sso login"
  local account
  account=$(aws sts get-caller-identity --query Account --output text)
  info "Authenticated to AWS account: ${account}"
}

check_environment() {
  if [[ "${ENVIRONMENT}" == "prod" && "${CI:-false}" == "false" ]]; then
    warn "You are about to modify PRODUCTION. Are you sure? (yes/no)"
    read -r confirmation
    [[ "$confirmation" == "yes" ]] || error "Aborted."
  fi
}

# ─── Commands ─────────────────────────────────────────────────────────────────
cmd_init() {
  info "Initializing OpenTofu for environment: ${ENVIRONMENT}"
  tofu init \
    -backend-config="bucket=${STATE_BUCKET}" \
    -backend-config="key=${ENVIRONMENT}/terraform.tfstate" \
    -backend-config="region=${REGION}" \
    "$@"
}

cmd_plan() {
  check_environment
  local plan_file="${ENVIRONMENT}.tfplan"
  info "Running plan for environment: ${ENVIRONMENT}"
  tofu plan \
    -var-file="environments/${ENVIRONMENT}.tfvars" \
    -out="${plan_file}" \
    "$@"
  info "Plan saved to: ${plan_file}"
}

cmd_apply() {
  local plan_file="${ENVIRONMENT}.tfplan"
  [[ -f "${plan_file}" ]] || error "No saved plan found. Run 'plan' first."
  check_environment
  info "Applying plan: ${plan_file}"
  tofu apply "${plan_file}"
  rm -f "${plan_file}"
}

cmd_destroy() {
  check_environment
  warn "DESTROY will permanently delete resources in ${ENVIRONMENT}!"
  warn "Type the environment name to confirm: "
  read -r confirmation
  [[ "$confirmation" == "$ENVIRONMENT" ]] || error "Confirmation failed. Aborted."
  tofu destroy \
    -var-file="environments/${ENVIRONMENT}.tfvars" \
    "$@"
}

# ─── Main ─────────────────────────────────────────────────────────────────────
main() {
  check_dependencies
  check_aws_auth

  local command="${1:-help}"
  shift || true

  case "$command" in
    init)    cmd_init "$@" ;;
    plan)    cmd_plan "$@" ;;
    apply)   cmd_apply "$@" ;;
    destroy) cmd_destroy "$@" ;;
    *)
      echo "Usage: $(basename "$0") {init|plan|apply|destroy}"
      exit 1
      ;;
  esac
}

main "$@"
```

## Using the Wrapper

```bash
# Initialize for the staging environment
ENVIRONMENT=staging ./bin/tofu-wrapper.sh init

# Run a plan
ENVIRONMENT=staging ./bin/tofu-wrapper.sh plan

# Apply the saved plan
ENVIRONMENT=staging ./bin/tofu-wrapper.sh apply

# Destroy (requires typing environment name as confirmation)
ENVIRONMENT=staging ./bin/tofu-wrapper.sh destroy
```

## Adding to PATH

```bash
# Add a symlink so the wrapper is accessible from anywhere in the repo
ln -s "${PWD}/bin/tofu-wrapper.sh" "${PWD}/bin/tfw"
chmod +x bin/tfw

# Usage
ENVIRONMENT=dev ./bin/tfw plan
```

## Summary

A custom OpenTofu wrapper script enforces team conventions, validates authentication, adds environment-specific safety prompts, and standardizes the plan/apply workflow. It reduces human error and makes the IaC workflow consistent across team members and CI/CD systems.
