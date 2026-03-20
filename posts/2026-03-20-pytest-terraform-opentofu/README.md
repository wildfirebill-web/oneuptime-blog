# How to Use pytest-terraform with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, pytest, Python, Testing, Infrastructure Testing

Description: Learn how to use pytest-terraform to write Python-based tests for OpenTofu modules, leveraging pytest fixtures and assertions to validate deployed infrastructure.

## Introduction

pytest-terraform is a pytest plugin that provides fixtures for running OpenTofu operations within Python tests. It integrates with the pytest ecosystem — parametrize, fixtures, conftest — making it familiar to Python developers who want to test infrastructure with tools they already know.

## Installation

```bash
pip install pytest-terraform boto3
# or with uv:
uv add pytest-terraform boto3 pytest
```

## Basic Configuration

```python
# conftest.py
import pytest

# Configure pytest-terraform to use tofu instead of terraform
def pytest_configure(config):
    config.addinivalue_line(
        "markers", "terraform: mark test as requiring terraform/tofu"
    )

# pytest.ini or pyproject.toml
```

```ini
# pytest.ini
[pytest]
terraform_binary = tofu
terraform_dir = ../modules
```

## Basic Module Test

```python
# test_vpc_module.py
import pytest
from pytest_terraform import terraform

@terraform("vpc", scope="module")
def test_vpc_creation(vpc):
    """Test that VPC is created with correct configuration."""
    # vpc fixture provides access to outputs
    assert vpc["vpc_id"] is not None
    assert vpc["vpc_id"].startswith("vpc-")

    private_subnets = vpc["private_subnet_ids"]
    assert len(private_subnets) > 0

    public_subnets = vpc["public_subnet_ids"]
    assert len(public_subnets) > 0

@terraform("vpc", scope="module")
def test_vpc_cidr(vpc):
    """Test VPC CIDR block."""
    assert vpc["vpc_cidr"] == "10.99.0.0/16"
```

## Using AWS Boto3 for Validation

```python
# test_security_groups.py
import boto3
import pytest
from pytest_terraform import terraform

@terraform("security_groups", scope="function")
def test_security_group_no_public_ssh(security_groups, aws_region="us-east-1"):
    """Verify no security group allows SSH from 0.0.0.0/0."""
    sg_id = security_groups["web_security_group_id"]

    ec2 = boto3.client("ec2", region_name=aws_region)
    response = ec2.describe_security_groups(GroupIds=[sg_id])
    sg = response["SecurityGroups"][0]

    for rule in sg["IpPermissions"]:
        if rule.get("FromPort") == 22:
            for ip_range in rule.get("IpRanges", []):
                assert ip_range["CidrIp"] != "0.0.0.0/0", \
                    f"SSH port 22 is open to the internet from SG {sg_id}"
            for ipv6_range in rule.get("Ipv6Ranges", []):
                assert ipv6_range["CidrIpv6"] != "::/0", \
                    f"SSH port 22 is open to IPv6 internet from SG {sg_id}"
```

## Fixture for Terraform Variables

```python
# conftest.py
import pytest

@pytest.fixture(scope="session")
def terraform_vars():
    """Common variables for all terraform tests."""
    return {
        "environment": "test",
        "aws_region": "us-east-1",
    }

@pytest.fixture(scope="module")
def vpc_vars(terraform_vars):
    """Variables specific to VPC tests."""
    return {
        **terraform_vars,
        "vpc_cidr": "10.99.0.0/16",
        "azs": ["us-east-1a", "us-east-1b"],
        "private_subnet_cidrs": ["10.99.1.0/24", "10.99.2.0/24"],
        "public_subnet_cidrs":  ["10.99.101.0/24", "10.99.102.0/24"],
    }
```

## Testing with Parametrize

```python
# test_instance_types.py
import pytest
from pytest_terraform import terraform

@pytest.mark.parametrize("instance_type,expected_cpu", [
    ("t3.micro", 2),
    ("t3.small", 2),
    ("t3.medium", 2),
])
def test_instance_cpu_credits(instance_type, expected_cpu):
    """Test that T-series instances have burstable CPU credits."""
    import boto3
    ec2 = boto3.client("ec2", region_name="us-east-1")
    types = ec2.describe_instance_types(InstanceTypes=[instance_type])
    instance_info = types["InstanceTypes"][0]
    assert instance_info["VCpuInfo"]["DefaultVCpus"] == expected_cpu
```

## Directory Structure

```
tests/
├── conftest.py
├── fixtures/
│   ├── vpc/
│   │   ├── main.tf          # Test fixture module
│   │   ├── outputs.tf
│   │   └── variables.tf
│   └── security_groups/
│       ├── main.tf
│       └── outputs.tf
├── test_vpc_module.py
└── test_security_groups.py
```

## Running pytest-terraform Tests

```bash
# Run all tests
pytest tests/ -v

# Run a specific test file
pytest tests/test_vpc_module.py -v

# Run with specific AWS region
AWS_DEFAULT_REGION=us-east-1 pytest tests/ -v

# Keep infrastructure after tests (for debugging)
pytest tests/ --terraform-no-cleanup

# Verbose with output capture disabled
pytest tests/ -v -s
```

## Conclusion

pytest-terraform bridges OpenTofu infrastructure testing with Python's mature testing ecosystem. Teams already using pytest for application testing can apply the same patterns — parametrize, fixtures, conftest — to infrastructure tests. The `scope="module"` option reuses deployed infrastructure across multiple test functions, reducing deployment time and cost when multiple tests validate the same infrastructure.
