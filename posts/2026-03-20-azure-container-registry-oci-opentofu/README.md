# How to Use Azure Container Registry as OCI Registry for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure Container Registry, OCI Registry, Azure, Provider Distribution

Description: Learn how to use Azure Container Registry as an OCI registry for distributing OpenTofu providers and modules in Azure-centric environments.

## Introduction

Azure Container Registry (ACR) is OCI-compliant and supports storing arbitrary OCI artifacts alongside container images. For Azure-centric organizations, ACR offers Azure AD authentication, geo-replication, private endpoints, and integration with existing Azure infrastructure - making it ideal for OpenTofu provider and module distribution.

## Creating ACR for OpenTofu

```hcl
# acr.tf

resource "azurerm_resource_group" "opentofu" {
  name     = "opentofu-registry-rg"
  location = "East US"
}

resource "azurerm_container_registry" "opentofu" {
  name                = "mycompanyopentofu"
  resource_group_name = azurerm_resource_group.opentofu.name
  location            = azurerm_resource_group.opentofu.location
  sku                 = "Premium"  # Required for geo-replication and private endpoints

  admin_enabled = false  # Use Azure AD, not admin credentials

  tags = {
    Purpose = "opentofu-registry"
  }
}

# Geo-replication for multi-region availability

resource "azurerm_container_registry_replication" "west_europe" {
  name                    = "westeurope"
  container_registry_name = azurerm_container_registry.opentofu.name
  resource_group_name     = azurerm_resource_group.opentofu.name
  location                = "West Europe"
}

# Private endpoint for secure access
resource "azurerm_private_endpoint" "acr" {
  name                = "acr-private-endpoint"
  location            = azurerm_resource_group.opentofu.location
  resource_group_name = azurerm_resource_group.opentofu.name
  subnet_id           = azurerm_subnet.private.id

  private_service_connection {
    name                           = "acr-psc"
    private_connection_resource_id = azurerm_container_registry.opentofu.id
    is_manual_connection           = false
    subresource_names              = ["registry"]
  }
}
```

## Authentication

```bash
# Authenticate using Azure CLI (uses your Azure AD identity)
az acr login --name mycompanyopentofu

# For service principals (CI/CD)
az acr login \
  --name mycompanyopentofu \
  --username "$SERVICE_PRINCIPAL_ID" \
  --password "$SERVICE_PRINCIPAL_SECRET"

# Using Docker credential helper
# docker login mycompanyopentofu.azurecr.io
# with username/password from service principal
```

## Assigning ACR Roles

```hcl
# Role assignment for CI/CD service principal (push)
resource "azurerm_role_assignment" "acr_push" {
  scope                = azurerm_container_registry.opentofu.id
  role_definition_name = "AcrPush"
  principal_id         = azurerm_service_principal.cicd.object_id
}

# Role assignment for workload identities (pull)
resource "azurerm_role_assignment" "acr_pull" {
  for_each = toset([
    azurerm_user_assigned_identity.dev_machines.principal_id,
    azurerm_user_assigned_identity.staging.principal_id,
  ])

  scope                = azurerm_container_registry.opentofu.id
  role_definition_name = "AcrPull"
  principal_id         = each.value
}
```

## Pushing Providers to ACR

