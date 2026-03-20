# How to Create Azure Event Hubs Namespaces with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Event Hub, Data Streaming, Infrastructure as Code

Description: Learn how to create Azure Event Hubs namespaces, event hubs, and consumer groups with OpenTofu for high-throughput data streaming and ingestion.

Azure Event Hubs is a managed data streaming platform capable of ingesting millions of events per second. Managing namespaces and event hubs in OpenTofu ensures consistent capacity, retention, and access control configuration.

## Provider Configuration

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

## Event Hubs Namespace

```hcl
resource "azurerm_resource_group" "streaming" {
  name     = "streaming-rg"
  location = "eastus"
}

resource "azurerm_eventhub_namespace" "main" {
  name                = "myapp-eventhubs"
  location            = azurerm_resource_group.streaming.location
  resource_group_name = azurerm_resource_group.streaming.name
  sku                 = "Standard"  # Basic, Standard, Premium
  capacity            = 2           # Throughput Units (Standard only)

  auto_inflate_enabled     = true   # Automatically scale up
  maximum_throughput_units = 10

  tags = {
    Environment = "production"
    Team        = "data-platform"
  }
}
```

## Event Hub

```hcl
resource "azurerm_eventhub" "events" {
  name                = "application-events"
  namespace_name      = azurerm_eventhub_namespace.main.name
  resource_group_name = azurerm_resource_group.streaming.name

  partition_count   = 8    # Determines max throughput and parallelism
  message_retention = 7    # Days (1-7 for Standard, up to 90 for Premium)
}

resource "azurerm_eventhub" "logs" {
  name                = "application-logs"
  namespace_name      = azurerm_eventhub_namespace.main.name
  resource_group_name = azurerm_resource_group.streaming.name

  partition_count   = 4
  message_retention = 1
}
```

## Consumer Groups

```hcl
# Default consumer group is $Default (always exists)

# Analytics consumer group

resource "azurerm_eventhub_consumer_group" "analytics" {
  name                = "analytics"
  namespace_name      = azurerm_eventhub_namespace.main.name
  eventhub_name       = azurerm_eventhub.events.name
  resource_group_name = azurerm_resource_group.streaming.name
}

# Archive consumer group
resource "azurerm_eventhub_consumer_group" "archive" {
  name                = "archive"
  namespace_name      = azurerm_eventhub_namespace.main.name
  eventhub_name       = azurerm_eventhub.events.name
  resource_group_name = azurerm_resource_group.streaming.name
}
```

## Authorization Rules

```hcl
# Namespace-level rule - access to all event hubs
resource "azurerm_eventhub_namespace_authorization_rule" "sender" {
  name                = "sender"
  namespace_name      = azurerm_eventhub_namespace.main.name
  resource_group_name = azurerm_resource_group.streaming.name

  listen = false
  send   = true
  manage = false
}

resource "azurerm_eventhub_namespace_authorization_rule" "receiver" {
  name                = "receiver"
  namespace_name      = azurerm_eventhub_namespace.main.name
  resource_group_name = azurerm_resource_group.streaming.name

  listen = true
  send   = false
  manage = false
}

# Hub-level rule - scoped to one event hub
resource "azurerm_eventhub_authorization_rule" "events_sender" {
  name                = "events-sender"
  namespace_name      = azurerm_eventhub_namespace.main.name
  eventhub_name       = azurerm_eventhub.events.name
  resource_group_name = azurerm_resource_group.streaming.name

  listen = false
  send   = true
  manage = false
}
```

## Capture to Azure Blob Storage

```hcl
resource "azurerm_eventhub" "events_with_capture" {
  name                = "events-captured"
  namespace_name      = azurerm_eventhub_namespace.main.name
  resource_group_name = azurerm_resource_group.streaming.name
  partition_count     = 4
  message_retention   = 1

  capture_description {
    enabled             = true
    encoding            = "Avro"
    interval_in_seconds = 300   # Capture every 5 minutes
    size_limit_in_bytes = 10485760  # 10 MB

    destination {
      name                = "EventHubArchive.AzureBlockBlob"
      archive_name_format = "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"
      blob_container_name = "eventhub-capture"
      storage_account_id  = azurerm_storage_account.capture.id
    }
  }
}
```

## Conclusion

Azure Event Hubs in OpenTofu provides a scalable streaming foundation for high-throughput data pipelines. Create namespaces with auto-inflate for dynamic scaling, partition event hubs appropriately for parallel processing, create separate consumer groups for independent consumers, and enable capture to automatically archive raw events to blob storage for reprocessing.
