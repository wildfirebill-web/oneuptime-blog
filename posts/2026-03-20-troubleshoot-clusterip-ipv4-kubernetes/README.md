# How to Troubleshoot ClusterIP Service IPv4 Connectivity in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, ClusterIP, IPv4, Troubleshooting, kube-proxy, Networking

Description: Diagnose and fix common IPv4 ClusterIP service connectivity failures in Kubernetes including iptables issues, DNS problems, and endpoint mismatches.

ClusterIP services provide stable IPv4 addresses for pod-to-pod communication within a cluster. When they stop working, traffic fails silently — no TCP reset, just timeouts.

## Step 1: Verify the Service and Endpoints

```bash
# Check the service exists and has a ClusterIP
kubectl get svc my-service -n my-namespace
# NAME         TYPE        CLUSTER-IP      PORT(S)   AGE
# my-service   ClusterIP   10.96.45.123    80/TCP    10m

# CRITICAL: Check endpoints — this is the most common issue
kubectl get endpoints my-service -n my-namespace
# NAME         ENDPOINTS           AGE
# my-service   10.244.1.5:8080    10m   ← Pods are running
# my-service   <none>             10m   ← NO PODS MATCH THE SELECTOR!
```

If endpoints show `<none>`, the service selector doesn't match any running pods.

## Step 2: Verify the Selector Matches Pod Labels

```bash
# Get the service selector
kubectl get svc my-service -n my-namespace -o jsonpath='{.spec.selector}'
# {"app":"my-app","version":"v1"}

# Find pods matching this selector
kubectl get pods -n my-namespace -l "app=my-app,version=v1"
# If no pods show, the labels don't match — check pod labels
kubectl get pod my-pod -n my-namespace --show-labels
```

## Step 3: Test ClusterIP Connectivity from Within the Cluster

```bash
# Deploy a debug pod
kubectl run debug --image=alpine --restart=Never -- sleep 3600

# Try connecting to the service ClusterIP directly
kubectl exec debug -- wget -qO- --timeout=5 http://10.96.45.123:80

# Try by DNS name (tests CoreDNS + ClusterIP)
kubectl exec debug -- wget -qO- --timeout=5 http://my-service.my-namespace.svc.cluster.local
```

## Step 4: Verify kube-proxy is Running

```bash
# Check kube-proxy pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Check kube-proxy logs for errors
kubectl logs -n kube-system $(kubectl get pods -n kube-system -l k8s-app=kube-proxy -o name | head -1)
# Look for: "error syncing rules", "Failed to update iptables rules"
```

## Step 5: Verify iptables Rules on a Node

```bash
# SSH to a worker node and check iptables rules for the ClusterIP
sudo iptables -t nat -L KUBE-SERVICES -n | grep 10.96.45.123

# Expected:
# KUBE-SVC-xxxx  tcp -- 0.0.0.0/0  10.96.45.123  tcp dpt:80

# If no entry, kube-proxy hasn't synced this service
```

## Step 6: Check DNS Resolution

```bash
# DNS failure looks like ClusterIP failure but is a different root cause
kubectl exec debug -- nslookup my-service.my-namespace.svc.cluster.local

# Expected: server: 10.96.0.10 (CoreDNS), Address: 10.96.45.123
# If NXDOMAIN: DNS configuration issue — check CoreDNS
```

## Common Causes and Fixes

| Symptom | Cause | Fix |
|---|---|---|
| Endpoints = `<none>` | Selector mismatch | Fix pod labels or service selector |
| Connection refused | Pod not listening on expected port | Check containerPort |
| Timeout | kube-proxy not synced | Restart kube-proxy |
| DNS failure | CoreDNS issue | Check CoreDNS pods |
| Works between namespaces sometimes | NetworkPolicy | Check for deny policies |

## Quick Reset

```bash
# If kube-proxy rules are corrupted, restarting kube-proxy fixes it
kubectl rollout restart daemonset/kube-proxy -n kube-system
```

The `kubectl get endpoints` check resolves over 80% of ClusterIP connectivity issues by revealing selector mismatches immediately.
