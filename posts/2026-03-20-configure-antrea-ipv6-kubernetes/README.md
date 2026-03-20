# How to Configure Antrea for IPv6 in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Antrea, IPv6, Kubernetes, CNI, OVS, Dual-Stack, NetworkPolicy

Description: Configure Antrea CNI for IPv6 and dual-stack Kubernetes clusters using Open vSwitch (OVS) datapath, including IPAM, AntreaNetworkPolicy, and ClusterNetworkPolicy for IPv6.

## Introduction

Antrea is a CNI plugin built on Open vSwitch (OVS) that supports IPv6, dual-stack, and advanced network policies. It provides native IPv6 support for Kubernetes pods and services with OVS-accelerated forwarding.

## Step 1: Install Antrea with IPv6

```bash
# Install Antrea with dual-stack support

kubectl apply -f https://github.com/antrea-io/antrea/releases/latest/download/antrea.yml

# Or with Helm
helm repo add antrea https://charts.antrea.io
helm install antrea antrea/antrea \
    --namespace kube-system \
    --set featureGates.IPv6=true \
    --set featureGates.AntreaPolicy=true
```

## Step 2: Antrea ConfigMap for IPv6

```yaml
# antrea-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: antrea-config
  namespace: kube-system
data:
  antrea-agent.conf: |
    # Enable IPv6
    featureGates:
      IPv6: true
      AntreaPolicy: true
      AntreaProxy: true

    # Tunnel type (VXLAN supports IPv6)
    tunnelType: vxlan

    # Service CIDR (for kube-proxy replacement)
    serviceCIDR: 10.96.0.0/12
    serviceCIDRv6: fd00:10:96::/108

  antrea-controller.conf: |
    featureGates:
      AntreaPolicy: true
      Traceflow: true
```

## Step 3: Cluster Setup for Dual-Stack

```yaml
# kubeadm-config.yaml (cluster init)
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16,fd00:10:244::/48"
  serviceSubnet: "10.96.0.0/12,fd00:10:96::/108"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    node-ip: "192.168.1.1,2001:db8::node1"
```

```bash
# Initialize with dual-stack
kubeadm init --config kubeadm-config.yaml
```

## Step 4: AntreaNetworkPolicy for IPv6

```yaml
# antrea-policy-ipv6.yaml
apiVersion: crd.antrea.io/v1beta1
kind: AntreaNetworkPolicy
metadata:
  name: allow-ipv6-https
  namespace: production
spec:
  priority: 5
  podSelector:
    matchLabels:
      app: web
  ingress:
    # Allow HTTPS from IPv6 clients
    - action: Allow
      from:
        - ipBlock:
            cidr: 2001:db8:client::/48
      ports:
        - protocol: TCP
          port: 443
    # Allow from pods in same namespace
    - action: Allow
      from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production
  egress:
    - action: Allow
```

## Step 5: ClusterNetworkPolicy for IPv6 Egress

```yaml
# antrea-cluster-policy-ipv6.yaml
apiVersion: crd.antrea.io/v1beta1
kind: ClusterNetworkPolicy
metadata:
  name: restrict-ipv6-link-local
spec:
  priority: 1
  appliedTo:
    - podSelector: {}     # All pods
  ingress:
    # Drop link-local source (anti-spoofing)
    - action: Drop
      from:
        - ipBlock:
            cidr: "fe80::/10"
  egress:
    # Allow global IPv6
    - action: Allow
      to:
        - ipBlock:
            cidr: "::/0"
            except:
              - "fe80::/10"
              - "::/128"
```

## Step 6: Verify and Traceflow

```bash
# Check Antrea agents
kubectl get pods -n kube-system -l app=antrea

# Verify dual-stack pod IPs
kubectl get pods -o wide
# Pods should show both IPv4 and IPv6 addresses

# Test IPv6 connectivity
kubectl exec -it frontend-pod -- curl -6 http://[fd00:10:96::10]:80/

# Antrea Traceflow for IPv6
kubectl apply -f - << 'EOF'
apiVersion: crd.antrea.io/v1alpha1
kind: Traceflow
metadata:
  name: ipv6-trace
spec:
  source:
    namespace: default
    pod: frontend-pod
  destination:
    namespace: default
    pod: backend-pod
  ipHeader:
    protocol: 58  # ICMPv6
    ipv6Header:
      nextHeader: 58
EOF

kubectl get traceflow ipv6-trace -o yaml
```

## Conclusion

Antrea's OVS datapath supports IPv6 with the `IPv6: true` feature gate. Configure dual-stack by specifying both IPv4 and IPv6 pod CIDRs. AntreaNetworkPolicy and ClusterNetworkPolicy accept IPv6 CIDR blocks for fine-grained traffic control. Use Traceflow to debug IPv6 connectivity issues between pods. Monitor Antrea agent health with OneUptime's infrastructure monitors.
