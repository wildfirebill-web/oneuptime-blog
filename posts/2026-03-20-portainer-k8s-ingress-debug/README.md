# How to Debug Ingress Routing Problems in Portainer - K8s

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Ingress, Debugging, Networking

Description: Step-by-step guide to diagnosing and fixing Ingress routing issues in Kubernetes clusters managed by Portainer.

## Introduction

Ingress routing problems are among the most frustrating Kubernetes issues to debug. Traffic silently fails, returns 404s, or never reaches your pods. Portainer's Kubernetes interface provides a visual way to inspect Ingress objects, but effective debugging also requires CLI tools. This guide walks through systematic diagnosis of Ingress problems.

## Understanding Ingress Architecture

An Ingress request flows through multiple layers:

```text
Client → DNS → Load Balancer → Ingress Controller Pod → Service → Pod
```

Each layer can fail independently. The debugging strategy is to verify each hop.

## Step 1: Check the Ingress Object in Portainer

Navigate to **Kubernetes > Networking > Ingresses** in Portainer. Verify:
- The Ingress exists in the correct namespace
- The host and path rules are configured correctly
- The backend service name and port match your Service

```bash
# Also check via CLI

kubectl get ingress -A
kubectl describe ingress my-app-ingress -n production
```

Look for the `Address` field in the describe output - it should show the load balancer IP or hostname. If it's empty, the Ingress Controller isn't assigning an address.

## Step 2: Verify the Ingress Controller is Running

```bash
# Check for common ingress controllers
kubectl get pods -n ingress-nginx
kubectl get pods -n traefik
kubectl get pods -n kube-system | grep ingress

# Check Ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=50

# Verify the IngressClass
kubectl get ingressclass
```

If the Ingress Controller pods are crashing, fix them first before debugging routing.

## Step 3: Verify the IngressClass Annotation

```yaml
# Your Ingress must specify the correct IngressClass
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"  # Or use spec.ingressClassName
spec:
  ingressClassName: nginx  # Kubernetes 1.18+
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

```bash
# Check if a default IngressClass is set
kubectl get ingressclass -o yaml | grep default
```

## Step 4: Test the Backend Service

The Ingress routes to a Service, so the Service must be working:

```bash
# Verify the service exists and has endpoints
kubectl get svc my-app-service -n production
kubectl get endpoints my-app-service -n production

# If endpoints are empty, your pods aren't matching the service selector
kubectl describe svc my-app-service -n production
```

Check the Service selector matches the pod labels:

```bash
# Get pod labels
kubectl get pods -n production --show-labels

# Compare with service selector
kubectl get svc my-app-service -n production -o jsonpath='{.spec.selector}'
```

## Step 5: Test Connectivity Inside the Cluster

```bash
# Deploy a debug pod
kubectl run debug-pod --image=curlimages/curl --rm -it --restart=Never -- sh

# Inside the pod, test the service directly
curl http://my-app-service.production.svc.cluster.local

# Test the Ingress controller directly
kubectl get svc -n ingress-nginx
# Note the ClusterIP, then:
curl -H "Host: app.example.com" http://<ingress-controller-clusterip>
```

## Step 6: Check Path Matching Rules

A common issue is incorrect path types:

```bash
# PathType matters significantly
# Prefix: /api matches /api, /api/v1, /api/users
# Exact: /api matches ONLY /api, not /api/v1
# ImplementationSpecific: behavior depends on the controller
```

```yaml
# If your app is behind a prefix, ensure path stripping is configured
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /app(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

## Step 7: SSL/TLS Ingress Issues

```bash
# Check the TLS secret exists
kubectl get secret my-tls-secret -n production

# Verify the secret has the right fields
kubectl get secret my-tls-secret -n production -o jsonpath='{.data}' | python3 -c "
import sys, json, base64
data = json.load(sys.stdin)
for k, v in data.items():
    print(f'{k}: {len(base64.b64decode(v))} bytes')
"

# Check the Ingress TLS configuration
kubectl describe ingress my-app-ingress -n production | grep -A5 TLS
```

## Step 8: Check Ingress Controller Logs for Your Request

```bash
# Stream ingress-nginx logs while making a test request
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller -f &

# In another terminal, make a request
curl -v https://app.example.com/api/health

# Look for the request in the logs
# Successful: 200, Failed routing: 404 or "no endpoints"
```

## Diagnostic Script

```bash
#!/bin/bash
# ingress-debug.sh - Diagnose Ingress routing issues

NAMESPACE=${1:-default}
INGRESS_NAME=$2

if [ -z "$INGRESS_NAME" ]; then
  echo "Usage: $0 <namespace> <ingress-name>"
  exit 1
fi

echo "=== Ingress Details ==="
kubectl describe ingress "$INGRESS_NAME" -n "$NAMESPACE"

echo ""
echo "=== Backend Services ==="
SERVICES=$(kubectl get ingress "$INGRESS_NAME" -n "$NAMESPACE" \
  -o jsonpath='{.spec.rules[*].http.paths[*].backend.service.name}')
for svc in $SERVICES; do
  echo "Service: $svc"
  kubectl get endpoints "$svc" -n "$NAMESPACE" 2>/dev/null || echo "  ERROR: Service not found"
done

echo ""
echo "=== Ingress Controller Pods ==="
kubectl get pods -A -l 'app.kubernetes.io/component=controller' 2>/dev/null || \
kubectl get pods -n ingress-nginx 2>/dev/null

echo ""
echo "=== Recent Ingress Controller Events ==="
kubectl get events -n ingress-nginx --sort-by='.lastTimestamp' | tail -10
```

## Common Issues and Fixes

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| 404 on all paths | Wrong IngressClass | Add `ingressClassName` to spec |
| 502 Bad Gateway | Backend pods down | Check pod status and endpoints |
| SSL certificate error | Wrong TLS secret | Verify secret name and namespace |
| 308 redirect loop | HTTP→HTTPS redirect with HTTP backend | Set `ssl-redirect: "false"` annotation |
| Routing to wrong service | Path conflict | Check path specificity order |

## Conclusion

Debugging Kubernetes Ingress requires systematic verification of each layer: Ingress Controller health, IngressClass assignment, Service endpoints, pod readiness, and TLS configuration. Portainer's Ingress view provides a clear starting point, but the CLI commands above allow deeper inspection of the routing chain. Most Ingress issues fall into a small set of patterns: missing IngressClass, empty endpoints, path matching errors, and TLS misconfiguration.
