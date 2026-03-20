# How to Configure Ambassador/Emissary Ingress for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Ambassador, Emissary-Ingress, Kubernetes, Envoy, API Gateway

Description: Configure Ambassador (now Emissary-Ingress) API gateway for IPv6 in Kubernetes, including service exposure with IPv6 load balancers, Mapping configuration for IPv6 backends, and client IP handling.

## Introduction

Ambassador, now known as Emissary-Ingress, is an API gateway for Kubernetes built on Envoy Proxy. For IPv6, Emissary-Ingress leverages Envoy's native IPv6 listener support and can expose services over IPv6 load balancers. Mapping and Host CRDs configure routing to IPv6-capable backend services in dual-stack clusters.

## Install Emissary-Ingress with IPv6 (Helm)

```yaml
# emissary-values.yaml

service:
  type: LoadBalancer
  # Dual-stack for IPv6 external access
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6
  annotations:
    # AWS NLB dual-stack
    service.beta.kubernetes.io/aws-load-balancer-ip-address-type: "dualstack"
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"

# Emissary (Envoy) listens on all interfaces including IPv6 by default
# when deployed in a dual-stack Kubernetes cluster
```

```bash
# Install Emissary-Ingress via Helm
helm repo add datawire https://app.getambassador.io
helm install emissary-ingress datawire/emissary-ingress \
    -n emissary-system \
    --create-namespace \
    -f emissary-values.yaml

# Verify installation and IPv6 external IP
kubectl get svc emissary-ingress -n emissary-system
```

## Ambassador Mapping for IPv6 Backend

```yaml
# mapping-ipv6.yaml

apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: myapp-mapping
  namespace: production
spec:
  hostname: "api.example.com"
  prefix: /api/
  service: myapp:8080   # Kubernetes service name:port

  # Headers for IPv6 clients
  set_request_headers:
    X-Forwarded-Proto: "https"

  # Timeout configuration
  connect_timeout_ms: 5000
  timeout_ms: 60000
  idle_timeout_ms: 300000

  # Load balancing policy
  load_balancer:
    policy: round_robin
```

## Ambassador Host for TLS

```yaml
# host-ipv6.yaml

apiVersion: getambassador.io/v3alpha1
kind: Host
metadata:
  name: api-host
  namespace: production
spec:
  hostname: api.example.com
  acmeProvider:
    email: admin@example.com
    authority: https://acme-v02.api.letsencrypt.org/directory
  tlsSecret:
    name: api-tls
  requestPolicy:
    insecure:
      action: Redirect   # Redirect HTTP to HTTPS
```

## AmbassadorListener for IPv6

```yaml
# listener-ipv6.yaml

apiVersion: getambassador.io/v3alpha1
kind: Listener
metadata:
  name: https-listener
  namespace: emissary-system
spec:
  port: 8443
  protocol: HTTPS
  securityModel: XFP
  hostBinding:
    namespace:
      from: ALL

---
apiVersion: getambassador.io/v3alpha1
kind: Listener
metadata:
  name: http-listener
  namespace: emissary-system
spec:
  port: 8080
  protocol: HTTP
  securityModel: XFP
  hostBinding:
    namespace:
      from: ALL
```

## RateLimitService for IPv6

```yaml
# ratelimit-ipv6.yaml

apiVersion: getambassador.io/v3alpha1
kind: RateLimitService
metadata:
  name: ratelimit
  namespace: production
spec:
  service: "ratelimit.ratelimit-system:8081"
  protocol_version: v3
  timeout_ms: 1000

---
# Apply rate limit to a Mapping
apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: rate-limited-api
  namespace: production
spec:
  hostname: "api.example.com"
  prefix: /v1/
  service: api:8080
  rate_limits:
    - descriptor: "source_ip"
      # Rate limit by client IPv6 (full /128 address)
      # For IPv6 prefix-based limiting, configure in ratelimit service
      request_headers:
        - header_name: "X-Forwarded-For"
          descriptor_key: "source_ip"
```

## IPv6 Client IP Extraction in Emissary

```yaml
# xff-config.yaml — Configure trusted proxy CIDRs

apiVersion: getambassador.io/v3alpha1
kind: Module
metadata:
  name: ambassador
  namespace: emissary-system
spec:
  config:
    # Trusted proxies for X-Forwarded-For (IPv4 and IPv6)
    xff_num_trusted_hops: 1   # Skip the cloud load balancer hop
    # Emissary uses Envoy's XFF trusted hop count
    # For IPv6 LBs: set xff_num_trusted_hops to number of proxy hops
```

## Verify Emissary IPv6 Operation

```bash
# Check Emissary pods have IPv6
kubectl get pods -n emissary-system -o wide
# Pod IPs should be IPv6 in dual-stack cluster

# Check Envoy listeners in Emissary
kubectl exec -n emissary-system deployment/emissary-ingress -- \
    curl -s http://localhost:8001/listeners | jq '.listener_statuses[].name'
# Should show listeners on :: addresses

# Test routing over IPv6
EMISSARY_IPV6=$(kubectl get svc emissary-ingress -n emissary-system \
    -o jsonpath='{.status.loadBalancer.ingress[?(@.ipMode=="VIP")].ip}')

curl -6 -H "Host: api.example.com" \
    "http://[$EMISSARY_IPV6]:80/api/health"

# Check Mapping status
kubectl get mappings -n production
kubectl describe mapping myapp-mapping -n production | grep -A5 "Status"
```

## Conclusion

Emissary-Ingress (Ambassador) supports IPv6 through its underlying Envoy Proxy, which listens on `::` for all IPv6 interfaces when the pod has IPv6 connectivity in a dual-stack cluster. The Kubernetes service uses `ipFamilyPolicy: PreferDualStack` for IPv6 external load balancer addresses. Mapping CRDs route IPv6 client requests to backend Kubernetes services by name — service name resolution in dual-stack clusters returns both IPv4 and IPv6 endpoints. Configure `xff_num_trusted_hops` in the Ambassador Module to extract real client IPv6 addresses when behind cloud load balancers. The Host CRD manages TLS termination, and certificates for IPv6 ingress must include domain SANs (not IP SANs, since hostnames are used).