```bash
#!/bin/bash
# push-provider-to-acr.sh

set -euo pipefail

ACR_NAME="mycompanyopentofu"
ACR_REGISTRY="${ACR_NAME}.azurecr.io"
PROVIDER_NAMESPACE="hashicorp"
PROVIDER_TYPE="azurerm"
PROVIDER_VERSION="3.80.0"

# Login
az acr login --name "$ACR_NAME"

# Download provider
WORK_DIR=$(mktemp -d)
trap "rm -rf $WORK_DIR" EXIT

cat > "$WORK_DIR/versions.tf" << EOF
terraform {
  required_providers {
    azurerm = {
      source  = "${PROVIDER_NAMESPACE}/${PROVIDER_TYPE}"
      version = "= ${PROVIDER_VERSION}"
    }
  }
}
EOF

cd "$WORK_DIR"
tofu init -backend=false
tofu providers mirror \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  "$WORK_DIR/mirror/"

cd "$WORK_DIR/mirror/registry.opentofu.org/${PROVIDER_NAMESPACE}/${PROVIDER_TYPE}/"

ACR_REPO="${ACR_REGISTRY}/opentofu-providers/${PROVIDER_NAMESPACE}-${PROVIDER_TYPE}"

oras push "${ACR_REPO}:${PROVIDER_VERSION}" \
  --config /dev/null:application/vnd.opentofu.provider.config.v1+json \
  "terraform-provider-${PROVIDER_TYPE}_${PROVIDER_VERSION}_linux_amd64.zip:application/vnd.opentofu.provider.v1.linux.amd64" \
  "terraform-provider-${PROVIDER_TYPE}_${PROVIDER_VERSION}_linux_arm64.zip:application/vnd.opentofu.provider.v1.linux.arm64" \
  "terraform-provider-${PROVIDER_TYPE}_${PROVIDER_VERSION}_SHA256SUMS:application/vnd.opentofu.provider.v1.shasums"

echo "Pushed: ${ACR_REPO}:${PROVIDER_VERSION}"
```

## Pushing Modules to ACR

```bash
#!/bin/bash
# push-module-to-acr.sh

MODULE_NAME="${1:?Usage: $0 <module-dir> <version>}"
VERSION="${2:?}"
ACR_NAME="mycompanyopentofu"
ACR_REGISTRY="${ACR_NAME}.azurecr.io"

az acr login --name "$ACR_NAME"

tar -czf "${MODULE_NAME}-${VERSION}.tgz" \
  --exclude='.terraform' \
  --exclude='*.tfstate*' \
  --exclude='.git' \
  -C "$MODULE_NAME" .

ACR_REPO="${ACR_REGISTRY}/opentofu-modules/${MODULE_NAME}"

oras push "${ACR_REPO}:${VERSION}" \
  --config /dev/null:application/vnd.opentofu.module.config.v1+json \
  "${MODULE_NAME}-${VERSION}.tgz:application/vnd.opentofu.module.v1.tar+gzip"

oras tag "${ACR_REGISTRY}" \
  "opentofu-modules/${MODULE_NAME}:${VERSION}" \
  "opentofu-modules/${MODULE_NAME}:latest"

rm "${MODULE_NAME}-${VERSION}.tgz"
echo "Pushed: ${ACR_REPO}:${VERSION}"
```

## Configuring OpenTofu to Use ACR

```hcl
# ~/.terraform.rc

provider_installation {
  oci_mirror {
    url     = "oci://mycompanyopentofu.azurecr.io/opentofu-providers"
    include = ["registry.opentofu.org/hashicorp/*"]
  }

  direct {
    exclude = ["registry.opentofu.org/hashicorp/*"]
  }
}
```

```hcl
# For modules in ACR
module "vpc" {
  source = "oci://mycompanyopentofu.azurecr.io/opentofu-modules/azure-vnet:2.1.0"

  name                = "production"
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]
}
```

## ACR Token Scopes for Fine-Grained Access

```bash
# Create ACR token for read-only access to specific repositories
az acr token create \
  --name "opentofu-readonly" \
  --registry "$ACR_NAME" \
  --scope-map "opentofu-providers-pull"

# Create scope map
az acr scope-map create \
  --name "opentofu-providers-pull" \
  --registry "$ACR_NAME" \
  --repository "opentofu-providers/hashicorp-azurerm" content/read \
  --repository "opentofu-providers/hashicorp-kubernetes" content/read
```

## Conclusion

Azure Container Registry provides Azure AD authentication, geo-replication, private endpoints, and repository-scoped tokens for OpenTofu provider and module distribution. The `AcrPush` and `AcrPull` built-in roles cover most use cases, while ACR tokens provide fine-grained access to specific repositories. For CI/CD systems, use a service principal with `AcrPush` role; for developer machines and workloads, use `az acr login` with individual Azure AD identities or `AcrPull` role assignments on managed identities.
