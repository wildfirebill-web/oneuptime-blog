# How to Configure OVN-Kubernetes for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OVN-Kubernetes, IPv6, Kubernetes, CNI, OVN, Dual-Stack, SDN

Description: Configure OVN-Kubernetes CNI for IPv6 and dual-stack clusters, including network topology, load balancing, and network policy for IPv6 pods and services.

## Introduction

OVN-Kubernetes uses Open Virtual Network (OVN) and Open vSwitch (OVS) to provide software-defined networking for Kubernetes. It supports IPv6 and dual-stack natively, powering OpenShift's default SDN.

## Step 1: Install OVN-Kubernetes

```bash
# Clone OVN-Kubernetes

git clone https://github.com/ovn-org/ovn-kubernetes.git
cd ovn-kubernetes

# Install with dual-stack support
kubectl apply -f dist/images/ovn-kubernetes.yaml
```

## Step 2: ConfigMap for Dual-Stack

```yaml
# ovn-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ovn-config
  namespace: ovn-kubernetes
data:
  # Enable IPv6
  v6cidr: "fd00:10:244::/48"
  v4cidr: "10.244.0.0/16"

  # Service subnets
  v6servicecidr: "fd00:10:96::/108"
  v4servicecidr: "10.96.0.0/12"

  # Enable dual-stack
  enable_ipv6: "true"
  enable_ipv4: "true"

  # OVN northbound database
  net_cidr: "10.244.0.0/16/24,fd00:10:244::/48/64"
  svc_cidr: "10.96.0.0/12,fd00:10:96::/108"
```

## Step 3: Verify OVN IPv6 Logical Network

```bash
# Check OVN logical switches
ovn-nbctl ls-list

# View logical ports with IPv6 addresses
ovn-nbctl lsp-list <switch-name>

# Check OVN logical routers
ovn-nbctl lr-list

# View IPv6 static routes
ovn-nbctl lr-route-list <router-name>

# Check load balancers for IPv6 services
ovn-nbctl lb-list | grep "fd00:"
```

## Step 4: NetworkPolicy with IPv6

```yaml
# Standard Kubernetes NetworkPolicy works with OVN-K8s
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ipv6-ingress-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 2001:db8::/32
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

## Step 5: IPv6 Service Configuration

```yaml
# ipv6-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service-ipv6
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
  # Dual-stack service
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv6
    - IPv4
  type: LoadBalancer
```

```bash
# Apply and check assigned IPs
kubectl apply -f ipv6-service.yaml
kubectl get service web-service-ipv6
# Should show both IPv4 and IPv6 cluster IPs
```

## Step 6: Debug IPv6 in OVN-Kubernetes

```bash
# Check pod's OVN logical port
POD_NAME="my-pod"
POD_NS="default"
POD_IP=$(kubectl get pod $POD_NAME -n $POD_NS -o jsonpath='{.status.podIPs[*].ip}')
echo "Pod IPs: $POD_IP"

# Check OVN northbound for the pod's port
ovn-nbctl lsp-list ovn-worker | grep "$POD_IP"

# Check OVS flows for IPv6
ovs-ofctl dump-flows br-int | grep "ipv6"

# Test IPv6 connectivity between pods
kubectl exec -it pod-a -- ping6 fd00:10:244::5

# OVN-Kubernetes logs
kubectl logs -n ovn-kubernetes -l app=ovnkube-node --tail=50 | grep -i ipv6
```

## Conclusion

OVN-Kubernetes provides a full SDN with native IPv6 support. Configure dual-stack by setting both `v4cidr`/`v6cidr` in the ConfigMap. OVN logical switches, routers, and load balancers all support IPv6 addresses. Standard Kubernetes NetworkPolicy with IPv6 `ipBlock` CIDRs works natively. Monitor OVN-Kubernetes agent health, OVN northbound database status, and OVS flow tables with OneUptime.
