# How to Configure Windows Networking in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Window, Networking, Flannel, CNI

Description: Configure networking for Windows nodes in Rancher Kubernetes clusters including CNI selection, network policies, DNS resolution, and service connectivity.

## Introduction

Windows networking in Kubernetes has specific constraints compared to Linux. Not all CNI plugins support Windows, DNS resolution behaves differently, and network policies have limited support. This guide covers the networking setup required for Windows nodes in Rancher clusters.

## Prerequisites

- Rancher cluster with Linux control plane and Windows workers
- Windows Server 2019 or 2022 nodes
- Understanding of Kubernetes networking basics

## Step 1: Choose a Windows-Compatible CNI

```bash
# Supported CNI plugins for Windows nodes (as of 2026):

# 1. Flannel (vxlan or host-gw) - Most compatible, simplest
# 2. Calico - Supports Windows nodes with limited policy support
# 3. Antrea - Good Windows support

# For RKE2 clusters with Windows, Flannel is recommended
# Check current CNI
kubectl get configmap rke2-cfg -n kube-system -o yaml

# RKE2 config for Windows-compatible Flannel setup
# /etc/rancher/rke2/config.yaml on Linux control plane
cni: flannel
flannel-backend: vxlan  # vxlan works across different subnets
```

## Step 2: Verify Windows Node Networking

```powershell
# On Windows node - verify network setup after joining cluster

# Check HNS (Host Network Service) networks
Get-HNSNetwork

# Verify flannel vxlan adapter created
Get-NetAdapter | Where-Object {$_.Name -like "*flannel*"}

# Check if pod CIDR is assigned
ipconfig /all | findstr "IPv4"

# Verify DNS resolution inside pods
kubectl run test-win --image=mcr.microsoft.com/windows/nanoserver:ltsc2022 `
  --overrides='{"spec":{"nodeSelector":{"kubernetes.io/os":"windows"}}}' `
  --rm -it -- powershell.exe -Command "Resolve-DnsName kubernetes.default"
```

## Step 3: Configure Windows Node Network Policies

```yaml
# Windows supports limited network policy (requires Antrea or Calico)
# Basic deny-all then allow specific traffic

# Windows NetworkPolicy example
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: windows-app-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: windows-app
      # Must also have Windows node selector to match Windows pods
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow from ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Allow DNS
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Allow HTTPS to external services
    - ports:
        - protocol: TCP
          port: 443
```

## Step 4: Configure DNS for Windows Containers

```powershell
# Windows containers use different DNS configuration

# Verify DNS in running Windows pod
kubectl exec -it win-pod -n production -- powershell.exe

# Inside container:
Get-DnsClientServerAddress
Resolve-DnsName kubernetes.default.svc.cluster.local
Resolve-DnsName my-service.production.svc.cluster.local

# If DNS resolution fails, check CoreDNS
kubectl get configmap coredns -n kube-system -o yaml
```

```yaml
# coredns-windows-fix.yaml - Ensure CoreDNS handles Windows DNS correctly
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

## Step 5: Service Connectivity from Windows Pods

```powershell
# Test connectivity from Windows pod to Linux service

# Exec into Windows pod
kubectl exec -it \
  $(kubectl get pod -l app=windows-app -o name | head -1) \
  -n production \
  -- powershell.exe

# Test ClusterIP service connectivity
Invoke-WebRequest -Uri "http://linux-service.production.svc.cluster.local:8080/health"

# Test NodePort service
$nodeIP = "10.0.0.21"  # IP of a Linux worker node
Invoke-WebRequest -Uri "http://${nodeIP}:30080/health"

# Test external DNS
Invoke-WebRequest -Uri "https://api.example.com/health"
```

## Step 6: Configure Host Network for Windows Pods

```yaml
# For Windows pods that need direct host network access
apiVersion: apps/v1
kind: Deployment
metadata:
  name: win-host-network-app
  namespace: production
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      # Windows supports hostNetwork but with limitations
      hostNetwork: false  # Keep false for most workloads
      containers:
        - name: app
          image: registry.example.com/windows-app:v1.0
          # Configure specific host ports if needed
          ports:
            - containerPort: 8080
              hostPort: 8080  # Alternative to hostNetwork
```

## Step 7: Troubleshoot Windows Networking

```powershell
# Common Windows networking debugging steps

# Check HNS policy (load balancing rules for Services)
Get-HnsPolicyList

# Check if service IPs are reachable
Test-NetConnection -ComputerName "10.96.0.1" -Port 443  # Kubernetes API
Test-NetConnection -ComputerName "10.96.0.10" -Port 53  # CoreDNS

# Check Windows firewall rules for Kubernetes
Get-NetFirewallRule | Where-Object {$_.DisplayName -like "*kube*"}

# Verify network adapter configuration
Get-NetIPConfiguration | Where-Object {$_.InterfaceAlias -like "*vEthernet*"}

# Capture network traffic (requires Npcap/WinPcap)
netsh trace start capture=yes maxsize=100 tracefile=C:\trace.etl
# ... reproduce issue ...
netsh trace stop
```

## Conclusion

Windows networking in Kubernetes requires careful CNI selection-Flannel with vxlan backend provides the broadest compatibility. DNS resolution and service discovery work similarly to Linux once CoreDNS is properly configured, but Windows containers need additional time for network initialization. Test connectivity thoroughly after adding Windows nodes, particularly cross-OS service communication between Linux and Windows pods. Network policies have limited support on Windows but basic ingress/egress rules work with Antrea and Calico.
