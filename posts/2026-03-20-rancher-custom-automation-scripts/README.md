# How to Create Custom Automation Scripts for Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, automation, scripts, bash, python, api

Description: A practical guide to writing custom automation scripts for Rancher using Bash and Python, covering common administrative tasks and workflow automation.

## Overview

While Rancher provides a comprehensive UI and Terraform support, there are many scenarios where custom automation scripts provide the most efficient solution: onboarding new clusters, bulk configuration changes, health checks, and integration with existing operational tooling. This guide covers patterns and examples for writing effective Rancher automation scripts.

## Authentication Patterns

### Bash Authentication Helper

```bash
#!/bin/bash
# rancher-auth.sh - Source this in your automation scripts

# Set these via environment variables or a secrets manager
RANCHER_URL="${RANCHER_URL:-https://rancher.example.com}"
RANCHER_TOKEN="${RANCHER_TOKEN}"  # Format: token-xxx:secret

# Validate token
validate_auth() {
  RESPONSE=$(curl -s -k \
    -o /dev/null \
    -w "%{http_code}" \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    "${RANCHER_URL}/v3/users/me")

  if [ "${RESPONSE}" != "200" ]; then
    echo "ERROR: Authentication failed (HTTP ${RESPONSE})"
    exit 1
  fi
  echo "Authentication successful"
}

# Generic API call wrapper with error handling
rancher_api() {
  local METHOD="$1"
  local ENDPOINT="$2"
  local DATA="$3"

  curl -s -k \
    -X "${METHOD}" \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    -H "Content-Type: application/json" \
    ${DATA:+-d "${DATA}"} \
    "${RANCHER_URL}${ENDPOINT}"
}
```

### Python Authentication Client

```python
#!/usr/bin/env python3
# rancher_client.py - Reusable Rancher API client

import os
import requests
import json
from typing import Optional, Dict, Any

class RancherClient:
    def __init__(self, url: str = None, token: str = None):
        self.url = url or os.environ.get('RANCHER_URL')
        self.token = token or os.environ.get('RANCHER_TOKEN')
        self.session = requests.Session()
        self.session.verify = False   # Set to True in production with proper CA
        self.session.headers.update({
            'Authorization': f'Bearer {self.token}',
            'Content-Type': 'application/json'
        })

    def get(self, endpoint: str, params: Dict = None) -> Dict:
        """GET request to Rancher API"""
        resp = self.session.get(f"{self.url}{endpoint}", params=params)
        resp.raise_for_status()
        return resp.json()

    def post(self, endpoint: str, data: Dict) -> Dict:
        """POST request to Rancher API"""
        resp = self.session.post(f"{self.url}{endpoint}", json=data)
        resp.raise_for_status()
        return resp.json()

    def put(self, endpoint: str, data: Dict) -> Dict:
        """PUT request to Rancher API"""
        resp = self.session.put(f"{self.url}{endpoint}", json=data)
        resp.raise_for_status()
        return resp.json()

    def delete(self, endpoint: str) -> None:
        """DELETE request to Rancher API"""
        resp = self.session.delete(f"{self.url}{endpoint}")
        resp.raise_for_status()

    def get_clusters(self) -> list:
        """Get all clusters"""
        return self.get('/v3/clusters').get('data', [])

    def get_cluster_health(self, cluster_id: str) -> str:
        """Get cluster health status"""
        cluster = self.get(f'/v3/clusters/{cluster_id}')
        return cluster.get('state', 'unknown')
```

## Common Automation Scripts

### Cluster Health Check Script

```python
#!/usr/bin/env python3
# cluster-health-check.py
from rancher_client import RancherClient
import sys

def check_all_clusters():
    client = RancherClient()
    clusters = client.get_clusters()

    unhealthy = []
    for cluster in clusters:
        state = cluster.get('state', 'unknown')
        name = cluster.get('name', 'unknown')
        cluster_id = cluster.get('id', 'unknown')

        if state != 'active':
            unhealthy.append({
                'name': name,
                'id': cluster_id,
                'state': state
            })
            print(f"UNHEALTHY: {name} ({cluster_id}) - state: {state}")
        else:
            print(f"OK: {name} ({cluster_id}) - active")

    if unhealthy:
        print(f"\nFound {len(unhealthy)} unhealthy cluster(s)")
        sys.exit(1)
    else:
        print(f"\nAll {len(clusters)} clusters are healthy")

if __name__ == '__main__':
    check_all_clusters()
```

### Bulk Namespace Creation Script

