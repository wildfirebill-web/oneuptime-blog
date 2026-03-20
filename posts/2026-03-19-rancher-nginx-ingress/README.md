# How to Set Up NGINX Ingress Controller in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Ingress, Nginx

Description: Learn how to deploy and configure the NGINX Ingress Controller in a Rancher-managed Kubernetes cluster.

The NGINX Ingress Controller is one of the most popular ingress controllers for Kubernetes. It provides robust HTTP and HTTPS routing, load balancing, SSL termination, and advanced traffic management. This guide shows you how to set up and configure the NGINX Ingress Controller in Rancher.

## Prerequisites

- A running Rancher instance (v2.6 or later)
- A managed Kubernetes cluster (RKE, RKE2, or imported)
- Helm installed or access to the Rancher Apps marketplace
- kubectl access to your cluster

## Step 1: Check for Existing Ingress Controllers

Before installing, check if an ingress controller is already present:

```bash
kubectl get pods --all-namespaces | grep ingress
kubectl get ingressclass
```

RKE2 clusters often come with NGINX Ingress Controller pre-installed. If one already exists, you may want to update its configuration rather than install a new one.

## Step 2: Install NGINX Ingress Controller via Rancher Apps

1. Navigate to your cluster in the Rancher dashboard.
2. Go to **Apps** > **Charts**.
3. Search for **nginx-ingress** or **ingress-nginx**.
4. Select the **ingress-nginx** chart.
5. Click **Install**.
6. Choose the target namespace (typically `ingress-nginx`).
7. Configure the values as needed and click **Install**.

## Step 3: Install via Helm CLI

Alternatively, install using Helm from the command line:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer
```

For bare-metal clusters without a cloud load balancer:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

## Step 4: Verify the Installation

Check that the NGINX Ingress Controller pods are running:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

You should see the controller pod in a Running state and a service with an external IP (if using LoadBalancer type) or NodePort.

```bash
kubectl get ingressclass
```

Confirm that the `nginx` ingress class is available.

## Step 5: Configure the NGINX Ingress Controller

Customize the controller behavior by editing the ConfigMap:

```bash
kubectl edit configmap ingress-nginx-controller -n ingress-nginx
```

Common configuration options:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  proxy-body-size: "50m"
  proxy-read-timeout: "120"
  proxy-send-timeout: "120"
  use-forwarded-headers: "true"
  compute-full-forwarded-for: "true"
  enable-real-ip: "true"
  log-format-upstream: '$remote_addr - $request_id'
```

## Step 6: Create a Test Ingress Resource

Deploy a test application and create an Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
  - host: test.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 80
```

## Step 7: Enable Rate Limiting

Add rate limiting to protect your services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
    nginx.ingress.kubernetes.io/limit-connections: "5"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

## Step 8: Configure Custom Error Pages

Set up custom error pages for your ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: custom-errors-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/custom-http-errors: "404,503"
    nginx.ingress.kubernetes.io/default-backend: error-pages-service
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

## Step 9: Enable Monitoring

If you have Prometheus installed through Rancher Monitoring, enable metrics collection:

```bash
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.metrics.enabled=true \
  --set controller.metrics.serviceMonitor.enabled=true \
  --set controller.metrics.serviceMonitor.namespace=cattle-monitoring-system
```

## Step 10: Scale the Ingress Controller

For production environments, scale the controller for high availability:

```bash
kubectl scale deployment ingress-nginx-controller \
  -n ingress-nginx --replicas=3
```

Or configure it in the Helm values:

```yaml
controller:
  replicaCount: 3
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app.kubernetes.io/name
              operator: In
              values:
              - ingress-nginx
          topologyKey: kubernetes.io/hostname
```

## Troubleshooting

- Check controller logs: `kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx`
- Verify the IngressClass: `kubectl get ingressclass`
- Test connectivity: `curl -v -H "Host: test.example.com" http://<EXTERNAL-IP>/`
- Check for configuration errors: `kubectl exec -n ingress-nginx <pod> -- nginx -T`

## Summary

The NGINX Ingress Controller is a powerful and flexible solution for managing HTTP traffic in Rancher-managed Kubernetes clusters. By installing it through Rancher Apps or Helm, configuring its behavior via annotations and ConfigMaps, and enabling monitoring, you can build a production-ready ingress layer for your applications.
