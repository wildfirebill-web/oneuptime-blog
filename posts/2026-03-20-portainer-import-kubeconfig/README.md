# How to Import a Kubernetes Cluster Using Kubeconfig in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, kubeconfig, DevOps

Description: Learn how to import an existing Kubernetes cluster into Portainer using a kubeconfig file for quick cluster registration.

## Introduction

Importing a Kubernetes cluster via kubeconfig is the quickest way to add an existing cluster to Portainer. Unlike the agent-based approach, kubeconfig import requires no additional installation on the target cluster - Portainer uses the kubeconfig credentials to talk directly to the Kubernetes API server. This guide covers the complete kubeconfig import process.

## Prerequisites

- Portainer CE or BE running
- A valid kubeconfig file for your target cluster
- The Kubernetes API server must be reachable from Portainer
- Admin access to Portainer

## Understanding Kubeconfig

A kubeconfig file contains:
- Cluster API server URL
- TLS certificates or CA certificate
- User credentials (certificates, tokens, or command-based auth)
- Context combining cluster + user + namespace

```yaml
# Example kubeconfig structure

apiVersion: v1
kind: Config
clusters:
  - cluster:
      certificate-authority-data: BASE64_CA_CERT
      server: https://my-cluster.example.com:6443
    name: my-cluster

users:
  - name: admin
    user:
      client-certificate-data: BASE64_CERT
      client-key-data: BASE64_KEY

contexts:
  - context:
      cluster: my-cluster
      user: admin
      namespace: default
    name: my-cluster-context

current-context: my-cluster-context
```

## Step 1: Prepare the Kubeconfig

Before importing, ensure the kubeconfig is properly prepared:

```bash
# View your current kubeconfig
kubectl config view

# Export a specific context as a standalone kubeconfig
kubectl config view --context=my-cluster-context --minify --flatten \
  > portainer-kubeconfig.yaml

# Verify the context works
KUBECONFIG=portainer-kubeconfig.yaml kubectl cluster-info
```

## Step 2: Create a Dedicated Service Account (Recommended)

Using a dedicated service account is more secure than using admin credentials:

```bash
# Create namespace and service account
kubectl create namespace portainer

cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: portainer-service-account
  namespace: portainer
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portainer-cluster-role
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portainer-cluster-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: portainer-cluster-role
subjects:
  - kind: ServiceAccount
    name: portainer-service-account
    namespace: portainer
EOF

# Create a long-lived token for the service account
kubectl create token portainer-service-account \
  --namespace portainer \
  --duration=87600h \
  > /tmp/portainer-token.txt

TOKEN=$(cat /tmp/portainer-token.txt)
```

## Step 3: Create a Kubeconfig Using the Service Account Token

```bash
# Get cluster information
CLUSTER_NAME=$(kubectl config current-context)
CLUSTER_SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
CLUSTER_CA=$(kubectl config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')

# Create a kubeconfig for the service account
cat > portainer-sa-kubeconfig.yaml << EOF
apiVersion: v1
kind: Config
clusters:
  - cluster:
      certificate-authority-data: ${CLUSTER_CA}
      server: ${CLUSTER_SERVER}
    name: ${CLUSTER_NAME}
users:
  - name: portainer-service-account
    user:
      token: ${TOKEN}
contexts:
  - context:
      cluster: ${CLUSTER_NAME}
      user: portainer-service-account
    name: portainer-context
current-context: portainer-context
EOF

# Test the new kubeconfig
KUBECONFIG=portainer-sa-kubeconfig.yaml kubectl get nodes
```

## Step 4: Import the Kubeconfig in Portainer

1. Log in to Portainer as admin
2. Go to **Environments** (Home → gear icon)
3. Click **+ Add environment**
4. Select **Kubernetes**
5. Select **Import an existing Kubernetes environment using a kubeconfig file** (or paste kubeconfig)

### Option A: File Upload

1. Click **Browse** and select `portainer-sa-kubeconfig.yaml`
2. Portainer reads the file and extracts the cluster information

### Option B: Paste Kubeconfig

1. Click **Paste as text**
2. Paste the kubeconfig content
3. Portainer parses the YAML

## Step 5: Configure Environment Details

After uploading:

```text
Environment name:    production-cluster
Namespace:          (leave blank for all namespaces)
```

Click **Create environment**.

## Step 6: Handle Cloud Provider Authentication

### AWS EKS

EKS kubeconfigs use the `aws eks get-token` command for authentication:

```yaml
# EKS kubeconfig uses exec provider (not compatible with direct import)
users:
  - name: admin
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1alpha1
        command: aws
        args:
          - eks
          - get-token
          - --cluster-name
          - my-eks-cluster
```

For Portainer, convert EKS auth to a static token:

```bash
# Get a static token for EKS
TOKEN=$(aws eks get-token --cluster-name my-eks-cluster --query 'status.token' --output text)
```

Then use the static token in the kubeconfig.

### Azure AKS

```bash
# Get AKS credentials
az aks get-credentials --resource-group myrg --name my-aks --admin
# The --admin flag creates credentials with admin client certificate
```

### GKE

```bash
# Get GKE credentials
gcloud container clusters get-credentials my-gke-cluster --region us-central1
```

GKE kubeconfigs use `gcloud` exec provider. For Portainer, create a service account with a static token.

## Step 7: Verify the Import

After importing:

1. The new cluster appears in **Environments** with status **Up**
2. Click on the cluster to access it
3. Navigate to **Nodes** to see cluster nodes
4. Check **Namespaces** to see existing namespaces

## Step 8: Update Expired Kubeconfig

If credentials expire:

1. Generate new credentials or a new service account token
2. In Portainer, go to **Environments → {cluster} → Edit**
3. Update the kubeconfig with fresh credentials

## Conclusion

Importing Kubernetes clusters via kubeconfig is the fastest way to get Portainer managing an existing cluster. For production use, create a dedicated service account with appropriate RBAC instead of using admin credentials. Handle cloud-provider-specific authentication (EKS, AKS, GKE) by converting to static service account tokens that Portainer can use directly.
