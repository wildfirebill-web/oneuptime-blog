# How to Debug Kubernetes Service Connectivity Issues in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Networking, Services, Troubleshooting

Description: Diagnose and resolve Kubernetes service connectivity problems using Portainer's web interface and debugging techniques.

## Introduction

Service connectivity issues in Kubernetes are common and can manifest as connection timeouts, DNS resolution failures, or load balancing problems. Portainer provides visibility into services and their endpoints, helping diagnose network issues.

## Step 1: Check Service Configuration in Portainer

Navigate to: **Kubernetes > Services**

Verify:
- Service exists in the correct namespace
- Service type (ClusterIP, NodePort, LoadBalancer)
- Port mapping is correct
- Label selector matches pod labels

## Step 2: Verify Pod Labels Match Service Selector

```bash
# The most common mistake: selector doesn't match pod labels
kubectl get service myapp -n production -o yaml | grep -A5 selector

# Check pod labels
kubectl get pods -n production --show-labels | grep myapp

# Example mismatch:
# Service selector: app=myapp
# Pod label: app=my-app  (hyphen vs no-hyphen!)

# Fix by updating deployment labels
kubectl patch deployment myapp -n production -p '{
  "spec": {"template": {"metadata": {"labels": {"app": "myapp"}}}}
}'
```

## Step 3: Check Service Endpoints

```bash
# Check if the service has endpoints (pods backing it)
kubectl get endpoints myapp -n production

# If ENDPOINTS shows "<none>", the selector doesn't match any pods
# Expected: ENDPOINTS = 10.244.1.5:8080,10.244.2.3:8080,10.244.3.7:8080

# Describe endpoints for details
kubectl describe endpoints myapp -n production
```

## Step 4: Test Connectivity from Inside the Cluster

```bash
# Deploy a debug pod
kubectl run debug --rm -it --image=nicolaka/netshoot -n production -- bash

# Inside the debug pod:
# Test by service name
curl http://myapp.production.svc.cluster.local:8080/health

# Test by ClusterIP
curl http://10.96.45.123:8080/health

# DNS lookup
nslookup myapp.production.svc.cluster.local

# Check if port is open
nc -zv myapp 8080
nc -zv myapp.production.svc.cluster.local 8080
```

## Step 5: Check kube-proxy and CoreDNS

```bash
# Check kube-proxy is running
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Check CoreDNS is running  
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS resolution
kubectl run dnstest --rm -it --image=busybox -- nslookup kubernetes.default
```

## Common Issues and Fixes

```bash
# Issue: Service IP not responding
# Check if service has ClusterIP or is headless
kubectl get service myapp -n production
# ClusterIP: None means headless (no load balancing, returns pod IPs directly)

# Issue: LoadBalancer service has no external IP
kubectl get service myapp -n production
# If EXTERNAL-IP is <pending>, check cloud provider or MetalLB configuration

# Issue: NodePort not accessible externally
# Check firewall rules allow the NodePort (30000-32767 range)
# Verify node's firewall
sudo iptables -L -n | grep 30080

# Issue: Slow first connection
# DNS lookup timeout - check CoreDNS
kubectl get configmap coredns -n kube-system -o yaml
```

## Network Policy Blocking Traffic

```bash
# Check if a NetworkPolicy is blocking traffic
kubectl get networkpolicies -n production

# Test if you can reach the service from a pod that should have access
kubectl exec myapp-frontend-pod -n production -- curl myapp-backend:8080/health

# If blocked, check NetworkPolicy rules:
kubectl describe networkpolicy allow-frontend-to-backend -n production
```

## Portainer Service Diagnostics

```bash
# Via Portainer API: get service details
curl -s \
  -H "X-API-Key: your-api-key" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/api/v1/namespaces/production/services/myapp" \
  | python3 -c "
import sys, json
svc = json.load(sys.stdin)
spec = svc['spec']
print(f'Type: {spec[\"type\"]}')
print(f'ClusterIP: {spec.get(\"clusterIP\")}')
print(f'Selector: {spec.get(\"selector\")}')
for port in spec.get('ports', []):
    print(f'Port: {port[\"port\"]} -> {port[\"targetPort\"]}')
"
```

## Conclusion

Kubernetes service connectivity debugging follows a systematic path: verify service exists and has correct selector, check endpoints are populated, test connectivity from inside the cluster, and verify network policies. Portainer's Services view provides a quick overview, while debug pods deployed via Portainer enable hands-on network testing from within the cluster network.
