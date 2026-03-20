# How to Debug Kubernetes Service Connectivity Issues in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Networking, Services, Troubleshooting

Description: Diagnose and resolve Kubernetes service connectivity problems using Portainer's terminal and network inspection tools.

---

Service connectivity failures in Kubernetes occur when pods cannot reach other services by name, when port mappings are wrong, or when network policies block traffic. Portainer's terminal access lets you run connectivity tests from inside the cluster.

## Common Connectivity Issues

| Symptom | Likely Cause |
|---|---|
| DNS name not resolving | CoreDNS issue or wrong service name |
| Connection refused | Service port mismatch or pod not running |
| Connection timeout | Network policy blocking traffic |
| Intermittent failures | Not all pod replicas healthy |

## Step 1: Verify the Service Exists

In Portainer, go to **Kubernetes > Namespaces > [namespace] > Services** and confirm:

- Service name matches what your application is using
- Target port matches the container's port
- Selector labels match the pod's labels

\`\`\`bash
kubectl get svc -n production
kubectl describe svc my-service -n production
\`\`\`

## Step 2: Test DNS Resolution from a Pod

Use Portainer's terminal to exec into a running pod and test DNS:

\`\`\`bash
# Test DNS resolution
nslookup my-service
nslookup my-service.production.svc.cluster.local

# Test connectivity
curl -v http://my-service:8080/health
nc -zv my-service 8080
\`\`\`

## Step 3: Check Endpoint Readiness

A service with no endpoints means the selector labels don't match any running pods:

\`\`\`bash
# Check endpoints — empty means no matching pods
kubectl get endpoints my-service -n production

# Compare selector labels
kubectl get svc my-service -n production -o yaml | grep -A 5 selector
kubectl get pods -n production --show-labels
\`\`\`

## Step 4: Test Network Policies

If network policies are present, they may block legitimate traffic:

\`\`\`bash
# List network policies in the namespace
kubectl get networkpolicies -n production

# Run a connectivity test from the source pod namespace
kubectl exec -n production <source-pod> -- curl http://target-service:8080
\`\`\`

## Step 5: Check Service Type

For external access issues, verify the Service type:

- \`ClusterIP\`: Only reachable inside the cluster
- \`NodePort\`: Reachable on node IP + NodePort
- \`LoadBalancer\`: Gets an external IP from cloud provider

## Summary

Kubernetes service connectivity debugging follows a consistent path: verify the service exists and has the right port, confirm DNS resolves correctly, check that endpoints exist, and inspect network policies. Portainer's terminal access makes each of these steps executable without leaving the browser.
