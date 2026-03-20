# How to Set Up Global Server Load Balancing for IPv4 with Cloudflare

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cloudflare, GSLB, Load Balancing, IPv4, DNS, Global Traffic Management

Description: Configure Cloudflare Load Balancing to distribute IPv4 traffic globally across multiple origin servers using health checks, geo-steering, and failover policies.

## Introduction

Cloudflare Load Balancing provides DNS-based global server load balancing (GSLB) that distributes traffic across origin servers in multiple regions. It combines health monitoring, geographic steering, and automatic failover - all managed through Cloudflare's edge network.

## Prerequisites

- A Cloudflare account with the Load Balancing add-on (available on Pro and above)
- At least two origin servers (or pools) with public IPv4 addresses
- Your domain managed by Cloudflare DNS

## Step 1: Create Origin Pools

Pools are groups of servers that receive traffic. Create pools per region:

```bash
# Create a pool using the Cloudflare API

curl -s -X POST "https://api.cloudflare.com/client/v4/user/load_balancers/pools" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "us-east-pool",
    "description": "US East Coast origin servers",
    "enabled": true,
    "minimum_origins": 1,
    "origins": [
      {
        "name": "web-1",
        "address": "203.0.113.10",
        "enabled": true,
        "weight": 1
      },
      {
        "name": "web-2",
        "address": "203.0.113.11",
        "enabled": true,
        "weight": 1
      }
    ],
    "monitor": "MONITOR_ID"
  }'
```

## Step 2: Create a Health Monitor

```bash
# Create an HTTP health monitor
curl -s -X POST "https://api.cloudflare.com/client/v4/user/load_balancers/monitors" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "http",
    "description": "HTTP health check",
    "method": "GET",
    "path": "/health",
    "interval": 60,
    "retries": 2,
    "timeout": 5,
    "expected_codes": "2xx",
    "follow_redirects": false
  }'
```

## Step 3: Create the Load Balancer

```bash
# Create a load balancer with geographic steering
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/load_balancers" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "api.example.com",
    "description": "Global API load balancer",
    "default_pools": ["us-east-pool-id", "eu-west-pool-id"],
    "fallback_pool": "us-east-pool-id",
    "proxied": true,
    "steering_policy": "geo",
    "region_pools": {
      "WNAM": ["us-west-pool-id"],
      "ENAM": ["us-east-pool-id"],
      "WEU": ["eu-west-pool-id"]
    }
  }'
```

## Steering Policies

| Policy | Behavior |
|--------|---------|
| `off` | Route to pools in order |
| `geo` | Route to nearest pool by region |
| `random` | Random pool selection |
| `dynamic_latency` | Route to lowest-latency pool |
| `proximity` | Route by geographic proximity to origin |

## Failover Behavior

When a pool becomes unhealthy (all origins fail health checks), Cloudflare automatically routes traffic to the next pool in the priority list. With `proxied: true`, this happens without DNS TTL delays.

## Terraform Configuration

```hcl
resource "cloudflare_load_balancer" "api" {
  zone_id          = var.zone_id
  name             = "api.example.com"
  fallback_pool_id = cloudflare_load_balancer_pool.us_east.id
  default_pool_ids = [
    cloudflare_load_balancer_pool.us_east.id,
    cloudflare_load_balancer_pool.eu_west.id
  ]
  steering_policy  = "geo"
  proxied          = true

  region_pools {
    region   = "ENAM"
    pool_ids = [cloudflare_load_balancer_pool.us_east.id]
  }
}
```

## Monitoring Load Balancer Health

Use OneUptime or Cloudflare Analytics to monitor origin health and traffic distribution. Cloudflare's analytics show request distribution across pools and health check pass/fail rates.

## Conclusion

Cloudflare Load Balancing provides enterprise-grade GSLB with minimal operational overhead. Geographic steering with health-based failover ensures clients always reach the nearest healthy origin, delivering low latency and high availability globally.
