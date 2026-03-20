# How to Use expect_failures in OpenTofu Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Expect_failures, Error Testing, Infrastructure as Code

Description: Learn how to use the `expect_failures` argument in OpenTofu test run blocks to verify that validation rules, preconditions, and postconditions produce the expected errors.

## Introduction

`expect_failures` is an argument in `run` blocks that inverts the success condition: the run is marked as **passing** when the listed resources, variables, or data sources raise an error, and **failing** when they do not. It is the primary mechanism for testing error paths in OpenTofu modules.

## Syntax

```hcl
run "test_name" {
  command = plan  # or apply

  variables {
    # Input designed to trigger a failure
  }

  # List references that should fail
  expect_failures = [
    var.some_variable,
    resource_type.resource_name,
    data.data_type.data_name,
    check.check_block_name,
  ]
}
```

## Testing Variable Validation

```hcl
# The variable under test (in the module)

# variable "environment" {
#   type = string
#   validation {
#     condition     = contains(["dev", "staging", "prod"], var.environment)
#     error_message = "environment must be one of: dev, staging, prod"
#   }
# }

run "rejects_invalid_environment_value" {
  command = plan

  variables {
    environment = "production"  # Not in the allowed list
  }

  expect_failures = [
    var.environment,
  ]
}
```

## Testing Resource Preconditions

```hcl
run "rejects_oversized_instance_in_dev" {
  command = plan

  variables {
    environment   = "dev"
    instance_type = "m5.4xlarge"  # Should be rejected in dev
  }

  # The precondition on aws_instance.app is expected to fail
  expect_failures = [
    aws_instance.app,
  ]
}
```

## Testing Resource Postconditions

Postconditions fire after apply. To test them, you may need a contrived configuration or a mock that returns unexpected values:

```hcl
run "postcondition_fires_when_encryption_missing" {
  # Using a mock provider that returns unencrypted disk
  command = apply

  variables {
    enforce_encryption = false
  }

  expect_failures = [
    aws_ebs_volume.data,
  ]
}
```

## Testing `check` Blocks

OpenTofu `check` blocks (available since 1.5) run assertions after every apply. You can target them with `expect_failures`:

```hcl
# In the module:
# check "health_endpoint_reachable" {
#   assert {
#     condition     = data.http.health.status_code == 200
#     error_message = "Health endpoint returned non-200 status"
#   }
# }

run "check_block_fires_on_bad_endpoint" {
  command = apply

  variables {
    # URL that returns 503
    health_url = "https://httpstat.us/503"
  }

  expect_failures = [
    check.health_endpoint_reachable,
  ]
}
```

## Multiple Expected Failures

You can list multiple targets if the configuration should fail in several places simultaneously:

```hcl
run "multiple_validation_failures" {
  command = plan

  variables {
    environment = "invalid"   # fails var.environment
    region      = "us-fake-1" # fails var.region
  }

  expect_failures = [
    var.environment,
    var.region,
  ]
}
```

## Common Pitfalls

**Do not use `expect_failures` and `assert` together for the same resource.** If a resource is listed in `expect_failures`, the run completes when that resource fails-there is nothing to assert on:

```hcl
run "bad_example" {
  variables { environment = "bad" }

  expect_failures = [var.environment]

  # This assert will never run because the plan stops at the validation error
  assert {
    condition     = aws_instance.this.id != ""  # unreachable
    error_message = "Never reached"
  }
}
```

**Verify the right thing fails.** If a different resource or variable fails instead of the one listed in `expect_failures`, the test will still pass-which may mask a real issue. Be specific with your variable values to trigger exactly one failure path.

## Conclusion

`expect_failures` is essential for building a complete test suite. Without it, your tests only verify the happy path. With it, you can prove that your module rejects bad inputs gracefully and with clear error messages.
