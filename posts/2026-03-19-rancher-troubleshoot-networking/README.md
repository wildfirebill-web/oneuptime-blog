# How to Troubleshoot Networking Issues in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Network Policies

Description: A comprehensive troubleshooting guide for diagnosing and resolving common networking issues in Rancher-managed Kubernetes clusters.

Networking issues in Kubernetes can be complex, involving multiple layers from pods and services to ingress controllers and external load balancers. This guide provides a systematic approach to diagnosing and resolving the most common networking problems in Rancher-managed clusters.

## Prerequisites

- A running Rancher instance
- kubectl access to the affected cluster
- Basic understanding of Kubernetes networking concepts

## Step 1: Diagnose Pod-to-Pod Connectivity

Start by testing basic pod-to-pod communication:

```bash
# Create a debug pod
kubectl run netshoot --image=nicolaka/netshoot --rm -it -- bash

# From inside the debug pod, test connectivity to another pod
ping <POD_IP>
curl http://<POD_IP>:<PORT>
traceroute <POD_IP>
```

If pod-to-pod communication fails, check the CNI plugin:

```bash
kubectl get pods -n kube-system | grep -E "calico|canal|flannel|cilium"
kubectl logs -n kube-system -l k8s-app=canal --tail=50
```

## Step 2: Diagnose Service Connectivity

Test service resolution and connectivity:

```bash
kubectl run netshoot --image=nicolaka/netshoot --rm -it -- bash

# Test DNS resolution
nslookup my-service.default.svc.cluster.local
dig my-service.default.svc.cluster.local

# Test service connectivity
curl http://my-service.default.svc.cluster.local
wget -qO- --timeout=5 http://my-service
```

Check service endpoints:

```bash
kubectl get endpoints my-service -n default
kubectl describe svc my-service -n default
```

If there are no endpoints, the service selector does not match any running pods:

```bash
kubectl get pods --show-labels -n default
kubectl get svc my-service -n default -o jsonpath='{.spec.selector}'
```

## Step 3: Diagnose DNS Issues

Check CoreDNS is running:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

Test DNS from a pod:

```bash
kubectl run dns-test --image=busybox:1.36 --rm -it -- nslookup kubernetes.default
```

If DNS fails, check the CoreDNS ConfigMap:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Verify the DNS service IP:

```bash
kubectl get svc kube-dns -n kube-system
```

Check that pods have the correct DNS configuration:

```bash
kubectl exec <pod-name> -- cat /etc/resolv.conf
```

## Step 4: Diagnose Ingress Issues

Check the ingress controller is running:

```bash
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=100
```

Verify the Ingress resource:

```bash
kubectl get ingress -n default
kubectl describe ingress my-ingress -n default
```

Test the ingress from outside:

```bash
curl -v -H "Host: myapp.example.com" http://<INGRESS_IP>/
```

Check for configuration errors in the NGINX config:

```bash
kubectl exec -n ingress-nginx <pod-name> -- nginx -T | grep -A 10 "myapp.example.com"
```

## Step 5: Diagnose Network Policy Issues

List active network policies:

```bash
kubectl get networkpolicies --all-namespaces
kubectl describe networkpolicy <policy-name> -n <namespace>
```

Temporarily remove all network policies to test if they are the cause:

```bash
# Save existing policies first
kubectl get networkpolicies -n <namespace> -o yaml > backup-policies.yaml

# Delete policies (be cautious in production)
kubectl delete networkpolicies --all -n <namespace>
```

If connectivity works after removing policies, review the policy rules for the correct selectors and ports.

## Step 6: Check Node Networking

Verify node connectivity:

```bash
kubectl get nodes -o wide
```

Test connectivity between nodes:

```bash
# SSH to a node and ping another node
ping <OTHER_NODE_IP>

# Check node ports are open
nc -zv <NODE_IP> 30080
```

Check for iptables rules that might block traffic:

```bash
# On a node (via SSH)
iptables -L -n | grep -i drop
iptables -L -n -t nat | head -50
```

## Step 7: Diagnose Load Balancer Issues

For LoadBalancer services stuck in Pending:

```bash
kubectl describe svc <service-name> -n <namespace>
kubectl get events --field-selector involvedObject.name=<service-name>
```

Common causes:
- Cloud provider controller not running
- No available IPs in MetalLB pool
- Cloud provider quota exceeded
- Missing IAM permissions

## Step 8: Check Pod Network Configuration

Inspect a pod's network setup:

```bash
kubectl exec <pod-name> -- ip addr
kubectl exec <pod-name> -- ip route
kubectl exec <pod-name> -- cat /etc/resolv.conf
kubectl exec <pod-name> -- ss -tlnp
```

## Step 9: Common Issues and Solutions

**Problem: Pods cannot resolve external DNS**

```bash
# Check CoreDNS forward configuration
kubectl get configmap coredns -n kube-system -o yaml | grep forward

# Verify upstream DNS is reachable
kubectl run test --image=busybox --rm -it -- nslookup google.com
```

**Problem: Intermittent connection failures**

```bash
# Check for pod restarts
kubectl get pods -n <namespace> --sort-by='.status.containerStatuses[0].restartCount'

# Check node resource pressure
kubectl describe node <node-name> | grep -A 5 Conditions

# Check endpoint changes
kubectl get events --field-selector reason=EndpointsUpdated
```

**Problem: Connection timeouts between services**

```bash
# Check if the target pod is ready
kubectl get pods -l app=<target-app> -o wide

# Verify readiness probes
kubectl describe pod <pod-name> | grep -A 5 Readiness

# Test TCP connectivity
kubectl run test --image=nicolaka/netshoot --rm -it -- \
  bash -c "timeout 5 bash -c 'echo > /dev/tcp/<SERVICE_IP>/<PORT>' && echo 'open' || echo 'closed'"
```

## Step 10: Collect Diagnostic Information

Gather comprehensive networking diagnostics:

```bash
# Cluster info
kubectl cluster-info dump --output-directory=/tmp/cluster-dump

# CNI configuration
kubectl get configmap -n kube-system | grep -i cni
kubectl get ds -n kube-system

# All services and endpoints
kubectl get svc --all-namespaces
kubectl get endpoints --all-namespaces

# Network policies
kubectl get networkpolicies --all-namespaces

# Events related to networking
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | grep -i -E "network|dns|ingress|service"
```

## Troubleshooting Checklist

1. Can pods ping each other by IP?
2. Can pods resolve DNS names?
3. Do services have endpoints?
4. Do service selectors match pod labels?
5. Are network policies blocking traffic?
6. Is the ingress controller running?
7. Are node firewalls allowing traffic?
8. Is the CNI plugin healthy?
9. Are readiness probes passing?
10. Are there resource constraints causing throttling?

## Summary

Networking troubleshooting in Rancher-managed Kubernetes clusters requires a systematic approach, starting from basic pod connectivity and working up through services, DNS, ingress, and network policies. Using debug pods with networking tools, checking component logs, and verifying configurations at each layer will help you quickly identify and resolve issues. Keep this guide as a reference for diagnosing the most common networking problems in your Rancher environment.
