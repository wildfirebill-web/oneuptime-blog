# How to Configure kube-proxy in iptables Mode for IPv4 Service Routing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Kube-proxy, iptables, IPv4, Service Routing, Networking

Description: Configure and understand kube-proxy's iptables mode for IPv4 Kubernetes service routing, including how to tune it for performance and debug rule issues.

kube-proxy in iptables mode is the default service proxy in Kubernetes. It programs iptables DNAT rules to intercept ClusterIP traffic and forward it to pod endpoints.

## How iptables Mode Works

```text
Pod sends to ClusterIP 10.96.45.123:80
→ iptables KUBE-SERVICES chain intercepts
→ KUBE-SVC-xxx chain selects random endpoint
→ KUBE-SEP-xxx chain DNATs to pod IP:port
→ Packet delivered to pod
```

## Verifying iptables Mode is Active

```bash
# Check kube-proxy configuration

kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
# Expected: mode: "iptables" (or empty, which defaults to iptables)

# Or check the running proxy mode
kubectl logs -n kube-system $(kubectl get pods -n kube-system -l k8s-app=kube-proxy -o name | head -1) \
  | grep -i "using.*mode\|proxy mode"
# Expected: "Using iptables Proxier"
```

## Configuring iptables Mode Explicitly

```yaml
# kube-proxy-config.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "iptables"
# Sync interval for iptables rules
iptables:
  # Minimum interval between syncs (default: 30s)
  minSyncPeriod: 10s
  # Maximum interval between syncs
  syncPeriod: 60s
  # Mask for iptables masquerade rules
  masqueradeBit: 14
  masqueradeAll: false
```

Apply as a ConfigMap:

```bash
kubectl edit configmap kube-proxy -n kube-system
# Update mode and iptables settings, then restart kube-proxy
kubectl rollout restart daemonset/kube-proxy -n kube-system
```

## Viewing kube-proxy iptables Rules

```bash
# Main service routing chains
sudo iptables -t nat -L KUBE-SERVICES -n

# View a specific service's chain (get chain name from above)
sudo iptables -t nat -L KUBE-SVC-XXXXXXXXXXXX -n

# View endpoint chains (DNAT rules)
sudo iptables -t nat -L KUBE-SEP-XXXXXXXXXXXX -n

# Count all kube-proxy rules
sudo iptables-save | grep -c "^-"

# View statistics on a rule (how many packets matched)
sudo iptables -t nat -L KUBE-SERVICES -n -v
```

## Performance Considerations

In large clusters (>1000 services), iptables mode has scalability limits:

```bash
# Check how many iptables rules exist (warning: >10K can cause latency)
sudo iptables -t nat -L | wc -l

# Check kube-proxy sync time
kubectl logs -n kube-system <kube-proxy-pod> | grep -i "sync\|latency"
```

## Tuning iptables Synchronization

```yaml
# In kube-proxy ConfigMap, reduce sync frequency for large clusters
iptables:
  # Reduce to decrease CPU overhead of frequent syncs
  syncPeriod: 120s
  minSyncPeriod: 30s
```

## Cleaning Up Stale Rules

```bash
# kube-proxy normally cleans up stale rules automatically
# To force a full resync:
kubectl rollout restart daemonset/kube-proxy -n kube-system

# If kube-proxy is broken and you need emergency cleanup (DESTRUCTIVE):
# sudo iptables-restore < /dev/null  # DO NOT do this in production
```

For clusters with more than 1000 services or pods, consider migrating to IPVS mode for better performance.
