# How to Configure the OCI Provider in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Oracle Cloud, OCI, Provider Configuration, Infrastructure as Code

Description: Learn how to configure the Oracle Cloud Infrastructure (OCI) provider in OpenTofu using API key authentication and environment variables.

The Oracle Cloud Infrastructure (OCI) provider for OpenTofu lets you manage OCI resources - compute instances, virtual cloud networks, databases, and more - as code. This guide covers authentication methods and provider setup.

## Provider Requirements

```hcl
terraform {
  required_providers {
    oci = {
      source  = "oracle/oci"
      version = "~> 6.0"
    }
  }
}
```

## Authentication: API Key Method

The most common authentication method uses an RSA key pair:

### Step 1: Generate an API Key

```bash
# Generate a 2048-bit RSA key

openssl genrsa -out ~/.oci/oci_api_key.pem 2048

# Set correct permissions
chmod 600 ~/.oci/oci_api_key.pem

# Extract the public key
openssl rsa -pubout -in ~/.oci/oci_api_key.pem -out ~/.oci/oci_api_key_public.pem

# Get the fingerprint
openssl rsa -pubout -outform DER -in ~/.oci/oci_api_key.pem | openssl md5 -c
```

Upload the public key to your OCI user's API Keys in the Console.

### Step 2: Configure the Provider

```hcl
provider "oci" {
  region           = var.region
  tenancy_ocid     = var.tenancy_ocid
  user_ocid        = var.user_ocid
  fingerprint      = var.fingerprint
  private_key_path = var.private_key_path
}

variable "region"           { type = string; default = "us-ashburn-1" }
variable "tenancy_ocid"     { type = string }
variable "user_ocid"        { type = string }
variable "fingerprint"      { type = string }
variable "private_key_path" { type = string; default = "~/.oci/oci_api_key.pem" }
```

## Authentication: OCI Configuration File

If you have the OCI CLI configured (`~/.oci/config`), the provider reads it automatically with no additional configuration:

```hcl
provider "oci" {
  region = "us-ashburn-1"
  # Reads tenancy_ocid, user_ocid, fingerprint, and key_file from ~/.oci/config
}
```

## Authentication: Environment Variables

```bash
export TF_VAR_tenancy_ocid="ocid1.tenancy.oc1..aaaa..."
export TF_VAR_user_ocid="ocid1.user.oc1..aaaa..."
export TF_VAR_fingerprint="aa:bb:cc:..."
export TF_VAR_private_key_path="/path/to/key.pem"
```

Or use OCI environment variables directly:

```bash
export OCI_TENANCY_OCID="ocid1.tenancy.oc1..aaaa..."
export OCI_USER_OCID="ocid1.user.oc1..aaaa..."
export OCI_FINGERPRINT="aa:bb:cc:..."
export OCI_PRIVATE_KEY_PATH="/path/to/key.pem"
export OCI_REGION="us-ashburn-1"
```

## Instance Principal Authentication (for OCI Compute)

For OpenTofu running on an OCI Compute instance:

```hcl
provider "oci" {
  auth   = "InstancePrincipal"
  region = "us-ashburn-1"
}
```

## Multiple Provider Configurations

Use provider aliases for multi-region deployments:

```hcl
provider "oci" {
  alias        = "ashburn"
  region       = "us-ashburn-1"
  tenancy_ocid = var.tenancy_ocid
  # ...
}

provider "oci" {
  alias        = "phoenix"
  region       = "us-phoenix-1"
  tenancy_ocid = var.tenancy_ocid
  # ...
}
```

## Getting Essential OCIDs

You need several OCIDs for OCI resources:

```bash
# Get tenancy OCID
oci iam tenancy get --tenancy-id $(oci iam user get --user-id $(oci iam user list --query "data[?contains(\"name\", \"terraform\")]|[0].id" --raw-output) --query "data.\"compartment-id\"" --raw-output) --query "data.id" --raw-output

# List compartments
oci iam compartment list --all --compartment-id-in-subtree true
```

## Conclusion

The OCI provider supports multiple authentication methods: API key files for local development, the OCI CLI config file for interactive use, environment variables for CI/CD pipelines, and Instance Principal for workloads running on OCI Compute. Start with API key authentication and migrate to Instance Principal for production automation.
