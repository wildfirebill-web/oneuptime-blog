# How to Import an Existing Kubernetes Cluster into Portainer via Kubeconfig

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Kubeconfig, Import, DevOps

Description: Learn how to import an existing Kubernetes cluster into Portainer using a kubeconfig file for immediate management access.

---

If you already have a Kubernetes cluster with a kubeconfig file, you can import it directly into Portainer without installing the Portainer agent. This is the fastest way to get cluster visibility.

## Prerequisites

- A valid kubeconfig file for the target cluster
- The API server URL in the kubeconfig must be accessible from the Portainer server
- Portainer Business Edition (kubeconfig import is a BE feature)

## Step 1: Prepare the Kubeconfig File

Ensure the kubeconfig has the correct context and server URL:

```bash
# View your kubeconfig
cat ~/.kube/config

# Set the correct context
kubectl config use-context my-cluster-context

# Verify connectivity
kubectl cluster-info
kubectl get nodes
```

## Step 2: Import via the Portainer UI

1. Navigate to **Environments > Add environment**
2. Select **Kubernetes**
3. Choose **Import existing cluster**
4. Upload your kubeconfig file or paste its contents
5. Give the environment a name
6. Click **Connect**

## Step 3: Import via the API

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Base64-encode the kubeconfig
KUBECONFIG_B64=$(cat ~/.kube/config | base64 -w 0)

# Import the cluster via API
curl -X POST \
  https://localhost:9443/api/endpoints \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"Name\": \"my-k8s-cluster\",
    \"EndpointCreationType\": 5,
    \"KubernetesSettings\": {
      \"Kubeconfig\": \"$KUBECONFIG_B64\"
    }
  }" \
  --insecure
```

## Create a Restricted Service Account

For security, don't use the admin kubeconfig. Create a dedicated service account:

```bash
# Create a service account for Portainer
kubectl create serviceaccount portainer-svc -n kube-system

# Bind cluster-admin role (adjust permissions as needed)
kubectl create clusterrolebinding portainer-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:portainer-svc

# Get the token (Kubernetes 1.24+)
kubectl create token portainer-svc -n kube-system --duration=87600h > /tmp/portainer-token

# Create kubeconfig using the service account token
kubectl config set-credentials portainer-user \
  --token=$(cat /tmp/portainer-token)
kubectl config set-context portainer-context \
  --cluster=$(kubectl config current-context) \
  --user=portainer-user
kubectl config use-context portainer-context
kubectl config view --minify --flatten > /tmp/portainer-kubeconfig.yaml
```

## Verify the Import

After importing, the cluster should appear in **Environments** as online. Navigate to it and verify you can see nodes and workloads.

---

*Monitor your imported Kubernetes cluster with [OneUptime](https://oneuptime.com) infrastructure monitoring.*
