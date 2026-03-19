# How to Create Clusters Using the Rancher API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, API, REST API, Cluster Management, Automation

Description: Learn how to provision and create Kubernetes clusters programmatically using the Rancher API with practical examples for RKE2, K3s, and imported clusters.

Creating clusters through the Rancher API is essential for infrastructure automation. Instead of clicking through the UI, you can define cluster configurations as code, version them, and provision clusters on demand. This guide walks you through creating different types of clusters using the Rancher API.

## Prerequisites

You need the following before you begin:

- A running Rancher server (v2.6+)
- An API token with cluster creation permissions
- curl and jq installed on your machine

Set up your environment variables:

```bash
export RANCHER_URL="https://rancher.example.com"
export RANCHER_TOKEN="token-xxxxx:yyyyyyyyyyyyyyyy"
```

## Creating a Custom Cluster (RKE2)

Custom clusters allow you to bring your own infrastructure. You register existing nodes with Rancher, and it installs the Kubernetes distribution on them.

### Step 1: Create the Cluster Resource

```bash
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "provisioning.cattle.io.cluster",
    "metadata": {
      "name": "my-rke2-cluster",
      "namespace": "fleet-default"
    },
    "spec": {
      "kubernetesVersion": "v1.28.9+rke2r1",
      "localClusterAuthEndpoint": {
        "enabled": true
      },
      "rkeConfig": {
        "machineGlobalConfig": {
          "cni": "calico",
          "disable-kube-proxy": false,
          "etcd-expose-metrics": false
        },
        "registries": {},
        "upgradeStrategy": {
          "controlPlaneConcurrency": "1",
          "controlPlaneDrainOptions": {},
          "workerConcurrency": "1",
          "workerDrainOptions": {}
        }
      }
    }
  }' \
  "${RANCHER_URL}/v1/provisioning.cattle.io.clusters"
```

### Step 2: Generate Registration Commands

After the cluster is created, generate the node registration command:

```bash
CLUSTER_ID="my-rke2-cluster"

# Get the cluster registration token
curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/clusterregistrationtokens?clusterId=${CLUSTER_ID}" | jq '.data[0] | {
    nodeCommand: .nodeCommand,
    insecureNodeCommand: .insecureCommand,
    manifestUrl: .manifestUrl
  }'
```

### Step 3: Register Nodes

Run the registration command on each node. For control plane nodes:

```bash
# On the node, run the command from the previous step with roles
curl -fL https://rancher.example.com/system-agent-install.sh | \
  sudo sh -s - \
  --server https://rancher.example.com \
  --label 'cattle.io/os=linux' \
  --token xxxxxxxxxx \
  --etcd --controlplane
```

For worker nodes:

```bash
curl -fL https://rancher.example.com/system-agent-install.sh | \
  sudo sh -s - \
  --server https://rancher.example.com \
  --label 'cattle.io/os=linux' \
  --token xxxxxxxxxx \
  --worker
```

## Creating a K3s Cluster

K3s clusters are lightweight and ideal for edge or development use cases:

```bash
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "provisioning.cattle.io.cluster",
    "metadata": {
      "name": "my-k3s-cluster",
      "namespace": "fleet-default"
    },
    "spec": {
      "kubernetesVersion": "v1.28.9+k3s1",
      "rkeConfig": {
        "machineGlobalConfig": {
          "cni": "flannel"
        },
        "upgradeStrategy": {
          "controlPlaneConcurrency": "1",
          "workerConcurrency": "1"
        }
      }
    }
  }' \
  "${RANCHER_URL}/v1/provisioning.cattle.io.clusters"
```

## Importing an Existing Cluster

If you already have a running Kubernetes cluster, you can import it into Rancher:

### Step 1: Create the Import Resource

```bash
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "provisioning.cattle.io.cluster",
    "metadata": {
      "name": "imported-cluster",
      "namespace": "fleet-default"
    },
    "spec": {}
  }' \
  "${RANCHER_URL}/v1/provisioning.cattle.io.clusters"
```

