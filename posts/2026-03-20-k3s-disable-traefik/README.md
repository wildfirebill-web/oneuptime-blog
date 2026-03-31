# How to Disable Traefik in K3s

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Rancher, Traefik, Ingress, Nginx

Description: Learn how to disable the default Traefik ingress controller in K3s and replace it with an alternative like NGINX Ingress Controller.

## Introduction

K3s ships with Traefik v2 as its default ingress controller. While Traefik is an excellent ingress solution, many teams prefer NGINX Ingress Controller for its compatibility, community support, and familiarity. Others may use Istio, Kong, or another ingress solution that conflicts with Traefik. This guide shows how to cleanly disable Traefik and deploy a replacement.

## Method 1: Disable Traefik Before Installation

The cleanest approach is to disable Traefik before K3s is installed:

```bash
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
token: "ClusterToken"
# Disable Traefik during installation

disable:
  - traefik
EOF

# Install K3s
curl -sfL https://get.k3s.io | sudo sh -

# Verify Traefik is NOT deployed
kubectl get pods -n kube-system | grep traefik
# Should return empty
```

## Method 2: Disable Traefik on an Existing K3s Cluster

If K3s is already running with Traefik:

```bash
# Add traefik to the disable list
sudo tee -a /etc/rancher/k3s/config.yaml > /dev/null <<EOF
disable:
  - traefik
EOF

# Restart K3s to apply the change
sudo systemctl restart k3s

# K3s will automatically delete the Traefik Helm chart
# Monitor the deletion
kubectl -n kube-system get pods -w | grep traefik
```

## Verifying Traefik is Disabled

```bash
# Check no Traefik pods are running
kubectl get pods -n kube-system | grep traefik

# Check no Traefik services exist
kubectl get svc -n kube-system | grep traefik

# Check no Traefik HelmChart resource exists
kubectl get helmchart -n kube-system
# traefik should not appear in the list
```

## Installing NGINX Ingress Controller

After disabling Traefik, install the NGINX Ingress Controller:

```bash
# Add the ingress-nginx Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --version 4.9.1

# Wait for it to be ready
kubectl -n ingress-nginx rollout status deployment/ingress-nginx-controller
```

### Configuring NGINX with ServiceLB (K3s Default LB)

If you're keeping ServiceLB, NGINX will get a LoadBalancer IP automatically:

```bash
# Check the assigned external IP
kubectl -n ingress-nginx get svc ingress-nginx-controller

# Expected output:
# NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)
# ingress-nginx-controller   LoadBalancer   10.43.x.x      192.168.1.x    80,443
```

### Configuring NGINX with NodePort (Bare Metal)

```bash
# Install with NodePort for bare metal environments
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --set controller.service.type=NodePort \
    --set controller.service.nodePorts.http=80 \
    --set controller.service.nodePorts.https=443 \
    --set controller.hostPort.enabled=true
```

### Configuring NGINX with MetalLB

```bash
# Install with LoadBalancer type (MetalLB provides the IP)
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --set controller.service.type=LoadBalancer \
    --set controller.service.annotations."metallb\.universe\.tf/address-pool"=production
```

## Migrating Traefik IngressRoutes to Kubernetes Ingress

If you had Traefik-specific `IngressRoute` objects, migrate them to standard `Ingress` resources:

### Before (Traefik IngressRoute)

```yaml
# Traefik-specific IngressRoute (not usable with NGINX)
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: myapp
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`myapp.example.com`)
      kind: Rule
      services:
        - name: myapp-svc
          port: 80
```

### After (Standard Kubernetes Ingress for NGINX)

```yaml
# Standard Ingress (works with NGINX, Traefik, and most ingress controllers)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    # NGINX-specific annotations
    nginx.ingress.kubernetes.io/rewrite-target: /
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
                name: myapp-svc
                port:
                  number: 80
```

## NGINX with TLS/HTTPS

```yaml
# nginx-ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-tls
  annotations:
    # Use cert-manager for automatic TLS
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls-secret
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-svc
                port:
                  number: 80
```

## Customizing NGINX Ingress with HelmChartConfig

You can also customize the NGINX deployment via Helm values after installation:

```bash
# Customize NGINX with specific annotations and settings
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --set controller.metrics.enabled=true \
    --set controller.metrics.serviceMonitor.enabled=true \
    --set controller.config.proxy-body-size="100m" \
    --set controller.config.proxy-read-timeout="300" \
    --set controller.replicaCount=2
```

## Testing the New Ingress

```bash
# Deploy a test application
kubectl create deployment test-app --image=nginx
kubectl expose deployment test-app --port=80

# Create an ingress rule
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
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
                name: test-app
                port:
                  number: 80
EOF

# Add hosts entry for local testing
echo "$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}') test.example.com" | sudo tee -a /etc/hosts

# Test
curl http://test.example.com
```

## Conclusion

Disabling Traefik in K3s is a one-line configuration change. Once disabled, K3s automatically removes the Traefik Helm chart and all associated resources. Replacing it with NGINX Ingress Controller is straightforward with Helm. The key is to ensure you convert any Traefik-specific `IngressRoute` resources to standard Kubernetes `Ingress` objects, which are compatible with NGINX and any other standards-compliant ingress controller.
