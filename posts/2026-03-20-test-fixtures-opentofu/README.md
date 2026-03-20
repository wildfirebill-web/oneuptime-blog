# How to Set Up Test Fixtures for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Fixtures, Test Infrastructure, Test Setup

Description: Learn how to create and manage OpenTofu test fixtures — reusable infrastructure configurations that set up prerequisites for module testing and integration tests.

## Introduction

Test fixtures are minimal OpenTofu configurations that create the prerequisite infrastructure needed to test a module in isolation. A VPC module test might need no fixtures, but a database module test needs a VPC fixture already deployed. Good fixtures are fast to apply, cheap to run, and easy to understand.

## Fixture Structure

```
modules/
└── database/
    ├── main.tf
    ├── outputs.tf
    ├── variables.tf
    └── tests/
        ├── unit/
        │   └── unit.tftest.hcl     # Uses mock_provider
        ├── integration/
        │   └── integration.tftest.hcl  # Uses real AWS
        └── fixtures/
            └── networking/          # Prerequisite: VPC for the DB test
                ├── main.tf
                ├── outputs.tf
                └── terraform.tfvars
```

## Creating a VPC Fixture

```hcl
# tests/fixtures/networking/main.tf
# Minimal VPC for use as a test fixture

provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "test" {
  cidr_block           = "10.99.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "test-fixture-vpc", Purpose = "testing" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.test.id
  cidr_block        = "10.99.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = { Name = "test-fixture-private-${count.index}", Purpose = "testing" }
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_db_subnet_group" "test" {
  name       = "test-fixture-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_security_group" "db" {
  name   = "test-fixture-db-sg"
  vpc_id = aws_vpc.test.id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.test.cidr_block]
  }
}
```

```hcl
# tests/fixtures/networking/outputs.tf
output "vpc_id"             { value = aws_vpc.test.id }
output "private_subnet_ids" { value = aws_subnet.private[*].id }
output "db_subnet_group"    { value = aws_db_subnet_group.test.name }
output "db_security_group"  { value = aws_security_group.db.id }
```

## Using Fixtures in Test Files

```hcl
# tests/integration/database_test.tftest.hcl

# Reference the pre-deployed fixture
# Option 1: Deploy the fixture inline in the test
run "setup_networking" {
  command = apply

  module {
    source = "./fixtures/networking"
  }
}

# Use the fixture's outputs in the module under test
run "test_database_creation" {
  command = apply

  variables {
    vpc_id          = run.setup_networking.vpc_id
    subnet_ids      = run.setup_networking.private_subnet_ids
    db_subnet_group = run.setup_networking.db_subnet_group
    security_group  = run.setup_networking.db_security_group
    environment     = "test"
    db_identifier   = "test-db-${formatdate("MMDDhhmmss", timestamp())}"
  }

  assert {
    condition     = aws_db_instance.main.status == "available"
    error_message = "Database should be in available state"
  }
}
```

## Shared Fixtures with Terraform Data Sources

```hcl
# For long-running fixtures, deploy once and reference via data source

# tests/fixtures/shared/main.tf (deployed once per test session)
# ...

# tests/integration/database_test.tftest.hcl
# Reference pre-existing fixture via data sources
data "aws_vpc" "test_fixture" {
  tags = {
    Name    = "test-fixture-vpc"
    Purpose = "testing"
  }
}

data "aws_subnets" "test_fixture_private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.test_fixture.id]
  }
  tags = {
    Purpose = "testing"
  }
}

run "test_with_shared_fixture" {
  command = apply

  variables {
    vpc_id     = data.aws_vpc.test_fixture.id
    subnet_ids = data.aws_subnets.test_fixture_private.ids
  }
}
```

## Fixture Lifecycle Management

```bash
# scripts/setup-test-fixtures.sh
#!/bin/bash

echo "Deploying test fixtures..."
cd tests/fixtures/networking
tofu init
tofu apply -auto-approve
FIXTURE_OUTPUTS=$(tofu output -json)

# Export fixture outputs for tests
export TEST_VPC_ID=$(echo $FIXTURE_OUTPUTS | jq -r '.vpc_id.value')
export TEST_SUBNET_IDS=$(echo $FIXTURE_OUTPUTS | jq -r '.private_subnet_ids.value | join(",")')

echo "Fixtures deployed: VPC=$TEST_VPC_ID"
```

```bash
# scripts/teardown-test-fixtures.sh
#!/bin/bash

echo "Destroying test fixtures..."
cd tests/fixtures/networking
tofu destroy -auto-approve
echo "Fixtures destroyed"
```

## CI/CD Integration

```yaml
# .github/workflows/integration-tests.yml
jobs:
  integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.TEST_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy fixtures
        run: bash scripts/setup-test-fixtures.sh

      - name: Run integration tests
        run: tofu test tests/integration/ -verbose

      - name: Teardown fixtures
        if: always()  # Always clean up
        run: bash scripts/teardown-test-fixtures.sh
```

## Conclusion

Test fixtures reduce integration test complexity by separating prerequisite infrastructure from the module under test. The `run "setup" { module { source = "./fixtures/..." } }` pattern in OpenTofu test files is convenient for self-contained tests, while shared fixtures deployed once per test session are more efficient for suites with many test files that need the same prerequisites.
