# How to Configure RKE2 Networking with Canal

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Canal, Flannel, Calico, Networking, CNI, Kubernetes, Network Policy

Description: Learn how to configure Canal as the CNI plugin in RKE2, combining Flannel for pod networking with Calico for NetworkPolicy enforcement in a simple, compatible setup.

---

Canal is the default CNI for RKE2. It combines Flannel (for pod-to-pod networking via VXLAN) with Calico (for NetworkPolicy enforcement). It is the most compatible option for environments where you need basic network policy without complex BGP routing.

---

## Step 1: Configure RKE2 to Use Canal

Canal is the default, but you can explicitly configure it:

```yaml
# /etc/rancher/rke2/config.yaml
token: my-cluster-token
tls-san:
  - "rke2.example.com"

# Explicitly select Canal as the CNI
cni: canal

# Pod network CIDR (Canal/Flannel uses this for pod IPs)
cluster-cidr: 10.42.0.0/16

# Service network CIDR
service-cidr: 10.43.0.0/16
```

---

## Step 2: Verify Canal Is Running

After cluster initialization:

```bash
# Check that all Canal pods are running
kubectl get pods -n kube-system -l k8s-app=canal

# Verify pod networking is functional
kubectl run test-pod --image=busybox --rm -it -- ping -c 3 8.8.8.8
```

---

## Step 3: Configure Flannel Backend

Canal supports multiple Flannel backends. VXLAN is the default and works over standard networking. For direct routing (lower overhead), use `host-gw`:

```yaml
# rke2-canal-config.yaml (HelmChartConfig)
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-canal
  namespace: kube-system
spec:
  valuesContent: |-
    flannel:
      backend: "vxlan"   # vxlan (default) or host-gw or wireguard
    calico:
      # Enable IPIP for cross-subnet routing (alternative to VXLAN)
      encapsulation: "None"
```

Apply this before or shortly after cluster initialization.

---

## Step 4: Test NetworkPolicy Enforcement (Calico Component)

Canal's Calico component enforces NetworkPolicies. Test it by deploying a policy:

```yaml
# deny-all-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: test-ns
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

```bash
kubectl create namespace test-ns
kubectl apply -f deny-all-ingress.yaml

# Test: this should fail (denied by policy)
kubectl run client --image=busybox --rm -it -n test-ns \
  -- wget -qO- http://my-service.test-ns.svc.cluster.local
```

---

## Step 5: Configure MTU

For environments with jumbo frames or VXLAN overhead, tune the MTU:

```yaml
# In the HelmChartConfig for rke2-canal
spec:
  valuesContent: |-
    flannel:
      mtu: 1450   # reduce from 1500 to account for VXLAN overhead
```

---

## When to Choose Canal vs. Other CNIs

| Requirement | Recommended CNI |
|---|---|
| Simple setup, NetworkPolicy support | Canal (default) |
| BGP routing, advanced policy | Calico |
| eBPF, high performance, observability | Cilium |
| Air-gapped, simple pod networking only | Flannel |

---

## Best Practices

- Set the MTU explicitly to avoid packet fragmentation in cloud environments.
- Do not mix Canal with other CNI installations — only one CNI should be active per cluster.
- Use `host-gw` backend when all nodes are in the same L2 network for better performance than VXLAN.
