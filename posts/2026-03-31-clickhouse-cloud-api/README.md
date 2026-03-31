# How to Use ClickHouse Cloud API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Cloud, API, REST API, Automation, DevOps

Description: Learn how to use the ClickHouse Cloud REST API to manage services, users, and configurations programmatically for automation and infrastructure-as-code workflows.

---

The ClickHouse Cloud API is a REST API that gives you full programmatic control over your ClickHouse Cloud organization - creating services, managing users, configuring networking, and accessing usage metrics. It is the foundation for infrastructure-as-code and CI/CD integrations.

## Authentication

The API uses Bearer token authentication. Generate an API key in the ClickHouse Cloud console under "Organization Settings" - "API Keys":

```bash
export CLICKHOUSE_API_KEY="your-api-key-here"
export ORG_ID="your-org-id"
```

## Base URL

```text
https://api.clickhouse.cloud/v1
```

## List All Services

```bash
curl https://api.clickhouse.cloud/v1/organizations/${ORG_ID}/services \
  -H "Authorization: Bearer ${CLICKHOUSE_API_KEY}" \
  | jq '.result[] | {id: .id, name: .name, state: .state}'
```

## Get Service Details

```bash
curl https://api.clickhouse.cloud/v1/organizations/${ORG_ID}/services/${SERVICE_ID} \
  -H "Authorization: Bearer ${CLICKHOUSE_API_KEY}"
```

## Create a Service

```bash
curl -X POST https://api.clickhouse.cloud/v1/organizations/${ORG_ID}/services \
  -H "Authorization: Bearer ${CLICKHOUSE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "prod-analytics",
    "provider": "aws",
    "region": "us-east-1",
    "tier": "production",
    "minTotalMemoryGb": 24,
    "maxTotalMemoryGb": 96
  }'
```

## Update Service Settings

```bash
curl -X PATCH https://api.clickhouse.cloud/v1/organizations/${ORG_ID}/services/${SERVICE_ID} \
  -H "Authorization: Bearer ${CLICKHOUSE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "minTotalMemoryGb": 48,
    "ipAccessList": [
      {"source": "10.0.0.0/8", "description": "Internal network"}
    ]
  }'
```

## Manage Organization Members

```bash
# List members
curl https://api.clickhouse.cloud/v1/organizations/${ORG_ID}/members \
  -H "Authorization: Bearer ${CLICKHOUSE_API_KEY}"
```

## Query Execution via API

You can also run SQL queries through the service's HTTP interface:

```bash
curl -X POST "https://${SERVICE_HOST}:8443" \
  -H "X-ClickHouse-User: default" \
  -H "X-ClickHouse-Key: ${PASSWORD}" \
  --data-binary "SELECT version() FORMAT JSON"
```

## Checking API Rate Limits

The ClickHouse Cloud API has rate limits. Check response headers:

```text
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1710000000
```

## Summary

The ClickHouse Cloud REST API covers the full lifecycle of service management - from provisioning and scaling to networking and user management. Use it to build Terraform providers, CI/CD workflows, and custom automation scripts that integrate ClickHouse Cloud into your infrastructure platform.
