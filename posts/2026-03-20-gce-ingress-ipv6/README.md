# How to Configure GCE Ingress Controller for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, GKE, GCE, Google Cloud, Kubernetes, Ingress, Load Balancer

Description: Configure the GCE Ingress Controller on Google Kubernetes Engine (GKE) to create IPv6-capable Global HTTP(S) Load Balancers, including dual-stack service configuration and IPv6 external IP assignment.

## Introduction

Google Kubernetes Engine (GKE) uses the GCE Ingress Controller to provision Google Cloud's HTTP(S) Load Balancer for Kubernetes Ingress resources. GKE supports IPv6 for both external and internal load balancers when the cluster is configured with dual-stack networking. External HTTP(S) Load Balancers on GCP support IPv6 natively through anycast IPv6 addresses.

## Prerequisites: GKE Cluster with IPv6

```bash
# Create a GKE cluster with dual-stack networking

gcloud container clusters create my-cluster \
    --location=us-central1 \
    --enable-ip-alias \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=EXTERNAL \
    --cluster-ipv4-cidr=/16 \
    --cluster-ipv6-cidr=/48

# Or for private cluster with internal IPv6
gcloud container clusters create my-cluster \
    --location=us-central1 \
    --enable-ip-alias \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=INTERNAL

# Verify cluster has IPv6 pod CIDR
kubectl get node -o yaml | grep -A5 "podCIDRs"
# Should show: - 10.x.x.x/24 and - xxxx::/112

# Install credentials
gcloud container clusters get-credentials my-cluster --location=us-central1
```

## GKE Ingress with IPv6 (Global Load Balancer)

```yaml
# ingress-gce-ipv6.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    # Use GCE (global) ingress class
    kubernetes.io/ingress.class: "gce"

    # IPv6 is automatically included when:
    # 1. Cluster has stack-type=IPV4_IPV6
    # 2. Nodes have external IPv6 addresses (ipv6-access-type=EXTERNAL)

    # Static global IP (create with gcloud compute addresses)
    kubernetes.io/ingress.global-static-ip-name: "myapp-global-ip"

    # TLS configuration
    ingress.gcp.kubernetes.io/pre-shared-cert: "myapp-ssl-cert"

    # HTTP-to-HTTPS redirect
    kubernetes.io/ingress.allow-http: "false"

spec:
  ingressClassName: gce
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com
```

## Create Static Global IPv6 Address

```bash
# Create a global external IPv6 anycast address for the load balancer
gcloud compute addresses create myapp-global-ipv6 \
    --ip-version=IPV6 \
    --global

# Get the assigned IPv6 address
gcloud compute addresses describe myapp-global-ipv6 --global \
    --format="get(address)"
# Returns: 2600:1901:xxxx:xxxx::

# Create a dual-stack global IP (both IPv4 and IPv6)
# Note: GCP provides a single IPv4 and single IPv6 address
gcloud compute addresses create myapp-global-ip --global
gcloud compute addresses create myapp-global-ipv6 --ip-version=IPV6 --global

# Update DNS with the IPv6 address
gcloud dns record-sets transaction start --zone=example-zone
gcloud dns record-sets transaction add "2600:1901:xxxx:xxxx::" \
    --name=app.example.com. \
    --ttl=300 \
    --type=AAAA \
    --zone=example-zone
gcloud dns record-sets transaction execute --zone=example-zone
```

## BackendConfig for IPv6 Health Checks

```yaml
# backendconfig-ipv6.yaml

apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: myapp-backend-config
  namespace: production
spec:
  # Health check configuration (same for IPv4 and IPv6 backends)
  healthCheck:
    checkIntervalSec: 10
    timeoutSec: 5
    healthyThreshold: 2
    unhealthyThreshold: 3
    type: HTTP
    requestPath: /health
    port: 8080

  # Connection draining
  connectionDraining:
    drainingTimeoutSec: 60

  # Session affinity (optional, works with IPv6 clients)
  sessionAffinity:
    affinityType: GENERATED_COOKIE
    affinityCookieTtlSec: 3600
```

```yaml
# Attach BackendConfig to service
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
  annotations:
    # Link BackendConfig for health check and other LB settings
    cloud.google.com/backend-config: '{"default": "myapp-backend-config"}'
spec:
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6
  selector:
    app: myapp
  type: NodePort    # GCE Ingress requires NodePort service
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

## GKE Ingress for Internal IPv6 Load Balancer

```yaml
# ingress-gce-internal-ipv6.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-internal
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "gce-internal"
    # Internal load balancer with IPv6
    # Requires cluster with ipv6-access-type=INTERNAL

    # Subnet for the internal LB
    kubernetes.io/ingress.regional-static-ip-name: "myapp-internal-ip"
spec:
  rules:
    - host: app.internal.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080
```

## Verify GCE Ingress IPv6 Operation

```bash
# Check ingress has an address
kubectl describe ingress myapp -n production | grep -E "Address|Rules"

# Get the load balancer IPv6 address
gcloud compute addresses list --global | grep ipv6

# Test DNS resolution (should return AAAA record)
dig AAAA app.example.com
# Expected: app.example.com AAAA 2600:1901:xxxx::

# Test HTTP over IPv6
curl -6 -H "Host: app.example.com" "http://[2600:1901:xxxx::]:80/"

# Check backend health (IPv6 pods should be HEALTHY)
gcloud compute backend-services get-health myapp-backend-service \
    --global \
    --format="table(status.healthStatus[].ipAddress,status.healthStatus[].healthState)"

# Check forwarding rules include IPv6
gcloud compute forwarding-rules list --global | grep IPV6
```

## GKE Network Policy for IPv6

```yaml
# network-policy-ipv6.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-lb-ipv6
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
    # Allow from GCP load balancer health check ranges (both IPv4 and IPv6)
    - from:
        - ipBlock:
            cidr: "35.191.0.0/16"    # GCP health check IPv4
        - ipBlock:
            cidr: "130.211.0.0/22"   # GCP health check IPv4
        - ipBlock:
            cidr: "2600:1901::/32"   # GCP load balancer IPv6 range
      ports:
        - port: 8080
```

## Conclusion

GKE's GCE Ingress Controller creates Global HTTP(S) Load Balancers with IPv6 support when the GKE cluster is configured with `--stack-type=IPV4_IPV6` and `--ipv6-access-type=EXTERNAL`. Static global IPv6 anycast addresses are created with `gcloud compute addresses create --ip-version=IPV6 --global` and referenced via the `kubernetes.io/ingress.global-static-ip-name` annotation. The BackendConfig CRD configures health checks and other load balancer settings that work identically for IPv4 and IPv6 backends. GKE services must use `type: NodePort` for GCE Ingress. Update DNS with AAAA records pointing to the global IPv6 anycast address for IPv6 client access. Network policies must include GCP's IPv6 health check ranges (`2600:1901::/32`) to allow health probes to reach backend pods.
