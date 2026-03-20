# How to Create Azure Cosmos DB with MongoDB API Using OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Cosmos DB, MongoDB, OpenTofu, NoSQL, Database

Description: Learn how to create Azure Cosmos DB with the MongoDB API using OpenTofu for MongoDB-compatible storage with global distribution and elastic scale.

## Overview

Azure Cosmos DB's MongoDB API provides wire-protocol compatibility with MongoDB, allowing you to use existing MongoDB drivers and tools while benefiting from Cosmos DB's global distribution and automatic scaling. OpenTofu manages the full lifecycle of these resources.

## Step 1: Create the Cosmos DB Account with MongoDB API

```hcl
# main.tf - Cosmos DB account with MongoDB API

resource "azurerm_cosmosdb_account" "mongodb" {
  name                = "my-mongodb-cosmos"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  offer_type          = "Standard"
  kind                = "MongoDB"  # MongoDB API

  # MongoDB version compatibility
  mongo_server_version = "4.2"

  automatic_failover_enabled = true

  consistency_policy {
    consistency_level = "Session"
  }

  # Primary write region
  geo_location {
    location          = "eastus"
    failover_priority = 0
  }

  # Read replica in secondary region
  geo_location {
    location          = "westus2"
    failover_priority = 1
  }

  # Enable MongoDB-specific capabilities
  capabilities {
    name = "EnableMongo"
  }

  capabilities {
    name = "MongoDBv3.4"
  }
}
```

## Step 2: Create a MongoDB Database

```hcl
# MongoDB database
resource "azurerm_cosmosdb_mongo_database" "app_db" {
  name                = "appdb"
  resource_group_name = azurerm_resource_group.rg.name
  account_name        = azurerm_cosmosdb_account.mongodb.name

  # Shared throughput at the database level
  throughput = 400
}
```

## Step 3: Create MongoDB Collections

```hcl
# Collection with a shard key for horizontal scaling
resource "azurerm_cosmosdb_mongo_collection" "users" {
  name                = "users"
  resource_group_name = azurerm_resource_group.rg.name
  account_name        = azurerm_cosmosdb_account.mongodb.name
  database_name       = azurerm_cosmosdb_mongo_database.app_db.name

  # Shard key for data distribution (equivalent to partition key)
  shard_key = "userId"

  # Throughput for this specific collection (overrides database-level)
  throughput = 400

  # Create indexes on the collection
  index {
    keys   = ["_id"]
    unique = true
  }

  index {
    keys   = ["email"]
    unique = true
  }

  index {
    keys   = ["createdAt"]
    unique = false
  }

  # TTL index for automatic document expiry
  default_ttl_seconds = -1  # -1 disables TTL
}

resource "azurerm_cosmosdb_mongo_collection" "products" {
  name                = "products"
  resource_group_name = azurerm_resource_group.rg.name
  account_name        = azurerm_cosmosdb_account.mongodb.name
  database_name       = azurerm_cosmosdb_mongo_database.app_db.name
  shard_key           = "categoryId"

  index {
    keys   = ["_id"]
    unique = true
  }

  index {
    keys   = ["sku"]
    unique = true
  }
}
```

## Step 4: Outputs

```hcl
output "mongodb_connection_string" {
  value       = azurerm_cosmosdb_account.mongodb.connection_strings[0]
  sensitive   = true
  description = "MongoDB-compatible connection string"
}

output "cosmos_endpoint" {
  value = azurerm_cosmosdb_account.mongodb.endpoint
}
```

## Summary

Azure Cosmos DB with MongoDB API, managed via OpenTofu, provides a drop-in replacement for MongoDB with the added benefits of global distribution and SLA-backed availability. Existing MongoDB applications need only a connection string change to use Cosmos DB as the backend.
