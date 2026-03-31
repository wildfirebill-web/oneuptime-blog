# How to Import an Existing Kubernetes Cluster into Portainer via Kubeconfig - K8s

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, kubeconfig, Import, Environment

Description: Import an existing Kubernetes cluster into Portainer using a kubeconfig file for immediate visual management without deploying an agent.

## Introduction

The kubeconfig import method lets you connect any Kubernetes cluster to Portainer without installing an agent. If you already have a `~/.kube/config` for a cluster, you can paste it directly into Portainer. This is the fastest way to add a Kubernetes environment, especially for managed clusters like EKS, AKS, or GKE.

## Prerequisites

- Portainer running
- A kubeconfig file with cluster-admin or sufficient permissions
- Network access from Portainer to the Kubernetes API server

## Step 1: Prepare the Kubeconfig

```bash
# View your current kubeconfig

kubectl config view --raw

# Get kubeconfig for a specific context
kubectl config view --raw --minify --context=my-cluster

# If you have multiple clusters, extract just the one you need
kubectl config view --raw --minify \
  --context=arn:aws:eks:us-east-1:123456:cluster/my-cluster > my-cluster-kubeconfig.yaml

# Verify the kubeconfig works
kubectl --kubeconfig=my-cluster-kubeconfig.yaml cluster-info
```

## Step 2: Add Kubernetes Environment via Kubeconfig Import

### Via Web UI

1. Go to **Environments** → **Add environment**
2. Select **Kubernetes**
3. Select **Import** (or "From kubeconfig")
4. Paste the kubeconfig content or upload the file
5. Give the environment a name
6. Click **Connect**

### Via Portainer API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Read and encode the kubeconfig
KUBECONFIG_CONTENT=$(cat my-cluster-kubeconfig.yaml | base64 | tr -d '\n')

# Import the cluster
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/import \
  -d "{
    \"Name\": \"Production K8s Cluster\",
    \"KubeConfig\": \"${KUBECONFIG_CONTENT}\"
  }"
```

## Step 3: Create a Dedicated Service Account for Portainer

For security, don't use admin credentials. Create a dedicated service account:

```yaml
# portainer-k8s-sa.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: portainer
  namespace: portainer

---
apiVersion: v1
kind: Namespace
metadata:
  name: portainer

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portainer
rules:
  # Full access for initial setup
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portainer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: portainer
subjects:
  - kind: ServiceAccount
    name: portainer
    namespace: portainer
```

```bash
kubectl apply -f portainer-k8s-sa.yml

# Get the service account token (Kubernetes 1.24+)
kubectl create token portainer -n portainer --duration=8760h

# For older K8s versions, get the automatically-created token
SA_SECRET=$(kubectl get sa portainer -n portainer -o jsonpath='{.secrets[0].name}')
kubectl get secret $SA_SECRET -n portainer -o jsonpath='{.data.token}' | base64 -d

# Get cluster CA
kubectl get secret $SA_SECRET -n portainer -o jsonpath='{.data.ca\.crt}'
```

```bash
# Build a kubeconfig for the service account
CLUSTER_SERVER=$(kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.server}')
SA_TOKEN=$(kubectl create token portainer -n portainer --duration=8760h)
CA_DATA=$(kubectl get secret $SA_SECRET -n portainer -o jsonpath='{.data.ca\.crt}')

cat > portainer-sa-kubeconfig.yaml << EOF
apiVersion: v1
kind: Config
clusters:
- name: my-cluster
  cluster:
    server: $CLUSTER_SERVER
    certificate-authority-data: $CA_DATA
users:
- name: portainer-sa
  user:
    token: $SA_TOKEN
contexts:
- name: portainer-context
  context:
    cluster: my-cluster
    user: portainer-sa
current-context: portainer-context
EOF
```

## Verifying the Import

```bash
# Check the imported environment appears
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/endpoints \
  | python3 -c "
import sys, json
for env in json.load(sys.stdin):
    print(f'ID={env[\"Id\"]} Name={env[\"Name\"]} Type={env[\"Type\"]} Status={env[\"Status\"]}')
"

# Test K8s API via Portainer
ENDPOINT_ID=6
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}/kubernetes/namespaces" \
  | python3 -c "import sys,json; [print(ns['Name']) for ns in json.load(sys.stdin)]"
```

## Conclusion

Kubeconfig import is the fastest way to add an existing Kubernetes cluster to Portainer. It works with any Kubernetes distribution - EKS, AKS, GKE, on-premises, or minikube. For production use, create a dedicated service account with appropriate permissions rather than using admin credentials. The agent-based method provides deeper integration (namespace-level RBAC, resource quotas), but kubeconfig import is perfect for quickly getting a cluster under Portainer management.
