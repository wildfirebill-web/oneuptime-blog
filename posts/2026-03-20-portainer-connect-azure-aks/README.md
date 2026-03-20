# How to Connect Portainer to an Azure AKS Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, AKS, Kubernetes, Cloud

Description: Connect Portainer to an Azure Kubernetes Service (AKS) cluster for visual management of Azure-hosted Kubernetes workloads.

## Introduction

Azure Kubernetes Service (AKS) is Microsoft's managed Kubernetes service. Connecting AKS to Portainer gives teams a visual interface for deploying and managing applications without requiring Azure portal or kubectl knowledge. This guide covers connecting AKS via kubeconfig and the Portainer Agent.

## Prerequisites

- Azure CLI installed and authenticated (`az login`)
- An existing AKS cluster
- Portainer running and accessible

## Step 1: Get AKS Credentials

```bash
# Get the kubeconfig for your AKS cluster
az aks get-credentials \
  --resource-group my-resource-group \
  --name my-aks-cluster \
  --file aks-portainer.kubeconfig

# Verify connectivity
kubectl --kubeconfig=aks-portainer.kubeconfig cluster-info
kubectl --kubeconfig=aks-portainer.kubeconfig get nodes
```

## Step 2: Create a Service Account for Portainer

```bash
# Create namespace and service account
kubectl --kubeconfig=aks-portainer.kubeconfig apply -f - << 'EOF'
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

## Step 3: Build a Static Kubeconfig

```bash
# Get cluster server address
CLUSTER_SERVER=$(kubectl --kubeconfig=aks-portainer.kubeconfig \
  config view --raw --minify -o jsonpath='{.clusters[0].cluster.server}')

# Get cluster CA certificate
CLUSTER_CA=$(kubectl --kubeconfig=aks-portainer.kubeconfig \
  config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')

# Create a service account token
SA_TOKEN=$(kubectl --kubeconfig=aks-portainer.kubeconfig \
  create token portainer-sa -n portainer --duration=8760h)

# Build the kubeconfig
cat > portainer-aks.kubeconfig << EOF
apiVersion: v1
kind: Config
clusters:
- name: aks-cluster
  cluster:
    server: $CLUSTER_SERVER
    certificate-authority-data: $CLUSTER_CA
users:
- name: portainer-sa
  user:
    token: $SA_TOKEN
contexts:
- name: portainer-aks
  context:
    cluster: aks-cluster
    user: portainer-sa
current-context: portainer-aks
EOF

# Test the kubeconfig
kubectl --kubeconfig=portainer-aks.kubeconfig get nodes
```

## Step 4: Import AKS into Portainer

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

KUBECONFIG_B64=$(base64 -w 0 portainer-aks.kubeconfig)

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/import \
  -d "{
    \"Name\": \"AKS West Europe Production\",
    \"KubeConfig\": \"${KUBECONFIG_B64}\"
  }"
```

## Method 2: Deploy Portainer Agent in AKS

```bash
# Install Portainer Agent via Helm
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

helm install portainer-agent \
  --create-namespace \
  -n portainer \
  portainer/portainer-agent \
  --kubeconfig=aks-portainer.kubeconfig

# Get the agent service endpoint
kubectl get svc -n portainer --kubeconfig=aks-portainer.kubeconfig
```

For AKS with Azure Load Balancer, create a LoadBalancer service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: portainer-agent-lb
  namespace: portainer
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"  # Internal LB
spec:
  type: LoadBalancer
  selector:
    app: portainer-agent
  ports:
    - port: 9001
      targetPort: 9001
```

## AKS-Specific Considerations

### Azure AD Integration

AKS supports Azure AD for authentication. Standard kubeconfig from `az aks get-credentials` uses Azure AD tokens which expire. For Portainer, always use a static service account token.

### Private AKS Clusters

For private AKS (API server not public):
```bash
# Use Azure Private Link or deploy Portainer inside AKS
# Portainer Agent inside AKS doesn't need API server access

# Or use Portainer Edge Agent for async management
```

### AKS Node Pools

Portainer can view nodes across multiple node pools. Check node labels to understand pool membership:
```bash
kubectl get nodes --show-labels --kubeconfig=aks-portainer.kubeconfig | grep agentpool
```

## Conclusion

AKS clusters integrate seamlessly with Portainer via kubeconfig import. The key is using a static service account token rather than Azure AD token-based authentication. For private AKS clusters or environments where Portainer runs in a different network, the Portainer Agent deployed inside AKS is the recommended approach — it communicates outbound from the cluster to Portainer, eliminating the need for inbound network access to the API server.
