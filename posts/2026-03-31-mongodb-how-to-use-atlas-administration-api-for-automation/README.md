# How to Use Atlas Administration API for Automation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Administration Api, Automation, REST API

Description: Automate MongoDB Atlas operations using the Administration API to manage clusters, users, backups, and alerts programmatically in scripts and pipelines.

---

## What Is the Atlas Administration API?

The Atlas Administration API is a REST API that exposes full control over MongoDB Atlas resources. Everything available in the Atlas UI is also available via the API, making it possible to automate infrastructure management, integrate Atlas into CI/CD pipelines, and build custom tooling.

Base URL: `https://cloud.mongodb.com/api/atlas/v1.0`

## Authentication with Digest Auth

The API uses HTTP Digest Authentication with API keys:

```bash
curl --user "PUBLIC_KEY:PRIVATE_KEY" --digest \
  --request GET \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups"
```

### Create an API Key

1. In Atlas, go to **Access Manager > Organization Access**
2. Click **Create API Key**
3. Assign the key a role (e.g., **Organization Owner** or **Project Data Access Admin**)
4. Note the public and private key (private key shown only once)
5. Whitelist the IP addresses allowed to use the key

### Project-Level API Keys

```bash
curl --user "publicKey:privateKey" --digest \
  --request POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/apiKeys" \
  --header "Content-Type: application/json" \
  --data '{
    "desc": "CI/CD pipeline key",
    "roles": ["GROUP_CLUSTER_MANAGER", "GROUP_DATA_ACCESS_ADMIN"]
  }'
```

## Core API Operations

### List Projects

```bash
curl --user "publicKey:privateKey" --digest \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups"
```

### Create a Cluster

```bash
curl --user "publicKey:privateKey" --digest \
  --request POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/clusters" \
  --header "Content-Type: application/json" \
  --data '{
    "name": "myCluster",
    "clusterType": "REPLICASET",
    "providerSettings": {
      "providerName": "AWS",
      "regionName": "US_EAST_1",
      "instanceSizeName": "M30"
    },
    "mongoDBMajorVersion": "7.0",
    "diskSizeGB": 100
  }'
```

### Monitor Cluster Status Until Ready

```bash
#!/bin/bash
CLUSTER_NAME="myCluster"
GROUP_ID="your-group-id"

while true; do
  STATE=$(curl -s --user "publicKey:privateKey" --digest \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${GROUP_ID}/clusters/${CLUSTER_NAME}" \
    | python3 -c "import sys, json; print(json.load(sys.stdin)['stateName'])")

  echo "Cluster state: $STATE"

  if [ "$STATE" = "IDLE" ]; then
    echo "Cluster is ready"
    break
  fi

  sleep 30
done
```

### Update Cluster Tier

```bash
curl --user "publicKey:privateKey" --digest \
  --request PATCH \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/clusters/{clusterName}" \
  --header "Content-Type: application/json" \
  --data '{
    "providerSettings": {
      "instanceSizeName": "M60"
    }
  }'
```

## User Management

```bash
# Create a database user
curl --user "publicKey:privateKey" --digest \
  --request POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/databaseUsers" \
  --header "Content-Type: application/json" \
  --data '{
    "databaseName": "admin",
    "roles": [{ "databaseName": "appdb", "roleName": "readWrite" }],
    "username": "appUser",
    "password": "SecurePass123!"
  }'

# Update a user role
curl --user "publicKey:privateKey" --digest \
  --request PATCH \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/databaseUsers/admin/appUser" \
  --header "Content-Type: application/json" \
  --data '{
    "roles": [
      { "databaseName": "appdb", "roleName": "readWrite" },
      { "databaseName": "logsdb", "roleName": "read" }
    ]
  }'
```

## Alerts Management

```bash
curl --user "publicKey:privateKey" --digest \
  --request POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/alertConfigs" \
  --header "Content-Type: application/json" \
  --data '{
    "eventTypeName": "REPLICATION_OPLOG_WINDOW_RUNNING_OUT",
    "enabled": true,
    "threshold": {
      "operator": "LESS_THAN",
      "units": "HOURS",
      "threshold": 2
    },
    "notifications": [
      {
        "typeName": "EMAIL",
        "emailAddress": "ops@example.com",
        "intervalMin": 5,
        "delayMin": 0
      }
    ]
  }'
```

## Python Client Example

```python
import requests
from requests.auth import HTTPDigestAuth
import os

class AtlasAPI:
    BASE_URL = "https://cloud.mongodb.com/api/atlas/v1.0"

    def __init__(self, public_key, private_key, project_id):
        self.auth = HTTPDigestAuth(public_key, private_key)
        self.project_id = project_id

    def get_clusters(self):
        resp = requests.get(
            f"{self.BASE_URL}/groups/{self.project_id}/clusters",
            auth=self.auth
        )
        resp.raise_for_status()
        return resp.json()["results"]

    def create_cluster(self, name, tier="M30", region="US_EAST_1"):
        payload = {
            "name": name,
            "clusterType": "REPLICASET",
            "providerSettings": {
                "providerName": "AWS",
                "regionName": region,
                "instanceSizeName": tier
            },
            "mongoDBMajorVersion": "7.0"
        }
        resp = requests.post(
            f"{self.BASE_URL}/groups/{self.project_id}/clusters",
            auth=self.auth,
            json=payload
        )
        resp.raise_for_status()
        return resp.json()

    def pause_cluster(self, cluster_name):
        resp = requests.patch(
            f"{self.BASE_URL}/groups/{self.project_id}/clusters/{cluster_name}",
            auth=self.auth,
            json={"paused": True}
        )
        resp.raise_for_status()
        return resp.json()

# Usage
api = AtlasAPI(
    os.environ["ATLAS_PUBLIC_KEY"],
    os.environ["ATLAS_PRIVATE_KEY"],
    os.environ["ATLAS_PROJECT_ID"]
)

clusters = api.get_clusters()
for c in clusters:
    print(f"{c['name']} - {c['stateName']} - {c['providerSettings']['instanceSizeName']}")
```

## Rate Limits and Pagination

The API enforces rate limits. For paginated responses:

```bash
# Get page 2 with 100 items per page
curl --user "publicKey:privateKey" --digest \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/clusters?pageNum=2&itemsPerPage=100"
```

Use the `totalCount` field to determine how many pages to fetch.

## Summary

The Atlas Administration API provides REST endpoints for every Atlas operation, authenticated with HTTP Digest Auth using API keys. Use it to automate cluster provisioning, user management, backup operations, and alert configuration in CI/CD pipelines or custom tooling. Wrap API calls in a client class for Python, JavaScript, or any language, and handle pagination for list endpoints that return large result sets.
