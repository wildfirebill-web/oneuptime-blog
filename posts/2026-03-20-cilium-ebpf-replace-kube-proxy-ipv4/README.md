# How to Replace kube-proxy with Cilium eBPF for IPv4 Service Handling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cilium, eBPF, Kubernetes, IPv4, kube-proxy Replacement, Performance

Description: Remove kube-proxy and configure Cilium's eBPF-based service handling for IPv4 Kubernetes services, achieving better performance and lower latency.

Cilium can completely replace kube-proxy using eBPF programs that intercept and redirect service traffic at the kernel level. This eliminates iptables overhead and provides faster service routing with socket-level load balancing.

## Why Replace kube-proxy?

- **eBPF socket-based load balancing**: connections are redirected before leaving the socket layer, eliminating an entire kernel network stack traversal
- **No iptables rules**: avoids O(n) rule scanning and frequent iptables-save/restore
- **Better observability**: full flow visibility via Hubble
- **Feature richness**: topology-aware routing, DSR, health checking

## Option 1: Install Cilium without kube-proxy (Fresh Cluster)

```bash
# Initialize kubeadm WITHOUT kube-proxy
sudo kubeadm init \
  --pod-network-cidr=10.0.0.0/16 \
  --skip-phases=addon/kube-proxy

# Setup kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Install Cilium with kube-proxy replacement
helm install cilium cilium/cilium \
  --version 1.15.0 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<CONTROL_PLANE_IP> \
  --set k8sServicePort=6443
```

## Option 2: Migrate an Existing Cluster

```bash
# Step 1: Install Cilium in partial replacement mode first
helm install cilium cilium/cilium \
  --version 1.15.0 \
  --namespace kube-system \
  --set kubeProxyReplacement=false

# Wait for Cilium to be ready
kubectl rollout status daemonset/cilium -n kube-system

# Step 2: Upgrade to full kube-proxy replacement
helm upgrade cilium cilium/cilium \
  --version 1.15.0 \
  --namespace kube-system \
  --reuse-values \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<CONTROL_PLANE_IP> \
  --set k8sServicePort=6443

# Step 3: Remove kube-proxy DaemonSet
kubectl delete daemonset kube-proxy -n kube-system

# Step 4: Clean up iptables rules left by kube-proxy
sudo iptables-save | grep -v KUBE | sudo iptables-restore
```

## Verifying kube-proxy Replacement

```bash
# Verify Cilium is handling service routing
cilium status | grep KubeProxyReplacement
# Expected: KubeProxyReplacement: True

# Check socket-based load balancing is active
kubectl exec -n kube-system $(kubectl get pods -n kube-system -l k8s-app=cilium -o name | head -1) \
  -- cilium service list

# Verify no kube-proxy pods remain
kubectl get pods -n kube-system -l k8s-app=kube-proxy
# Expected: No resources found
```

## Testing Service Routing

```bash
# Create a test service
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --port=80

# Connect from a pod
kubectl run test --image=alpine --restart=Never -- sleep 3600
kubectl exec test -- wget -qO- http://nginx

# Check Cilium service entries
kubectl exec -n kube-system <cilium-pod> -- cilium service list | grep nginx
```

## Monitoring with Hubble

```bash
# Observe service traffic flows
cilium hubble port-forward &
hubble observe --to-service default/nginx --follow

# Output shows:
# source-pod → nginx-service (FORWARDED)
# nginx-service → nginx-pod-x (TRANSLATED)
```

Cilium with eBPF kube-proxy replacement delivers measurably lower latency (often 10-30% improvement) for service-heavy workloads by moving load balancing decisions to socket level.
