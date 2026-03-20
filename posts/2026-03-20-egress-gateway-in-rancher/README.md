# How to Configure Egress Gateway in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Egress Gateway, Networks, Calico, Kubernetes, Security

Description: Configure an egress gateway in Rancher to route outbound traffic from specific namespaces or pods through a dedicated IP for firewall whitelisting and compliance.

## Introduction

Egress gateways provide a fixed, predictable source IP for outbound connections from Kubernetes pods. In regulated environments, external firewalls require whitelisted source IPs. Without an egress gateway, pods use different node IPs depending on scheduling, making firewall rules unstable.

## Use Cases

- Whitelisting Kubernetes services at external firewalls
- Compliance requirements for predictable outbound IPs
- Network monitoring and auditing
- Connecting to third-party APIs that require IP whitelisting

## Option 1: Calico Egress Gateway

```yaml
# Install Calico Egress Gateway (requires Calico Enterprise or Calico Cloud)

# Or use open-source Calico with EgressIPSet

# Create an IP pool for egress
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: egress-ip-pool
spec:
  cidr: 10.0.100.0/29    # Small pool for egress IPs
  natOutgoing: false       # Use the pool IP directly (no SNAT)
  ipipMode: Never
  vxlanMode: Never
```

## Option 2: Istio Egress Gateway

If Istio is installed in your cluster:

```yaml
# istio-egress-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: egress-gateway
  namespace: istio-system
spec:
  selector:
    istio: egressgateway
  servers:
    - port:
        number: 443
        name: tls
        protocol: TLS
      hosts:
        - api.external-service.com
      tls:
        mode: PASSTHROUGH
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-external-service-through-egress
  namespace: production
spec:
  hosts:
    - api.external-service.com
  gateways:
    - mesh
    - istio-system/egress-gateway
  tls:
    - match:
        - gateways:
            - mesh
          port: 443
          sniHosts:
            - api.external-service.com
      route:
        - destination:
            host: istio-egressgateway.istio-system.svc.cluster.local
            subset: tls
```

## Option 3: NGINX Egress Proxy

A simpler approach using NGINX as an HTTP egress proxy:

```yaml
# nginx-egress-proxy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: egress-proxy
  namespace: kube-system
spec:
  replicas: 2
  template:
    spec:
      # Pin to specific nodes with known external IPs
      nodeSelector:
        egress-node: "true"
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
```

```nginx
# nginx.conf for forward proxy
events {}
http {
    server {
        listen 8080;
        location / {
            proxy_pass http://$http_host$request_uri;
            resolver 8.8.8.8;
        }
    }
}
```

## Configure Pods to Use Egress Proxy

```yaml
# Pod environment variables
env:
  - name: HTTP_PROXY
    value: "http://egress-proxy.kube-system.svc.cluster.local:8080"
  - name: HTTPS_PROXY
    value: "http://egress-proxy.kube-system.svc.cluster.local:8080"
  - name: NO_PROXY
    value: "localhost,cluster.local,10.0.0.0/8"    # Bypass proxy for internal
```

## Conclusion

Egress gateways in Rancher solve the compliance challenge of unpredictable outbound IPs from Kubernetes pods. The right solution depends on your CNI: Calico users can use EgressIPSet, Istio users get a built-in egress gateway, and simpler deployments can use an NGINX proxy approach.
