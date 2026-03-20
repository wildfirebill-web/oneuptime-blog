# How to Configure Istio Sidecar Injection in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Istio, Service Mesh, Sidecar

Description: Learn how to configure automatic and manual Istio sidecar injection for your workloads running in Rancher-managed Kubernetes clusters.

Istio's sidecar injection is the mechanism by which Envoy proxy containers are added to your application pods. These sidecar proxies intercept all network traffic to and from your services, enabling traffic management, observability, and security features. This guide explains how to configure sidecar injection in a Rancher-managed environment.

## Prerequisites

- Istio installed on your Rancher-managed cluster (see the Istio installation guide)
- `kubectl` access to your cluster
- Basic understanding of Kubernetes namespaces and pod specifications

## Understanding Sidecar Injection

Istio supports two modes of sidecar injection:

1. **Automatic injection**: The Istio mutating webhook automatically injects the Envoy sidecar into all pods in labeled namespaces
2. **Manual injection**: You explicitly annotate pods or use `istioctl kube-inject` to inject sidecars

## Step 1: Enable Automatic Sidecar Injection at the Namespace Level

The most common approach is to enable automatic injection at the namespace level:

```bash
# Enable automatic sidecar injection for a specific namespace
kubectl label namespace my-app istio-injection=enabled

# Verify the label
kubectl get namespace my-app --show-labels
# Output should show: istio-injection=enabled

# To disable injection for a namespace
kubectl label namespace my-app istio-injection=disabled --overwrite
```

## Step 2: Control Injection at the Pod Level

You can override namespace-level injection settings at the individual pod level using annotations:

```yaml
# deployment.yaml - Force inject sidecar even if namespace injection is disabled
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        # Force enable injection for this specific pod
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: my-app
        image: my-app:latest
        ports:
        - containerPort: 8080
```

```yaml
# deployment-no-inject.yaml - Disable injection for a specific pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-utility
  namespace: my-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-utility
  template:
    metadata:
      labels:
        app: my-utility
      annotations:
        # Disable injection for this pod even if namespace has injection enabled
        sidecar.istio.io/inject: "false"
    spec:
      containers:
      - name: my-utility
        image: my-utility:latest
```

## Step 3: Manual Sidecar Injection

For cases where you need more control, use manual injection with `istioctl`:

```bash
# Inject sidecar into a deployment manifest and apply it
istioctl kube-inject -f deployment.yaml | kubectl apply -f -

# Or save the injected manifest for review
istioctl kube-inject -f deployment.yaml -o deployment-injected.yaml

# Verify the injected manifest has the sidecar container
grep -A5 "istio-proxy" deployment-injected.yaml
```

## Step 4: Customize Sidecar Configuration

You can customize the sidecar proxy behavior using the `Sidecar` custom resource:

```yaml
# sidecar-config.yaml - Restrict which services a workload can reach
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: my-app
spec:
  egress:
  # Only allow traffic to services in the same namespace
  - hosts:
    - "./*"
    # Allow traffic to the istio-system namespace for control plane
    - "istio-system/*"
```

```bash
# Apply the sidecar configuration
kubectl apply -f sidecar-config.yaml
```

## Step 5: Verify Sidecar Injection

After enabling injection and restarting pods, verify the sidecar is running:

```bash
# Check that pods have 2 containers (app + istio-proxy)
kubectl get pods -n my-app

# Expected output shows READY 2/2 for injected pods:
# NAME                      READY   STATUS    RESTARTS   AGE
# my-app-xxxxxxxxx-xxxxx    2/2     Running   0          1m

# Describe a pod to see the sidecar container details
kubectl describe pod my-app-xxxxxxxxx-xxxxx -n my-app | grep -A5 "istio-proxy"

# Check the sidecar proxy logs
kubectl logs my-app-xxxxxxxxx-xxxxx -n my-app -c istio-proxy
```

## Step 6: Restart Existing Pods to Apply Injection

Existing pods must be restarted to get the sidecar injected:

```bash
# Rolling restart of a deployment to trigger sidecar injection
kubectl rollout restart deployment/my-app -n my-app

# Monitor the rollout
kubectl rollout status deployment/my-app -n my-app

# For a DaemonSet
kubectl rollout restart daemonset/my-daemonset -n my-app
```

## Configuring Injection in Rancher UI

Rancher's UI provides a convenient way to manage Istio injection settings:

1. Navigate to your cluster in the Rancher UI
2. Go to **Istio** in the left menu (if Istio is installed)
3. Click on **Namespaces** to view injection status
4. Toggle the injection setting for each namespace

## Troubleshooting Injection Issues

```bash
# Check if the mutating webhook is properly configured
kubectl get mutatingwebhookconfiguration istio-sidecar-injector -o yaml

# Verify the webhook is targeting the correct namespaces
kubectl get mutatingwebhookconfiguration istio-sidecar-injector \
  -o jsonpath='{.webhooks[0].namespaceSelector}'

# Check Istiod logs for injection errors
kubectl logs -n istio-system -l app=istiod --tail=50
```

## Conclusion

Configuring Istio sidecar injection properly is fundamental to getting value from the service mesh. By using namespace-level labels for broad application and pod-level annotations for exceptions, you have precise control over which workloads participate in the mesh. Always restart pods after enabling injection to ensure all running workloads have the sidecar proxy deployed.
