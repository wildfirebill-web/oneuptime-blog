# How to Configure Weave Net for IPv6 in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, Weave Net, CNI, Dual-Stack, Networking

Description: Configure Weave Net CNI for IPv6 support in Kubernetes, enable dual-stack networking with Weave's mesh overlay, and verify pod IPv6 connectivity with Weave Net routing.

## Introduction

Weave Net is a CNI plugin that creates a mesh overlay network across Kubernetes nodes. Weave Net supports IPv6 through environment variable configuration in the DaemonSet. When IPv6 is enabled, Weave Net creates a mesh that routes both IPv4 and IPv6 pod traffic. Note that Weave Net is now in maintenance mode; for new deployments consider Calico or Cilium which have more active development.

## Install Weave Net with IPv6

```bash
# Generate Weave Net manifest with IPv6 enabled

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# Alternatively, download and modify the manifest
curl -L "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')" \
    -o weave-net.yaml
```

## Configure Weave Net for IPv6

```yaml
# Edit the Weave Net DaemonSet to enable IPv6
# Key environment variables for IPv6:

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: weave-net
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - name: weave
          env:
            # Enable IPv6
            - name: IPALLOC_RANGE
              value: "10.244.0.0/16"
            - name: IPALLOC_INIT
              value: "seed=1234"
            # IPv6 configuration
            - name: IPV6
              value: "1"
            - name: IPALLOC_RANGE_V6
              value: "fd00:weave::/112"
```

```bash
# Apply the modified manifest
kubectl apply -f weave-net.yaml

# Patch the DaemonSet to add IPv6 env vars
kubectl -n kube-system patch daemonset weave-net \
    --type='json' \
    -p='[
        {"op":"add","path":"/spec/template/spec/containers/0/env/-",
         "value":{"name":"IPV6","value":"1"}},
        {"op":"add","path":"/spec/template/spec/containers/0/env/-",
         "value":{"name":"IPALLOC_RANGE_V6","value":"fd00:weave::/112"}}
    ]'

# Restart Weave Net
kubectl -n kube-system rollout restart daemonset weave-net
```

## Verify Weave Net IPv6

```bash
# Check Weave Net pods
kubectl -n kube-system get pods -l name=weave-net

# Inspect Weave Net status
kubectl -n kube-system exec daemonset/weave-net -c weave -- \
    /home/weave/weave --local status

# Check Weave IPAM for IPv6
kubectl -n kube-system exec daemonset/weave-net -c weave -- \
    /home/weave/weave --local status ipam

# Deploy test pods and check IPv6 addresses
kubectl run test1 --image=busybox --command -- sleep infinity
kubectl run test2 --image=busybox --command -- sleep infinity

kubectl get pods test1 test2 -o jsonpath='{range .items[*]}{.metadata.name}: {.status.podIPs}{"\n"}{end}'

# Test connectivity
TEST2_IPV6=$(kubectl get pod test2 -o jsonpath='{.status.podIPs[1].ip}')
kubectl exec test1 -- ping6 -c 3 "$TEST2_IPV6"
```

## Weave Net IPv6 Troubleshooting

```bash
# Check Weave Net logs for IPv6 messages
kubectl -n kube-system logs daemonset/weave-net -c weave | grep -i "ipv6\|IPv6"

# Check Weave network interface on node
ip -6 addr show weave

# Verify IPv6 routes added by Weave
ip -6 route show | grep weave

# Check peers (other nodes in Weave mesh)
kubectl -n kube-system exec daemonset/weave-net -c weave -- \
    /home/weave/weave --local status peers

# Common issue: IPV6=1 env var not set
kubectl -n kube-system get daemonset weave-net -o json | \
    jq '.spec.template.spec.containers[0].env'
```

## Migration from Weave Net to Cilium

```bash
# Since Weave Net is in maintenance mode, migration to Cilium is recommended

# Step 1: Create Cilium manifest without deleting Weave
helm template cilium cilium/cilium \
    --namespace kube-system \
    --set ipv4.enabled=true \
    --set ipv6.enabled=true > cilium.yaml

# Step 2: Drain nodes one at a time and switch CNI
# Step 3: Remove Weave Net after Cilium is fully operational
kubectl -n kube-system delete daemonset weave-net
kubectl delete clusterrole weave-net
kubectl delete clusterrolebinding weave-net
```

## Conclusion

Configure Weave Net for IPv6 by setting the `IPV6=1` and `IPALLOC_RANGE_V6` environment variables in the Weave Net DaemonSet. Weave creates a mesh overlay that routes both IPv4 and IPv6 pod traffic. Verify IPv6 pod addresses and test connectivity with `ping6` between pods. Note that Weave Net is in maintenance mode - for new or migrating clusters, Calico or Cilium provide more active development and richer IPv6 features including network policy enforcement and eBPF acceleration.
