# How to Set Up Nginx Ingress Controller with a MetalLB IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NGINX Ingress, MetalLB, Kubernetes, IPv4, Load Balancing, Networking

Description: Deploy the Nginx Ingress Controller on bare-metal Kubernetes with MetalLB providing an external IPv4 address for HTTP and HTTPS routing.

The Nginx Ingress Controller handles HTTP/HTTPS routing inside Kubernetes while MetalLB provides the external IPv4 address that external traffic enters through. Together they form a complete bare-metal ingress solution.

## Step 1: Prerequisites

```bash
# Ensure MetalLB is installed and configured

kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system

# Verify your IP pool has available addresses
# (e.g., 192.168.1.200-250)
```

## Step 2: Install Nginx Ingress Controller

```bash
# Install using the official manifest for bare-metal
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml

# Wait for it to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

The bare-metal manifest creates the controller with a `NodePort` service by default. Change it to `LoadBalancer` for MetalLB:

```bash
# Patch the service to use LoadBalancer type
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  -p '{"spec": {"type": "LoadBalancer"}}'

# Watch for MetalLB to assign an IP
kubectl get svc ingress-nginx-controller -n ingress-nginx -w
# EXTERNAL-IP: 192.168.1.200 (assigned by MetalLB)
```

## Step 3: Deploy a Sample Application

```yaml
# sample-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl apply -f sample-app.yaml
```

## Step 4: Create an Ingress Resource

```yaml
# hello-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  # Route hello.example.com to the hello service
  - host: hello.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 80
```

```bash
kubectl apply -f hello-ingress.yaml
```

## Step 5: Test the Setup

```bash
# Get the external IP assigned by MetalLB
EXTERNAL_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Ingress IP: $EXTERNAL_IP"

# Test with the Host header (or configure DNS/hosts file)
curl -H "Host: hello.example.com" http://$EXTERNAL_IP

# Or add to /etc/hosts for local testing
echo "$EXTERNAL_IP hello.example.com" | sudo tee -a /etc/hosts
curl http://hello.example.com
```

## Verifying the Full Chain

```bash
# MetalLB assigns IP to Nginx service
kubectl get svc ingress-nginx-controller -n ingress-nginx

# Nginx routes based on Host header to backend service
kubectl describe ingress hello-ingress

# Check Nginx controller logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | tail -20
```

This combination of MetalLB + Nginx Ingress is the standard approach for production-grade HTTP routing on bare-metal Kubernetes clusters.
