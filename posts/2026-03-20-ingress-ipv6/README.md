# How to Configure IPv6 Ingress in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, Ingress, NGINX, Dual-Stack, Load Balancer

Description: Configure Kubernetes Ingress controllers for IPv6, set up nginx-ingress-controller to accept IPv6 client connections, and create Ingress resources for dual-stack traffic routing.

## Introduction

Kubernetes Ingress provides HTTP/HTTPS routing to services. For IPv6 support, the Ingress controller (typically NGINX or Traefik) must bind to IPv6 addresses. The NGINX Ingress Controller supports IPv6 by binding to `::` (all interfaces) when IPv6 is enabled. Ingress objects themselves are address-family agnostic — IPv6 access is handled at the controller level, not in Ingress YAML.

## Install NGINX Ingress Controller with IPv6

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.5/deploy/static/provider/cloud/deploy.yaml

# Check if NGINX service has IPv6 ClusterIP
kubectl -n ingress-nginx get svc ingress-nginx-controller

# Patch service to prefer dual-stack
kubectl -n ingress-nginx patch svc ingress-nginx-controller \
    -p '{"spec":{"ipFamilyPolicy":"PreferDualStack","ipFamilies":["IPv4","IPv6"]}}'

# Verify the service has both ClusterIPs
kubectl -n ingress-nginx get svc ingress-nginx-controller \
    -o jsonpath='{.spec.clusterIPs}'
```

## Configure NGINX Ingress for IPv6 Binding

```yaml
# Patch NGINX Ingress Controller to explicitly bind IPv6
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  # Enable dual-stack in NGINX
  use-forwarded-headers: "true"
  # NGINX will listen on :: (all interfaces including IPv6)
  bind-address: "::"
```

```bash
# Apply ConfigMap
kubectl apply -f nginx-configmap.yaml

# Restart NGINX Ingress Controller to apply
kubectl -n ingress-nginx rollout restart deployment/ingress-nginx-controller

# Check NGINX is listening on IPv6
kubectl -n ingress-nginx exec deployment/ingress-nginx-controller -- \
    ss -tlnp6 | grep ":80\|:443"
```

## Create Ingress Resource for IPv6 Traffic

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-real-ip-header: "X-Real-IP"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
      secretName: example-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

```bash
kubectl apply -f ingress.yaml

# Test via IPv6 (if Ingress has external IPv6)
INGRESS_IPV6=$(kubectl -n ingress-nginx get svc ingress-nginx-controller \
    -o jsonpath='{.status.loadBalancer.ingress[?(@.ip contains ":")].ip}')

curl -6 -H "Host: example.com" "https://[$INGRESS_IPV6]/" --insecure
```

## NGINX Ingress with External IPv6 Load Balancer

```yaml
# For cloud environments: annotate for dual-stack external LB
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
  annotations:
    # AWS: get dual-stack NLB
    service.beta.kubernetes.io/aws-load-balancer-ip-address-type: dualstack
    # GCP: get dual-stack LB
    cloud.google.com/load-balancer-type: "External"
spec:
  type: LoadBalancer
  ipFamilyPolicy: PreferDualStack
  ipFamilies: [IPv4, IPv6]
  ports:
    - name: http
      port: 80
    - name: https
      port: 443
  selector:
    app.kubernetes.io/name: ingress-nginx
```

## Verify IPv6 Ingress Connectivity

```bash
# Get Ingress external IPs
kubectl -n ingress-nginx get svc ingress-nginx-controller \
    -o jsonpath='{.status.loadBalancer.ingress}'

# Test with curl over IPv6
curl -6 -v "https://example.com/" 2>&1 | grep "Connected to"
# Should show IPv6 address

# Check NGINX access logs for IPv6 clients
kubectl -n ingress-nginx logs deployment/ingress-nginx-controller | \
    grep "::" | tail -20

# Add AAAA DNS record for your domain
dig AAAA example.com
# Should return the Ingress controller's IPv6 external IP
```

## Conclusion

Configure Kubernetes NGINX Ingress for IPv6 by setting `bind-address: "::"` in the Ingress controller's ConfigMap, patching the Service to use `ipFamilyPolicy: PreferDualStack`, and adding IPv6 load balancer annotations in cloud environments. Ingress YAML resources themselves are IP-family agnostic — IPv6 routing happens at the controller level. Add AAAA DNS records pointing to the Ingress controller's external IPv6 for clients to connect via IPv6. Verify connectivity with `curl -6 -H "Host: example.com" https://[ingress-ipv6]/`.
