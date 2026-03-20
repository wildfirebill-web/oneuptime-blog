# How to Configure RKE2 Networking with Cilium

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Cilium, eBPF, CNI, Networking, Rancher

Description: Learn how to configure Cilium as the CNI plugin for RKE2, leveraging eBPF for high-performance, secure Kubernetes networking.

Cilium is a modern CNI plugin that uses Linux eBPF (extended Berkeley Packet Filter) technology to provide high-performance networking, security, and observability for Kubernetes. It replaces iptables with eBPF programs for load balancing and network policy, resulting in significant performance improvements. This guide covers deploying and configuring Cilium as the CNI in RKE2.

## Prerequisites

- Linux kernel 5.4+ (5.10+ recommended for full eBPF feature set)
- RKE2 v1.21+
- Minimum 4 GB RAM per node for Cilium
- Understanding of eBPF concepts

## Step 1: Install RKE2 with Cilium

```yaml
# /etc/rancher/rke2/config.yaml - Configure Cilium as CNI
cni: cilium

# Since Cilium handles kube-proxy functionality via eBPF,
# you can optionally disable kube-proxy
disable-kube-proxy: true

# Pod and service CIDRs
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
```

```bash
# Install RKE2 with Cilium configuration
# Make sure config.yaml is in place before starting RKE2
curl -sfL https://get.rke2.io | sudo sh -
sudo systemctl enable rke2-server
sudo systemctl start rke2-server

# Monitor Cilium deployment
kubectl get pods -n kube-system | grep cilium
```

## Step 2: Customize Cilium with HelmChartConfig

RKE2 deploys Cilium via Helm. Customize it using HelmChartConfig:

```yaml
# cilium-config.yaml - Advanced Cilium configuration
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-cilium
  namespace: kube-system
spec:
  valuesContent: |-
    # Enable kube-proxy replacement via eBPF
    kubeProxyReplacement: "strict"

    # Kubernetes service host for kube-proxy replacement
    k8sServiceHost: "10.0.0.10"  # Your API server IP/LB
    k8sServicePort: "6443"

    # Enable Hubble for observability
    hubble:
      enabled: true
      relay:
        enabled: true
      ui:
        enabled: true

    # Enable native routing (no overlay for same-subnet traffic)
    nativeRoutingCIDR: "10.0.0.0/8"

    # Auto-direct node routes for intra-cluster routing
    autoDirectNodeRoutes: true

    # Load balancing algorithm
    loadBalancer:
      algorithm: "maglev"  # Options: random, maglev
      mode: "dsr"          # Direct Server Return for better performance

    # MTU (auto-detected if not set)
    # mtu: 1500

    # Enable Prometheus metrics
    prometheus:
      enabled: true
      port: 9962

    # Enable operator Prometheus metrics
    operator:
      prometheus:
        enabled: true
```

## Step 3: Install Hubble CLI

Hubble provides deep visibility into network flows:

```bash
# Install Hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all \
  https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz

tar xzvf hubble-linux-amd64.tar.gz
sudo mv hubble /usr/local/bin/

# Verify Hubble installation
hubble version

# Access Hubble Relay (forward port from Hubble relay)
kubectl port-forward service/hubble-relay \
  -n kube-system 4245:80 &

# Observe network flows
hubble observe --all-namespaces

# Observe flows for a specific pod
hubble observe --pod my-app/my-pod --follow

# Observe HTTP traffic
hubble observe --protocol http --follow
```

## Step 4: Configure Cilium Network Policies

Cilium supports standard Kubernetes NetworkPolicy plus its own CiliumNetworkPolicy:

```yaml
# cilium-network-policy.yaml - Cilium-specific L7 policies
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: l7-policy
  namespace: my-app
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      # Cilium L7 HTTP policy
      rules:
        http:
        # Only allow GET requests to /api/
        - method: "GET"
          path: "/api/"
        # Allow POST requests to /api/data
        - method: "POST"
          path: "/api/data"
```

## Step 5: Configure Cilium for Cluster Mesh

Connect multiple Kubernetes clusters with Cilium Cluster Mesh:

```bash
# Enable cluster mesh on the first cluster
cilium clustermesh enable --service-type LoadBalancer

# Enable cluster mesh on the second cluster
# (run from the second cluster's context)
cilium clustermesh enable --service-type LoadBalancer

# Connect the two clusters
cilium clustermesh connect \
  --destination-context cluster2-context \
  --source-context cluster1-context

# Verify cluster mesh status
cilium clustermesh status
```

## Step 6: Monitor Cilium Health

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
curl -LO https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
tar xzf cilium-linux-amd64.tar.gz
sudo mv cilium /usr/local/bin/

# Check overall Cilium status
cilium status

# Check connectivity between pods
cilium connectivity test

# Check Cilium agents on each node
kubectl get pods -n kube-system -l k8s-app=cilium -o wide

# Check Cilium configuration
kubectl exec -n kube-system \
  $(kubectl get pods -n kube-system -l k8s-app=cilium -o name | head -1) \
  -- cilium status

# View eBPF load balancer entries
kubectl exec -n kube-system \
  $(kubectl get pods -n kube-system -l k8s-app=cilium -o name | head -1) \
  -- cilium service list
```

## Conclusion

Cilium brings eBPF-powered networking to RKE2, delivering superior performance by bypassing the kernel's traditional networking stack. The Hubble observability platform provides deep visibility into network flows that traditional CNI plugins cannot match. For organizations running latency-sensitive workloads or requiring L7 network policies, Cilium's capabilities make it an excellent choice over Canal or Calico. The tradeoff is a higher minimum kernel version requirement and more complex initial configuration.
