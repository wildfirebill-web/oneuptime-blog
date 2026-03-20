# How to Test OpenTofu Configurations with Terratest (Go-Based Testing)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Terratest, Testing, Go, Infrastructure as Code

Description: Learn how to write automated infrastructure tests for OpenTofu configurations using Terratest, a Go-based testing framework that deploys real infrastructure and validates its behavior.

## Introduction

Terratest is a Go testing library from Gruntwork that allows you to write automated tests that deploy real infrastructure, validate its behavior, and then destroy it. This gives you confidence that your OpenTofu modules work correctly in practice.

## Prerequisites

- Go 1.21 or later installed
- OpenTofu installed
- Cloud provider credentials configured

## Setting Up a Test Project

Create a test directory:

```text
modules/
  networking/
    main.tf
    variables.tf
    outputs.tf
test/
  networking_test.go
  go.mod
```

Initialize Go module:

```bash
cd test
go mod init github.com/myorg/infra-tests
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/stretchr/testify/assert
```

## Writing a Basic Test

```go
// test/networking_test.go
package test

import (
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestNetworkingModule(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir:    "../modules/networking",
        TerraformBinary: "tofu",
        Vars: map[string]interface{}{
            "name":       "test-network",
            "cidr_block": "10.0.0.0/16",
            "region":     "us-east-1",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)

    subnetIds := terraform.OutputList(t, terraformOptions, "subnet_ids")
    assert.Equal(t, 3, len(subnetIds))
}
```

## Testing with AWS SDK Validation

```go
import (
    "github.com/gruntwork-io/terratest/modules/aws"
)

func TestEC2Instance(t *testing.T) {
    region := "us-east-1"
    opts := &terraform.Options{
        TerraformDir:    "../modules/compute",
        TerraformBinary: "tofu",
    }

    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    instanceId := terraform.Output(t, opts, "instance_id")

    instance := aws.GetEc2InstanceById(t, instanceId, region)
    assert.Equal(t, "running", aws.GetInstanceState(t, instanceId, region))
    assert.Equal(t, "t3.micro", *instance.InstanceType)
}
```

## Running Tests

```bash
cd test
go test -v -run TestNetworkingModule -timeout 30m
```

Run all tests:

```bash
go test -v -timeout 60m ./...
```

## Best Practices

- Use unique, random names to avoid test conflicts: `terratest/modules/random`
- Always use `defer terraform.Destroy()` to clean up
- Set generous timeouts for real infrastructure operations
- Use `t.Parallel()` to run independent tests concurrently

## Conclusion

Terratest brings the discipline of real end-to-end testing to infrastructure code. By deploying actual resources and validating them with Go's testing framework, you catch bugs that static analysis and unit tests miss.
