# How to Add a Kubernetes Environment to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Environments, Kubernetes Management

Description: Connect a Kubernetes cluster to Portainer for visual management of deployments, services, namespaces, and resources.

## Introduction

Portainer can manage Kubernetes clusters, providing a visual interface for deployments, services, configmaps, and other resources. You can connect Kubernetes via the Portainer Agent deployed in the cluster, or via kubeconfig for remote access. This guide covers both methods.

## Method 1: Deploy Portainer Agent in Kubernetes (Recommended)

### Step 1: Create Portainer Namespace and Service Account

```bash
# Create namespace
kubectl create namespace portainer

# Create service account and RBAC
kubectl apply -f - << 'EOF'
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

### Step 2: Deploy Portainer Agent via Helm

```bash
# Add Portainer Helm repo
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

# Install agent
helm install --create-namespace \
  -n portainer \
  portainer-agent \
  portainer/portainer-agent \
  --set env.key=AGENT_SECRET \
  --set env.value=shared-agent-secret
```

Or via Kubernetes manifest:

```yaml
# portainer-agent-k8s.yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: portainer-agent
  namespace: portainer
spec:
  selector:
    matchLabels:
      app: portainer-agent
  template:
    metadata:
      labels:
        app: portainer-agent
    spec:
      serviceAccountName: portainer-sa
      containers:
      - name: portainer-agent
        image: portainer/agent:latest
        env:
        - name: LOG_LEVEL
          value: "INFO"
        - name: KUBERNETES_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        ports:
        - containerPort: 9001
          protocol: TCP
        resources:
          limits:
            memory: "256Mi"
            cpu: "100m"

---
apiVersion: v1
kind: Service
metadata:
  name: portainer-agent
  namespace: portainer
spec:
  selector:
    app: portainer-agent
  ports:
    - port: 9001
      targetPort: 9001
  type: ClusterIP
```

```bash
kubectl apply -f portainer-agent-k8s.yml

# Get agent service cluster IP
kubectl get svc portainer-agent -n portainer
```

### Step 3: Add Kubernetes Environment in Portainer

1. Go to **Environments** → **Add environment**
2. Select **Kubernetes**
3. Select **Agent** as connection type
4. Enter the agent URL: `tcp://portainer-agent.portainer.svc.cluster.local:9001` (if Portainer is in cluster)
   OR `tcp://CLUSTER_IP:9001` (if Portainer is external)
5. Click **Connect**

## Method 2: Kubeconfig Import

1. Go to **Environments** → **Add environment**
2. Select **Kubernetes**
3. Select **Import** and paste your kubeconfig content
4. Click **Connect**

```bash
# Get your current kubeconfig
cat ~/.kube/config

# Or for a specific cluster
kubectl config view --raw --minify
```

## Verifying the Kubernetes Environment

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# List environments and check Kubernetes ones
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/endpoints \
  | python3 -c "
import sys, json
for env in json.load(sys.stdin):
    if env.get('Type') in [6, 7]:  # Type 6/7 = Kubernetes
        print(f'K8s: ID={env[\"Id\"]} Name={env[\"Name\"]} Status={env[\"Status\"]}')
"

# Check namespaces in the K8s environment
ENDPOINT_ID=6
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}/kubernetes/namespaces" \
  | python3 -c "import sys,json; [print(ns['Name']) for ns in json.load(sys.stdin)]"
```

## Configuring Kubernetes Environment Options

After connecting, configure the environment:

1. Click on the environment → **Settings**
2. Configure:
   - **Allow users to use specific storage classes**: Enable/disable specific storage
   - **Enable features using the metrics server**: For resource usage display
   - **Restrict default namespace**: Prevent users from deploying to `default`
   - **Namespace access management**: Enable per-namespace access control

## Conclusion

Kubernetes environments in Portainer provide a user-friendly interface to manage complex Kubernetes resources. The agent-based method is recommended for security and functionality — it provides deeper integration including the Portainer-specific RBAC for namespace-level access control. Once connected, your team can deploy applications, manage configurations, and troubleshoot Kubernetes workloads without kubectl expertise.
