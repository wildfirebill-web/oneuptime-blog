# How to Use tofu validate to Check Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use tofu validate to check OpenTofu configuration for syntax errors and internal consistency without accessing remote state or provider APIs.

## Introduction

`tofu validate` checks configuration files for syntax errors, type mismatches, missing required arguments, and other structural issues. It does not connect to cloud providers or read state. It is fast and safe to run in any environment, making it a useful first step in CI/CD pipelines before running `plan`.

## Basic Usage

```bash
tofu validate

# Success:
# Success! The configuration is valid.

# Error:
# Error: Unsupported argument
# On main.tf line 12, in resource "aws_s3_bucket" "data":
#   12:   bucket_name = "acme-data"
# An argument named "bucket_name" is not expected here.
# Did you mean "bucket"?
```

## What validate Checks

- HCL syntax errors
- Unknown resource types or data sources (for initialized providers)
- Invalid argument names
- Type mismatches (string where number expected, etc.)
- Missing required arguments
- Invalid references (referencing non-existent outputs or locals)
- Circular dependencies between resources

## What validate Does NOT Check

- Whether credentials are valid
- Whether referenced resources actually exist in the cloud
- Whether plan would succeed
- Remote state contents

## Run validate Before plan

```bash
# Fast and catches common errors before the slower plan
tofu validate && tofu plan
```

## JSON Output for CI/CD

```bash
# Machine-readable validation output
tofu validate -json

# Output:
# {
#   "valid": true,
#   "error_count": 0,
#   "warning_count": 0,
#   "diagnostics": []
# }
```

Parse with jq:

```bash
VALID=$(tofu validate -json | jq -r '.valid')
if [ "$VALID" != "true" ]; then
  echo "Validation failed"
  tofu validate -json | jq '.diagnostics[]'
  exit 1
fi
```

## Validate Without Init

```bash
# validate requires initialized providers to check resource schemas
# Run init first
tofu init -backend=false  # Skip backend, just download providers
tofu validate
```

The `-backend=false` flag initializes providers without configuring a backend — useful for validation in CI before backend credentials are available.

## Pre-Commit Hook

```bash
# .git/hooks/pre-commit
#!/bin/bash
tofu validate
if [ $? -ne 0 ]; then
  echo "OpenTofu validation failed. Fix errors before committing."
  exit 1
fi
```

## CI/CD Integration

```yaml
# GitHub Actions validation step
- name: Validate
  run: |
    tofu init -backend=false -input=false
    tofu validate -json | tee validate-output.json

    VALID=$(jq -r '.valid' validate-output.json)
    if [ "$VALID" != "true" ]; then
      echo "Validation failed:"
      jq '.diagnostics[]' validate-output.json
      exit 1
    fi
```

## Validate Multiple Directories

```bash
# Validate all configuration directories
find . -name "*.tf" -not -path "*/.terraform/*" \
  -exec dirname {} \; | sort -u | while read DIR; do
    echo "Validating $DIR"
    (cd "$DIR" && tofu init -backend=false -input=false && tofu validate)
done
```

## Conclusion

`tofu validate` is a fast, safe check that catches configuration errors without touching infrastructure or state. Run it as the first step in your CI/CD pipeline after `tofu init -backend=false`. Use `-json` output for programmatic parsing in pipelines. It will not catch all errors — provider-specific validation happens at plan time — but it eliminates common syntax and structural mistakes before you pay for a full plan run.
