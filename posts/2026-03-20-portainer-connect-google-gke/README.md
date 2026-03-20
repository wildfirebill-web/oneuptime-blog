# How to Connect Portainer to a Google GKE Cluster - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Google Cloud, GKE, Kubernetes, Cloud

Description: Connect Portainer to a Google Kubernetes Engine (GKE) cluster for visual management of GCP-hosted Kubernetes workloads.

## Introduction

Google Kubernetes Engine (GKE) is Google Cloud's managed Kubernetes service. Connecting GKE to Portainer provides a visual management layer for your GCP Kubernetes infrastructure. This guide covers kubeconfig and agent-based connection methods for GKE.

## Prerequisites

- Google Cloud SDK (`gcloud`) installed and authenticated
- An existing GKE cluster
- Portainer running and accessible

## Step 1: Get GKE Credentials

```bash
# Authenticate if needed

gcloud auth login

# Get credentials for your GKE cluster
gcloud container clusters get-credentials my-gke-cluster \
  --region us-central1 \
  --project my-gcp-project \
  --kubeconfig gke-portainer.kubeconfig

# Verify access
kubectl --kubeconfig=gke-portainer.kubeconfig cluster-info
kubectl --kubeconfig=gke-portainer.kubeconfig get nodes
```

## Step 2: Create a Service Account for Portainer

```bash
kubectl --kubeconfig=gke-portainer.kubeconfig apply -f - << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: portainer

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: portainer-sa
  namespace: portainer

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portainer-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: portainer-sa
    namespace: portainer
EOF
```

## Step 3: Create a Static Token Kubeconfig

GKE's default kubeconfig uses gcloud token refresh, which won't work with Portainer. Create a static service account token:

```bash
# Create a service account token (Kubernetes 1.24+)
SA_TOKEN=$(kubectl --kubeconfig=gke-portainer.kubeconfig \
  create token portainer-sa -n portainer --duration=8760h)

# Get cluster server
CLUSTER_SERVER=$(kubectl --kubeconfig=gke-portainer.kubeconfig \
  config view --raw --minify -o jsonpath='{.clusters[0].cluster.server}')

# Get cluster CA certificate
CLUSTER_CA=$(kubectl --kubeconfig=gke-portainer.kubeconfig \
  config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')

# Build static kubeconfig
cat > portainer-gke.kubeconfig << EOF
apiVersion: v1
kind: Config
clusters:
- name: gke-cluster
  cluster:
    server: $CLUSTER_SERVER
    certificate-authority-data: $CLUSTER_CA
users:
- name: portainer-sa
  user:
    token: $SA_TOKEN
contexts:
- name: portainer-gke
  context:
    cluster: gke-cluster
    user: portainer-sa
current-context: portainer-gke
EOF

# Verify static kubeconfig works
kubectl --kubeconfig=portainer-gke.kubeconfig get namespaces
```

## Step 4: Import GKE into Portainer

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

KUBECONFIG_B64=$(base64 -w 0 portainer-gke.kubeconfig)

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/import \
  -d "{
    \"Name\": \"GKE US-Central Production\",
    \"KubeConfig\": \"${KUBECONFIG_B64}\"
  }"
```

## Method 2: Portainer Agent in GKE

```bash
# Add Portainer Helm repo
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

# Install agent in GKE
helm install portainer-agent \
  --create-namespace \
  -n portainer \
  portainer/portainer-agent \
  --kubeconfig=gke-portainer.kubeconfig

# For external Portainer access, expose agent as LoadBalancer
kubectl --kubeconfig=gke-portainer.kubeconfig \
  patch svc portainer-agent -n portainer \
  -p '{"spec":{"type":"LoadBalancer"}}'

# Get external IP (takes 1-2 minutes)
kubectl --kubeconfig=gke-portainer.kubeconfig \
  get svc portainer-agent -n portainer -w
```

## GKE-Specific Considerations

### GKE Autopilot

GKE Autopilot enforces strict security policies. DaemonSets may be restricted. Use the Portainer Agent Deployment (not DaemonSet) for Autopilot:

```bash
helm install portainer-agent \
  -n portainer \
  portainer/portainer-agent \
  --set deploymentKind=Deployment \
  --kubeconfig=gke-portainer.kubeconfig
```

### Private GKE Clusters

Private GKE clusters don't have a public API endpoint:
```bash
# Add master-authorized networks to allow Portainer to connect
gcloud container clusters update my-gke-cluster \
  --enable-master-authorized-networks \
  --master-authorized-networks PORTAINER_IP/32 \
  --region us-central1
```

Or use the Portainer Agent which doesn't require API access from outside.

### Workload Identity

If using GKE Workload Identity, ensure the Portainer service account has the necessary IAM bindings for any GCP APIs it needs to access.

## Conclusion

GKE integration with Portainer requires a static service account token rather than gcloud's OAuth-based token refresh. Once configured, the Portainer interface provides the same visual management capabilities for GKE as any other Kubernetes cluster. For private GKE clusters, the Portainer Agent is the cleanest solution as it initiates the connection from inside the cluster.
