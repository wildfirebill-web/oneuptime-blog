# How to Connect Portainer to an AWS EKS Cluster - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, AWS, EKS, Kubernetes, Cloud

Description: Connect Portainer to an Amazon EKS cluster for visual Kubernetes management using kubeconfig or the Portainer Agent.

## Introduction

Amazon EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes service. Connecting it to Portainer provides a visual interface for managing EKS workloads without requiring team members to learn `kubectl` and AWS IAM. This guide covers both the kubeconfig and agent methods.

## Prerequisites

- AWS CLI installed and configured with appropriate permissions
- An existing EKS cluster
- `kubectl` installed
- Portainer running and accessible

## Step 1: Get EKS Kubeconfig

```bash
# Update kubeconfig for your EKS cluster

aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-eks-cluster \
  --kubeconfig eks-portainer.kubeconfig

# Verify connectivity
kubectl --kubeconfig=eks-portainer.kubeconfig cluster-info
kubectl --kubeconfig=eks-portainer.kubeconfig get nodes
```

## Step 2: Create a Portainer Service Account in EKS

```bash
# Apply service account and RBAC
kubectl --kubeconfig=eks-portainer.kubeconfig apply -f - << 'EOF'
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
  name: portainer-sa-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: portainer-sa
    namespace: portainer
EOF

# Create a token for the service account
kubectl --kubeconfig=eks-portainer.kubeconfig \
  create token portainer-sa -n portainer --duration=8760h
```

## Step 3: Build Service Account Kubeconfig

```bash
# Get cluster endpoint
CLUSTER_ENDPOINT=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query "cluster.endpoint" \
  --output text)

# Get cluster CA certificate
CLUSTER_CA=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query "cluster.certificateAuthority.data" \
  --output text)

# Get service account token
SA_TOKEN=$(kubectl --kubeconfig=eks-portainer.kubeconfig \
  create token portainer-sa -n portainer --duration=8760h)

# Create kubeconfig for Portainer
cat > portainer-eks.kubeconfig << EOF
apiVersion: v1
kind: Config
clusters:
- name: eks-cluster
  cluster:
    server: $CLUSTER_ENDPOINT
    certificate-authority-data: $CLUSTER_CA
users:
- name: portainer-sa
  user:
    token: $SA_TOKEN
contexts:
- name: portainer-eks
  context:
    cluster: eks-cluster
    user: portainer-sa
current-context: portainer-eks
EOF

# Verify the service account kubeconfig works
kubectl --kubeconfig=portainer-eks.kubeconfig get namespaces
```

## Step 4: Import EKS into Portainer

### Via UI

1. Go to **Environments** → **Add environment** → **Kubernetes**
2. Select **Import**
3. Paste the content of `portainer-eks.kubeconfig`
4. Name: "EKS US-East Production"
5. Click **Connect**

### Via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

KUBECONFIG_B64=$(base64 -w 0 portainer-eks.kubeconfig)

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/import \
  -d "{
    \"Name\": \"EKS US-East Production\",
    \"KubeConfig\": \"${KUBECONFIG_B64}\"
  }"
```

## Method 2: Deploy Portainer Agent in EKS

For better Portainer integration:

```bash
# Install via Helm
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

helm install portainer-agent \
  --create-namespace \
  --namespace portainer \
  portainer/portainer-agent \
  --set ingress.enabled=false \
  --kubeconfig=eks-portainer.kubeconfig

# Get the agent service address
kubectl get svc -n portainer --kubeconfig=eks-portainer.kubeconfig

# Note the ClusterIP or create a LoadBalancer service for external access
```

## EKS-Specific Considerations

### IAM Authentication

EKS uses AWS IAM for authentication, but Portainer needs a plain kubeconfig with a static token (service account token). The `aws-iam-authenticator` method in the standard kubeconfig won't work with Portainer - always use a Kubernetes service account token.

### Private Cluster Access

If your EKS cluster is private (API endpoint not publicly accessible):

```bash
# Option 1: Run Portainer in the same VPC
# Option 2: Use Portainer Agent (inside cluster, no API access needed)
# Option 3: Use AWS VPN or Direct Connect to access the private endpoint
```

## Conclusion

Connecting EKS to Portainer provides a visual layer over Amazon's Kubernetes management. The key EKS-specific consideration is using service account tokens rather than IAM-based authentication. For private EKS clusters, the Portainer Agent method (installed inside the cluster) is the recommended approach since it doesn't require API server access from outside the VPC.
