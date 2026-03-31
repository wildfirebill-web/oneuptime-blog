# How to Choose Azure Cache for Redis Tier

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Azure, Azure Cache for Redis, Tier, Architecture

Description: Learn how to choose between Azure Cache for Redis Basic, Standard, Premium, and Enterprise tiers based on availability, features, and performance requirements.

---

Azure Cache for Redis offers four tiers. Picking the wrong one leads to either over-spending or missing critical features like clustering, persistence, or geo-replication.

## Tier Comparison

| Feature | Basic | Standard | Premium | Enterprise |
|---|---|---|---|---|
| SLA | None | 99.9% | 99.9% | 99.99% |
| Replication | No | Yes | Yes | Yes (active-active) |
| Clustering | No | No | Yes | Yes |
| Persistence (AOF/RDB) | No | No | Yes | Yes |
| Geo-replication | No | No | Passive | Active-Active |
| VNet Injection | No | No | Yes | Yes |
| Max Memory | 53 GB | 53 GB | 1.2 TB | 14 TB |
| Modules (Search, etc.) | No | No | No | Yes |

## Basic

Use only for dev/test. No replication means any node failure results in data loss and downtime.

```bash
az redis create \
  --name my-redis-dev \
  --resource-group rg-dev \
  --location eastus \
  --sku Basic \
  --vm-size c1
```

## Standard

Good for production workloads that need the 99.9% SLA with primary/replica replication but don't need clustering or persistence.

```bash
az redis create \
  --name my-redis-prod \
  --resource-group rg-prod \
  --location eastus \
  --sku Standard \
  --vm-size c2
```

## Premium

Use Premium when you need: clustering, AOF/RDB persistence, Virtual Network injection, or zone redundancy.

```bash
az redis create \
  --name my-redis-premium \
  --resource-group rg-prod \
  --location eastus \
  --sku Premium \
  --vm-size p2 \
  --enable-non-ssl-port false \
  --minimum-tls-version 1.2
```

## Enterprise

Use Enterprise for: modules (RediSearch, RedisBloom), active-active geo-replication, 99.99% SLA, or datasets exceeding 1.2 TB.

```bash
az redis enterprise create \
  --name my-redis-enterprise \
  --resource-group rg-prod \
  --location eastus \
  --sku EnterpriseFlash_F300 \
  --capacity 3
```

## Terraform - Premium with Zone Redundancy

```hcl
resource "azurerm_redis_cache" "premium" {
  name                = "my-redis-premium"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  capacity            = 2
  family              = "P"
  sku_name            = "Premium"
  enable_non_ssl_port = false
  minimum_tls_version = "1.2"
  zones               = ["1", "2", "3"]

  redis_configuration {
    maxmemory_policy = "allkeys-lru"
    rdb_backup_enabled = true
    rdb_backup_frequency = 60
    rdb_storage_connection_string = var.storage_connection_string
  }
}
```

## Decision Tree

```text
Dev/Test only?         -> Basic
Need SLA, no clustering?  -> Standard
Need clustering/persistence?  -> Premium
Need modules or active-active geo?  -> Enterprise
```

## Summary

Basic and Standard tiers cover dev and simple production workloads respectively. Premium adds clustering, persistence, and VNet injection - required for most serious production deployments. Enterprise is needed only when you require Redis modules, active-active geo-replication, or very large dataset support. Always start with Standard or Premium for production and scale up if needed.
