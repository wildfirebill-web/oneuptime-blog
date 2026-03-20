# IPv6 Networking in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, IPv6, Kubernetes, Networking, CNI

Description: Learn how to configure and enable IPv6 networking in Rancher-managed Kubernetes clusters, including dual-stack setup and CNI plugin configuration.

## Overview

Rancher is a Kubernetes management platform that supports deploying clusters with IPv6 and dual-stack networking. Enabling IPv6 in Rancher involves configuring the cluster's CNI plugin, API server flags, and node networking.

## Prerequisites

- Rancher 2.6+ installed
- Kubernetes 1.21+ (for stable dual-stack support)
- Nodes with IPv6 addresses assigned
- A CNI plugin that supports IPv6 (Calico, Cilium, or Canal)

## Step 1: Enable IPv6 on Nodes

Ensure each node has an IPv6 address and that IPv6 forwarding is enabled:

```bash
sudo sysctl -w net.ipv6.conf.all.forwarding=1
echo "net.ipv6.conf.all.forwarding = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 2: Configure the Cluster in Rancher

When creating a new RKE2 cluster in Rancher, configure dual-stack in the cluster YAML:

```yaml
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: my-cluster
spec:
  rkeConfig:
    machineGlobalConfig:
      cluster-cidr: "10.42.0.0/16,fd00:10:244::/56"
      service-cidr: "10.43.0.0/16,fd00:10:96::/112"
      cni: calico
```

## Step 3: Configure Calico for Dual-Stack

Edit the Calico configuration to enable IPv6:

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.42.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
    - blockSize: 122
      cidr: fd00:10:244::/56
      encapsulation: None
      natOutgoing: Disabled
```

## Step 4: Configure Cilium for IPv6 (Alternative)

If using Cilium as the CNI:

```bash
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set ipv6.enabled=true \
  --set tunnel=disabled \
  --set ipam.mode=kubernetes \
  --set k8s.requireIPv4PodCIDR=true \
  --set k8s.requireIPv6PodCIDR=true
```

## Step 5: Verify Dual-Stack Operation

```bash
# Check node addresses

kubectl get nodes -o wide

# Check pod IPv6 addresses
kubectl get pods -o wide -A | head -10

# Test IPv6 connectivity between pods
kubectl run test-pod --image=busybox --rm -it -- ping6 fd00::1

# Verify services get dual-stack ClusterIPs
kubectl get svc -o yaml | grep -A5 "clusterIPs:"
```

## Creating a Dual-Stack Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-dual-stack-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  ipFamilyPolicy: RequireDualStack
  ipFamilies:
  - IPv4
  - IPv6
```

## Troubleshooting

**Pod not getting IPv6 address:**
```bash
kubectl describe pod <pod-name> | grep -i ipv6
kubectl describe node <node-name> | grep -i cidr
```

**IPv6 routing not working:**
```bash
ip -6 route show
ip6tables -L -n
```

**CNI plugin errors:**
```bash
kubectl logs -n kube-system -l k8s-app=calico-node
```

## Best Practices

1. **Use dual-stack** rather than IPv6-only initially to maintain compatibility
2. **Configure firewall rules** for both IPv4 and IPv6 traffic
3. **Test pod-to-pod and service connectivity** over IPv6 explicitly
4. **Monitor IPv6 traffic** in your observability platform
5. **Update Ingress controllers** to listen on both protocol families

## Conclusion

IPv6 networking in Rancher is well-supported through dual-stack cluster configuration and IPv6-capable CNI plugins like Calico and Cilium. Enabling dual-stack allows your Kubernetes workloads to communicate over both protocol families, preparing your infrastructure for the IPv6-first future.