```bash
#!/bin/bash
# bulk-namespaces.sh - Create namespaces across multiple clusters

CLUSTERS_FILE="clusters.txt"
NAMESPACES=("monitoring" "logging" "security" "ingress")

while IFS= read -r cluster_id; do
  echo "Processing cluster: ${cluster_id}"

  for ns in "${NAMESPACES[@]}"; do
    # Create namespace via kubectl using cluster-specific kubeconfig
    kubectl --kubeconfig=<(rancher_api GET "/v3/clusters/${cluster_id}?action=generateKubeconfig" | jq -r '.config') \
      create namespace "${ns}" \
      --dry-run=client -o yaml | kubectl apply -f - 2>/dev/null

    echo "  Namespace ${ns} created/verified in ${cluster_id}"
  done
done < "${CLUSTERS_FILE}"
```

### Deploy App Across All Clusters

```python
#!/usr/bin/env python3
# deploy-app-all-clusters.py
# Deploys a Helm chart to all clusters with a specific label

from rancher_client import RancherClient
import json

def deploy_app_to_clusters(label_key: str, label_value: str, app_config: dict):
    client = RancherClient()
    clusters = client.get_clusters()

    # Filter clusters by label
    target_clusters = [
        c for c in clusters
        if c.get('labels', {}).get(label_key) == label_value
        and c.get('state') == 'active'
    ]

    print(f"Deploying to {len(target_clusters)} clusters")

    for cluster in target_clusters:
        cluster_id = cluster['id']
        cluster_name = cluster['name']

        try:
            # Get the project ID for the cluster
            projects = client.get(f'/v3/projects?clusterId={cluster_id}').get('data', [])
            default_project = next((p for p in projects if p['name'] == 'Default'), None)

            if not default_project:
                print(f"  SKIP: {cluster_name} - no Default project found")
                continue

            # Deploy the Helm chart
            app_data = {
                **app_config,
                'projectId': default_project['id'],
            }
            result = client.post('/v3/apps', app_data)
            print(f"  Deployed to {cluster_name}: {result.get('id')}")
        except Exception as e:
            print(f"  FAILED: {cluster_name} - {str(e)}")


# Example usage
app_config = {
    'name': 'monitoring',
    'targetNamespace': 'cattle-monitoring-system',
    'externalId': 'catalog://?catalog=rancher-charts&type=clusterCatalog&template=rancher-monitoring&version=103.0.0',
    'answers': {
        'prometheus.prometheusSpec.retention': '30d',
    }
}

deploy_app_to_clusters('env', 'production', app_config)
```

### Automated Cluster Labeling

```bash
#!/bin/bash
# label-clusters.sh - Automatically label clusters based on naming convention

# Example: cluster names like "prod-us-east-rke2" get labeled accordingly
source rancher-auth.sh

CLUSTERS=$(rancher_api GET /v3/clusters | jq -r '.data[] | [.id, .name] | @tsv')

while IFS=$'\t' read -r cluster_id cluster_name; do
  # Extract environment from cluster name
  if [[ "${cluster_name}" == prod-* ]]; then
    ENV="production"
  elif [[ "${cluster_name}" == staging-* ]]; then
    ENV="staging"
  else
    ENV="development"
  fi

  # Extract region from cluster name
  if [[ "${cluster_name}" == *us-east* ]]; then
    REGION="us-east"
  elif [[ "${cluster_name}" == *us-west* ]]; then
    REGION="us-west"
  elif [[ "${cluster_name}" == *eu-west* ]]; then
    REGION="eu-west"
  fi

  # Update cluster labels
  rancher_api PUT "/v3/clusters/${cluster_id}" \
    "{\"labels\": {\"env\": \"${ENV}\", \"region\": \"${REGION}\"}}" \
    > /dev/null

  echo "Labeled ${cluster_name}: env=${ENV}, region=${REGION}"
done <<< "${CLUSTERS}"
```

## Scheduling Scripts with CronJobs

```yaml
# Kubernetes CronJob to run health check daily
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rancher-health-check
  namespace: cattle-system
spec:
  schedule: "0 8 * * *"    # Daily at 8 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: health-checker
              image: registry.example.com/rancher-scripts:latest
              env:
                - name: RANCHER_URL
                  value: "https://rancher.example.com"
                - name: RANCHER_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: rancher-automation-token
                      key: token
                - name: SLACK_WEBHOOK
                  valueFrom:
                    secretKeyRef:
                      name: notifications
                      key: slack-webhook
              command: ["python3", "/scripts/cluster-health-check.py"]
          restartPolicy: OnFailure
```

## Conclusion

Custom automation scripts for Rancher enable you to integrate Rancher operations into your existing workflows, automate repetitive tasks, and respond to events programmatically. Using a shared authentication helper and API client library ensures consistency across scripts. Store all automation scripts in version control, manage credentials in Kubernetes Secrets or Vault, and schedule recurring tasks as Kubernetes CronJobs for reliability. Always test scripts against non-production environments before running them on production clusters.
