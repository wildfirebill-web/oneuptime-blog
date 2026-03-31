# How to Create and Manage Services in ClickHouse Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Cloud, Service Management, Cloud, Provisioning, Administration

Description: Learn how to create, configure, and manage ClickHouse Cloud services including sizing, regions, and operational controls through the console and API.

---

ClickHouse Cloud is the managed service offering that handles infrastructure, replication, and backups for you. Managing services effectively means knowing how to provision, resize, pause, and delete them through both the web console and the API.

## Creating a Service via the Console

1. Log in to [clickhouse.cloud](https://clickhouse.cloud)
2. Click "New Service"
3. Choose a cloud provider (AWS, GCP, or Azure) and region
4. Select a tier: Development, Production, or Dedicated
5. Name your service and click "Create Service"

Development services auto-pause after inactivity to save costs. Production services run continuously.

## Creating a Service via the API

```bash
curl -X POST https://api.clickhouse.cloud/v1/organizations/{orgId}/services \
  -H "Authorization: Bearer $CLICKHOUSE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "analytics-prod",
    "provider": "aws",
    "region": "us-east-1",
    "tier": "production",
    "minTotalMemoryGb": 24,
    "maxTotalMemoryGb": 96
  }'
```

## Listing Services

```bash
curl https://api.clickhouse.cloud/v1/organizations/{orgId}/services \
  -H "Authorization: Bearer $CLICKHOUSE_API_KEY"
```

## Pausing and Resuming a Service

Development services can be paused to stop billing for compute:

```bash
# Pause
curl -X PATCH https://api.clickhouse.cloud/v1/organizations/{orgId}/services/{serviceId}/state \
  -H "Authorization: Bearer $CLICKHOUSE_API_KEY" \
  -d '{"command": "stop"}'

# Resume
curl -X PATCH https://api.clickhouse.cloud/v1/organizations/{orgId}/services/{serviceId}/state \
  -H "Authorization: Bearer $CLICKHOUSE_API_KEY" \
  -d '{"command": "start"}'
```

## Connecting to Your Service

After creation, retrieve connection details from the console or API:

```bash
clickhouse client \
  --host your-service.clickhouse.cloud \
  --port 8443 \
  --user default \
  --password "$PASSWORD" \
  --secure
```

## Deleting a Service

```bash
curl -X DELETE https://api.clickhouse.cloud/v1/organizations/{orgId}/services/{serviceId} \
  -H "Authorization: Bearer $CLICKHOUSE_API_KEY"
```

Deleting a service also removes all associated data unless you have exported a backup.

## Monitoring Service Status

```bash
curl https://api.clickhouse.cloud/v1/organizations/{orgId}/services/{serviceId} \
  -H "Authorization: Bearer $CLICKHOUSE_API_KEY" \
  | jq '.service.state'
```

## Summary

ClickHouse Cloud services are created and managed through the web console or REST API. Use Development tier for non-production workloads to benefit from auto-pause, and Production tier for always-on analytics. Automate service management with the API to integrate into your infrastructure pipelines.
