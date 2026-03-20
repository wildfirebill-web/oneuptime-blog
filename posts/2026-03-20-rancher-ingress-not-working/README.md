# How to Troubleshoot Ingress Not Working in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, Ingress, Networking

Description: Step-by-step troubleshooting guide for Ingress failures in Rancher-managed clusters, covering nginx-ingress, cert-manager, DNS, and backend connectivity.

## Introduction

Ingress resources in Rancher-managed clusters can fail to route traffic for several reasons: the Ingress controller may not be running, the backend service may be unreachable, TLS termination may be misconfigured, or DNS may not resolve correctly. This guide provides a systematic approach to isolating and fixing Ingress issues.

## Step 1: Verify the Ingress Resource

```bash
# Check the Ingress resource definition

kubectl get ingress -n <namespace> <ingress-name> -o yaml

# Look for the ADDRESS field - if empty, the Ingress controller hasn't admitted the resource
kubectl get ingress -n <namespace>
# NAME    CLASS   HOSTS                  ADDRESS         PORTS   AGE
# myapp   nginx   myapp.example.com      10.0.0.100      80,443  5m
# If ADDRESS is empty, the Ingress controller is not processing it
```

## Step 2: Check the Ingress Controller

```bash
# List Ingress controller pods
kubectl get pods -n ingress-nginx                             # nginx-ingress
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik  # traefik

# Check for controller errors
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=100

# Verify the IngressClass
kubectl get ingressclass
# NAME    CONTROLLER             PARAMETERS   AGE
# nginx   k8s.io/ingress-nginx   <none>       2d

# If your Ingress spec doesn't specify ingressClassName, set the default
kubectl annotate ingressclass nginx \
  ingressclass.kubernetes.io/is-default-class=true
```

## Step 3: Verify Backend Service and Endpoints

```bash
# Check that the backend service exists
kubectl get service -n <namespace> <backend-service>

# Check that the service has healthy endpoints
kubectl get endpoints -n <namespace> <backend-service>
# If endpoints list is empty, no Pods match the service selector

# Verify Pod selector matches
kubectl get pods -n <namespace> -l <selector-labels>
kubectl describe service -n <namespace> <backend-service> | grep Selector
```

## Step 4: Test Traffic Flow

```bash
# Test the backend service directly (port-forward)
kubectl port-forward -n <namespace> service/<backend-service> 8080:80
curl http://localhost:8080/

# Test via the Ingress controller's ClusterIP directly
INGRESS_IP=$(kubectl get service -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -H "Host: myapp.example.com" http://${INGRESS_IP}/

# Test via the node's IP with the NodePort
NODE_PORT=$(kubectl get service -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
curl -H "Host: myapp.example.com" http://<node-ip>:${NODE_PORT}/
```

## Step 5: Troubleshoot TLS/HTTPS

```bash
# Check TLS secret referenced in the Ingress
kubectl get ingress -n <namespace> <ingress-name> -o jsonpath='{.spec.tls}'

# Verify the secret exists and contains valid certificate data
kubectl get secret -n <namespace> <tls-secret-name>
kubectl get secret -n <namespace> <tls-secret-name> -o json \
  | jq -r '.data["tls.crt"]' | base64 -d | openssl x509 -noout -dates

# Check if the certificate's CN/SAN matches the Ingress hostname
kubectl get secret -n <namespace> <tls-secret-name> -o json \
  | jq -r '.data["tls.crt"]' | base64 -d \
  | openssl x509 -noout -text | grep -A2 "Subject Alternative Name"
```

## Step 6: Check Ingress Controller Configuration

```bash
# View the nginx configuration generated for your Ingress
NGINX_POD=$(kubectl get pods -n ingress-nginx \
  -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

# Exec into the nginx pod and check configuration
kubectl exec -n ingress-nginx ${NGINX_POD} -- nginx -T 2>/dev/null \
  | grep -A 20 "server_name myapp.example.com"

# Check for Ingress admission webhook issues
kubectl get validatingwebhookconfiguration | grep ingress
kubectl describe validatingwebhookconfiguration ingress-nginx-admission
```

## Step 7: Check LoadBalancer Service

```bash
# The Ingress controller needs an external IP from a LoadBalancer service
kubectl get service -n ingress-nginx ingress-nginx-controller

# If EXTERNAL-IP is <pending>, the LoadBalancer provisioning failed
# For bare-metal clusters, MetalLB or similar is required
kubectl get pods -n metallb-system

# For cloud providers, check the cloud controller manager logs
kubectl logs -n kube-system -l component=cloud-controller-manager --tail=50
```

## Common Ingress Annotations

```yaml
# Example well-annotated Ingress for nginx
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    # Specify the IngressClass
    kubernetes.io/ingress.class: nginx
    # TLS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Increase proxy timeout for slow backends
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    # Custom error page
    nginx.ingress.kubernetes.io/custom-http-errors: "404,503"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
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

## Conclusion

Ingress troubleshooting in Rancher follows a clear path: verify the resource is admitted, confirm the Ingress controller is healthy, validate that backend services have endpoints, test the traffic path layer by layer, and finally check TLS configuration. The most common issues are empty endpoints (pod selector mismatch), missing IngressClass, and TLS certificate not matching the hostname.
