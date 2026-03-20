# How to Configure Istio Gateway in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Istio, Gateway, Ingress, Service Mesh

Description: A comprehensive guide to configuring Istio Gateway for managing inbound and outbound traffic at the edge of your service mesh in Rancher.

The Istio Gateway resource manages traffic entering and leaving the service mesh at its boundary. Unlike Kubernetes Ingress, Istio Gateway gives you full control over Layer 4-6 load balancing properties such as TLS settings and protocol selection, while VirtualServices handle the L7 routing rules. This guide covers how to configure Istio Gateways in a Rancher-managed cluster.

## Prerequisites

- Istio installed in your Rancher cluster with the ingress gateway enabled
- A domain name pointing to your Istio ingress gateway's external IP
- TLS certificates (we'll cover both self-signed and cert-manager approaches)
- `kubectl` access to the cluster

## Understanding Istio Gateway vs Kubernetes Ingress

The Istio Gateway works differently from a Kubernetes Ingress:

- **Gateway**: Configures the load balancer at the mesh edge (ports, protocols, TLS termination)
- **VirtualService**: Binds to a Gateway and configures routing rules
- This separation allows the same Gateway to be used by multiple teams with their own VirtualServices

## Step 1: Get the Ingress Gateway External IP

```bash
# Get the external IP assigned to the ingress gateway
kubectl get svc istio-ingressgateway -n istio-system

# Store the IP for use in DNS configuration
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.spec.ports[?(@.name=="https")].port}')

echo "Ingress IP: $INGRESS_HOST"
echo "HTTP Port: $INGRESS_PORT"
echo "HTTPS Port: $SECURE_INGRESS_PORT"
```

## Step 2: Create a Basic HTTP Gateway

```yaml
# http-gateway.yaml - Configure a basic HTTP gateway
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-app-gateway
  namespace: my-app
spec:
  # Select the ingress gateway pod using its label selector
  selector:
    istio: ingressgateway
  servers:
  # Configure HTTP on port 80
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    # Accept traffic for this hostname
    - "myapp.example.com"
```

```yaml
# virtual-service-gateway.yaml - Bind a VirtualService to the Gateway
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
  namespace: my-app
spec:
  hosts:
  - "myapp.example.com"
  # Reference the gateway
  gateways:
  - my-app-gateway
  http:
  - route:
    - destination:
        host: my-app-service
        port:
          number: 8080
```

```bash
kubectl apply -f http-gateway.yaml
kubectl apply -f virtual-service-gateway.yaml
```

## Step 3: Configure HTTPS with TLS Termination

### Using a Secret for TLS Certificates

```bash
# Create a TLS secret with your certificate and key
kubectl create -n istio-system secret tls myapp-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

```yaml
# https-gateway.yaml - Configure HTTPS gateway with TLS termination
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-app-https-gateway
  namespace: my-app
spec:
  selector:
    istio: ingressgateway
  servers:
  # Redirect HTTP to HTTPS
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "myapp.example.com"
    tls:
      # Redirect all HTTP traffic to HTTPS
      httpsRedirect: true
  # Configure HTTPS
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "myapp.example.com"
    tls:
      # Terminate TLS at the gateway
      mode: SIMPLE
      # Reference the TLS secret (must be in istio-system namespace)
      credentialName: myapp-tls-secret
```

## Step 4: Configure Mutual TLS (mTLS) at the Gateway

For applications requiring client certificate authentication:

```yaml
# mtls-gateway.yaml - Configure mTLS at the gateway
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-app-mtls-gateway
  namespace: my-app
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "myapp.example.com"
    tls:
      # Require client certificates
      mode: MUTUAL
      credentialName: myapp-tls-secret
      # CA certificate for validating client certs
      caCertificates: /etc/ssl/certs/ca-certificates.crt
```

## Step 5: Egress Gateway Configuration

Control outbound traffic from the mesh using an egress gateway:

```yaml
# egress-gateway.yaml - Route traffic to external services through egress gateway
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
  namespace: istio-system
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - external-api.example.com
    tls:
      mode: PASSTHROUGH
---
# ServiceEntry to register the external service
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-api
  namespace: my-app
spec:
  hosts:
  - external-api.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

## Step 6: Verify Gateway Configuration

```bash
# List all gateways
kubectl get gateway -A

# Describe the gateway for details
kubectl describe gateway my-app-gateway -n my-app

# Test the HTTP endpoint
curl -I http://myapp.example.com

# Test the HTTPS endpoint
curl -I https://myapp.example.com

# Use istioctl to check the proxy configuration on the ingress gateway
istioctl proxy-config listeners -n istio-system \
  deploy/istio-ingressgateway
```

## Conclusion

Istio Gateway provides a flexible and powerful way to manage traffic at the mesh boundary. By separating the gateway configuration (ports, protocols, TLS) from the routing rules (VirtualServices), Istio enables a clean separation of concerns that works well in multi-team environments. Combined with Rancher's cluster management capabilities, you can deploy and manage sophisticated ingress configurations with confidence.
