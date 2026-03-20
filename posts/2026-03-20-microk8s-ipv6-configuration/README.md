# How to Configure MicroK8s for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MicroK8s, Kubernetes, IPv6, Dual-Stack, Networking

Description: A step-by-step guide to enabling IPv6 and dual-stack networking in a MicroK8s Kubernetes cluster.

MicroK8s is a lightweight Kubernetes distribution ideal for development, CI/CD, and edge deployments. Starting with version 1.27, it includes built-in support for IPv6 and dual-stack networking through its Calico CNI plugin.

## Prerequisites

- Ubuntu 20.04 or later
- MicroK8s installed (`snap install microk8s --classic`)
- A network interface with an IPv6 address

## Step 1: Verify Host IPv6 Availability

```bash
# Check that the host has a routable IPv6 address

ip -6 addr show

# Verify IPv6 connectivity
ping6 -c 3 ipv6.google.com
```

## Step 2: Enable Required MicroK8s Add-ons

MicroK8s uses Calico for its default CNI. Enable it along with DNS:

```bash
# Enable Calico CNI and CoreDNS
microk8s enable dns
microk8s enable calico
```

## Step 3: Configure Calico for Dual-Stack

Edit the Calico configuration to enable IPv6:

```bash
# Open a shell with microk8s kubectl
microk8s kubectl edit installation default -n calico-system
```

Update the `calicoNetwork` section:

```yaml
# Calico Installation resource - dual-stack configuration
spec:
  calicoNetwork:
    ipPools:
      # IPv4 pool (existing)
      - cidr: 10.1.0.0/16
        encapsulation: VXLAN
        natOutgoing: Enabled
      # IPv6 pool (add this)
      - cidr: fd01::/48
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()
```

## Step 4: Update the MicroK8s API Server Configuration

MicroK8s stores its configuration in `/var/snap/microk8s/current/args/`. Update the kube-apiserver args to enable dual-stack:

```bash
# Append dual-stack service CIDR to API server args
sudo sh -c 'echo "--service-cluster-ip-range=10.152.183.0/24,fd98::/108" >> /var/snap/microk8s/current/args/kube-apiserver'

# Update kube-controller-manager with dual-stack cluster CIDR
sudo sh -c 'echo "--cluster-cidr=10.1.0.0/16,fd01::/48" >> /var/snap/microk8s/current/args/kube-controller-manager'
sudo sh -c 'echo "--service-cluster-ip-range=10.152.183.0/24,fd98::/108" >> /var/snap/microk8s/current/args/kube-controller-manager'
```

## Step 5: Restart MicroK8s

```bash
# Restart MicroK8s to apply the configuration changes
microk8s stop
microk8s start

# Wait for the cluster to be ready
microk8s status --wait-ready
```

## Step 6: Verify Dual-Stack Is Active

```bash
# Check that the kube-dns service has both IPv4 and IPv6 ClusterIPs
microk8s kubectl get svc kube-dns -n kube-system -o yaml | grep clusterIP

# Deploy a test pod and check it gets both address families
microk8s kubectl run test-pod --image=nginx --restart=Never
microk8s kubectl get pod test-pod -o jsonpath='{.status.podIPs}'
```

## Step 7: Test IPv6 Connectivity from a Pod

```bash
# Exec into the test pod
microk8s kubectl exec -it test-pod -- bash

# From inside the pod, test IPv6 external connectivity
curl -6 https://ipv6.google.com
```

## Troubleshooting

If pods do not receive IPv6 addresses, check that the Calico IP pool was created correctly:

```bash
# List Calico IP pools
microk8s kubectl get ippools -n calico-system

# Describe the IPv6 pool for details
microk8s kubectl describe ippool <ipv6-pool-name> -n calico-system
```

MicroK8s makes it straightforward to enable IPv6 by leveraging Calico's mature dual-stack support alongside simple configuration file edits.