### Step 2: Get the Import Command

```bash
# Wait a few seconds for the registration token to be generated
sleep 5

curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/clusterregistrationtokens?clusterId=imported-cluster" | jq -r '.data[0].manifestUrl'
```

### Step 3: Apply the Manifest on the Target Cluster

Run this on the cluster you want to import:

```bash
kubectl apply -f https://rancher.example.com/v3/import/xxxxxxxxxx.yaml
```

For clusters with self-signed certificates:

```bash
curl --insecure -sfL https://rancher.example.com/v3/import/xxxxxxxxxx.yaml | kubectl apply -f -
```

## Creating Clusters with Node Pools (Cloud Providers)

For cloud-hosted clusters, you first need to create cloud credentials and node templates, then reference them in the cluster creation.

### Step 1: Create Cloud Credentials

```bash
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "cloudCredential",
    "name": "aws-credentials",
    "amazonec2credentialConfig": {
      "accessKey": "AKIAIOSFODNN7EXAMPLE",
      "secretKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
      "defaultRegion": "us-east-1"
    }
  }' \
  "${RANCHER_URL}/v3/cloudCredentials"
```

### Step 2: Create the Cluster with Machine Pools

```bash
CREDENTIAL_ID="cattle-global-data:cc-xxxxx"

curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "provisioning.cattle.io.cluster",
    "metadata": {
      "name": "aws-cluster",
      "namespace": "fleet-default"
    },
    "spec": {
      "cloudCredentialSecretName": "'"${CREDENTIAL_ID}"'",
      "kubernetesVersion": "v1.28.9+rke2r1",
      "rkeConfig": {
        "machinePools": [
          {
            "name": "control-plane",
            "controlPlaneRole": true,
            "etcdRole": true,
            "workerRole": false,
            "quantity": 3,
            "machineConfigRef": {
              "kind": "Amazonec2Config",
              "name": "cp-config"
            }
          },
          {
            "name": "workers",
            "controlPlaneRole": false,
            "etcdRole": false,
            "workerRole": true,
            "quantity": 3,
            "machineConfigRef": {
              "kind": "Amazonec2Config",
              "name": "worker-config"
            }
          }
        ]
      }
    }
  }' \
  "${RANCHER_URL}/v1/provisioning.cattle.io.clusters"
```

## Monitoring Cluster Creation Progress

After creating a cluster, monitor its provisioning status:

```bash
CLUSTER_NAME="my-rke2-cluster"

# Poll until the cluster is active
while true; do
  state=$(curl -s -k \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    "${RANCHER_URL}/v1/provisioning.cattle.io.clusters/fleet-default/${CLUSTER_NAME}" | jq -r '.status.conditions[] | select(.type=="Ready") | .status')

  echo "Cluster state: ${state}"

  if [ "$state" = "True" ]; then
    echo "Cluster is ready."
    break
  fi

  sleep 30
done
```

## Setting Cluster Labels and Annotations

Add metadata to your cluster for organizational purposes:

```bash
curl -s -k -X PUT \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "metadata": {
      "labels": {
        "environment": "production",
        "team": "platform",
        "cost-center": "engineering"
      },
      "annotations": {
        "description": "Production workload cluster",
        "owner": "platform-team@example.com"
      }
    }
  }' \
  "${RANCHER_URL}/v1/provisioning.cattle.io.clusters/fleet-default/my-rke2-cluster"
```

## Summary

The Rancher API supports creating custom clusters (RKE2 and K3s), importing existing clusters, and provisioning cloud-hosted clusters with machine pools. By defining cluster configurations as API calls, you can version your infrastructure definitions, integrate with CI/CD pipelines, and provision clusters on demand. Monitor the provisioning process through the API to build fully automated cluster lifecycle management.
