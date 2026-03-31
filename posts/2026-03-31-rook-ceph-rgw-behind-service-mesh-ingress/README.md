# How to Set Up Ceph RGW Behind Service Mesh Ingress

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Service Mesh, Ingress, Kubernetes, S3

Description: Learn how to expose Ceph RGW (S3 API) through a service mesh ingress gateway, enabling TLS termination, traffic management, and observability for object storage access.

---

## Why Use Service Mesh Ingress for RGW

Exposing Ceph RGW through a service mesh ingress gateway (like Istio Ingress Gateway) provides:
- Centralized TLS termination with cert-manager integration
- Traffic routing and load balancing across RGW instances
- Rate limiting and circuit breaking for S3 traffic
- Unified observability with other services
- JWT authentication for RGW access

## Step 1 - Deploy Ceph RGW

Create a CephObjectStore with RGW service:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
  gateway:
    port: 80
    instances: 2
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
```

Verify the RGW service is running:

```bash
kubectl -n rook-ceph get svc rook-ceph-rgw-my-store
```

## Step 2 - Create the Istio Gateway

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: rgw-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: rgw-tls-cert
    hosts:
    - s3.example.com
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - s3.example.com
    tls:
      httpsRedirect: true
```

## Step 3 - Create TLS Certificate

Using cert-manager to provision the certificate:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rgw-tls-cert
  namespace: istio-system
spec:
  secretName: rgw-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - s3.example.com
```

## Step 4 - Create the VirtualService

Route traffic from the gateway to the RGW service:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: rgw-vs
  namespace: rook-ceph
spec:
  hosts:
  - s3.example.com
  gateways:
  - istio-system/rgw-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: rook-ceph-rgw-my-store.rook-ceph.svc.cluster.local
        port:
          number: 80
    timeout: 300s
    retries:
      attempts: 3
      retryOn: 5xx,reset,connect-failure
```

## Step 5 - Configure DNS

Point your S3 endpoint DNS to the Istio Ingress Gateway's external IP:

```bash
INGRESS_IP=$(kubectl -n istio-system get svc istio-ingressgateway \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Add DNS record: s3.example.com -> $INGRESS_IP"
```

## Testing the Setup

```bash
# Configure AWS CLI with Ceph credentials
aws configure set aws_access_key_id <rgw-access-key>
aws configure set aws_secret_access_key <rgw-secret-key>

# Test S3 access through the service mesh ingress
aws --endpoint-url https://s3.example.com s3 ls
aws --endpoint-url https://s3.example.com s3 mb s3://test-bucket
```

## Summary

Exposing Ceph RGW through a service mesh ingress gateway combines Ceph's S3-compatible object storage with enterprise-grade traffic management. The setup involves creating a CephObjectStore, configuring an Istio Gateway for TLS termination, and routing traffic via a VirtualService. This approach centralizes certificate management and provides consistent observability alongside your other services, making RGW a first-class citizen in your service mesh architecture.
