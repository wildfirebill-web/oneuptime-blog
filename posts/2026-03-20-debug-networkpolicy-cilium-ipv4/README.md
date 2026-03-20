# How to Debug Kubernetes NetworkPolicy Issues with Cilium for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cilium, Kubernetes, NetworkPolicy, IPv4, Hubble, Troubleshooting

Description: Use Cilium CLI, Hubble network observability, and policy trace commands to diagnose IPv4 NetworkPolicy enforcement issues in Kubernetes.

Cilium provides powerful debugging tools through the Cilium CLI and Hubble observability platform. Unlike iptables-based CNIs, Cilium operates at the eBPF level and offers granular visibility into why traffic is allowed or denied.

## Step 1: Verify Cilium Status

```bash
# Check overall Cilium health
cilium status

# Verify all Cilium pods are healthy
kubectl get pods -n kube-system -l k8s-app=cilium
kubectl get pods -n kube-system -l name=cilium-operator

# Run connectivity test
cilium connectivity test
```

## Step 2: Check Endpoint Policy Status

```bash
# List all Cilium endpoints (one per pod)
kubectl exec -n kube-system $(kubectl get pods -n kube-system -l k8s-app=cilium -o name | head -1) \
  -- cilium endpoint list

# Get detailed policy info for a specific endpoint
# First get the endpoint ID (number shown in endpoint list)
kubectl exec -n kube-system <cilium-pod-on-target-node> \
  -- cilium endpoint get <ENDPOINT_ID>

# Check allowed identities and policies
kubectl exec -n kube-system <cilium-pod-on-target-node> \
  -- cilium policy get
```

## Step 3: Use Hubble for Real-Time Flow Observation

```bash
# Enable Hubble if not already enabled
cilium hubble enable
cilium hubble port-forward &

# Install Hubble CLI
# (see https://github.com/cilium/hubble/releases)

# Observe all flows in a namespace
hubble observe --namespace production --follow

# Filter for dropped flows only
hubble observe --namespace production --verdict DROPPED --follow

# Filter by source or destination pod
hubble observe --from-pod production/my-app --follow
hubble observe --to-pod production/database --follow

# Filter by IPv4 address
hubble observe --ip-src 10.244.1.5 --follow
```

## Step 4: Policy Trace

Cilium can simulate what would happen to a specific packet:

```bash
# Trace traffic from pod A to pod B
kubectl exec -n kube-system <cilium-pod> \
  -- cilium policy trace \
  --src-k8s-pod production:my-app \
  --dst-k8s-pod production:database \
  --dport 5432 \
  --protocol tcp

# Expected output shows: ALLOWED or DENIED with reason
# Example:
# Final verdict: ALLOWED
# Resolving egress policy for source endpoint 1234
# Resolving ingress policy for destination endpoint 5678
```

## Step 5: Check Cilium NetworkPolicy Translation

```bash
# View how Kubernetes NetworkPolicies are translated to Cilium policies
kubectl exec -n kube-system <cilium-pod> \
  -- cilium policy get

# View the policy for a specific endpoint
kubectl exec -n kube-system <cilium-pod-on-node> \
  -- cilium endpoint get <ENDPOINT_ID> -o json | \
  jq '.status.policy'
```

## Step 6: View Dropped Packets with eBPF Tracing

```bash
# Monitor dropped packets in real time
kubectl exec -n kube-system <cilium-pod> \
  -- cilium monitor --type drop

# Filter for specific pod
kubectl exec -n kube-system <cilium-pod> \
  -- cilium monitor --related-to <ENDPOINT_ID>

# Output shows:
# xx drop (Policy denied) flow 0x0 to endpoint 1234, ...
# reason: policy-deny
```

## Step 7: Verify Label Selectors

Cilium uses pod labels for identity. Verify the pods have the correct labels:

```bash
# Check pod labels (must match NetworkPolicy podSelector)
kubectl get pod my-app -n production --show-labels

# Check namespace labels (must match namespaceSelector)
kubectl get namespace production --show-labels
```

Cilium's Hubble makes NetworkPolicy debugging dramatically more straightforward than iptables-based approaches by showing exactly which policies allowed or denied each flow.
