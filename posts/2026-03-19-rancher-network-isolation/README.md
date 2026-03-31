# How to Configure Network Isolation Between Projects in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Network Isolation, Project, Security

Description: A step-by-step guide to configuring network isolation between Rancher projects using built-in features and custom NetworkPolicy resources.

Network isolation between projects prevents pods in one project from communicating with pods in another project. This is essential for security, compliance, and preventing accidental cross-project dependencies. Rancher provides both built-in project network isolation and support for custom Kubernetes NetworkPolicy resources. This guide covers both approaches.

## Prerequisites

- Rancher v2.7+ with cluster owner or administrator access
- A CNI plugin that supports NetworkPolicy (Calico, Canal, Cilium, or Weave)
- Multiple projects with workloads
- Understanding of your traffic flow requirements

## Step 1: Verify CNI Support for NetworkPolicy

Not all CNI plugins support NetworkPolicy. Check your cluster's CNI:

```bash
# Check which CNI is installed

kubectl get pods -n kube-system -l k8s-app=canal -o name 2>/dev/null && echo "Canal CNI detected"
kubectl get pods -n kube-system -l k8s-app=calico-node -o name 2>/dev/null && echo "Calico CNI detected"
kubectl get pods -n kube-system -l k8s-app=cilium -o name 2>/dev/null && echo "Cilium CNI detected"
```

If your CNI does not support NetworkPolicy (for example, Flannel without Canal), network policies will have no effect. You will need to switch to a supported CNI.

## Step 2: Enable Project Network Isolation in Rancher

Rancher has a built-in feature to isolate network traffic between projects:

**Via the UI:**

1. Navigate to your cluster.
2. Go to **Cluster > Projects/Namespaces**.
3. Click the three-dot menu on the project you want to isolate.
4. Select **Edit Config**.
5. Enable **Project Network Isolation**.
6. Click **Save**.

Repeat for each project that should be isolated.

**What this does:**

Rancher creates NetworkPolicy resources in each namespace of the project that:

- Allow all traffic between namespaces within the same project
- Deny all traffic from namespaces in other projects
- Allow traffic from the Rancher system namespaces

## Step 3: Create Custom Default-Deny Policies

For more control, create explicit default-deny policies:

```yaml
# Default deny all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: team-a-production
spec:
  podSelector: {}
  policyTypes:
    - Ingress

---
# Default deny all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: team-a-production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

```bash
kubectl apply -f default-deny.yaml
```

With default-deny in place, you must explicitly allow all desired traffic.

## Step 4: Allow Intra-Project Communication

After setting default-deny, allow communication within the project:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-intra-project
  namespace: team-a-production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: team-a
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              project: team-a
```

Label your namespaces consistently:

```bash
kubectl label namespace team-a-production project=team-a
kubectl label namespace team-a-staging project=team-a
kubectl label namespace team-a-dev project=team-a
```

## Step 5: Allow DNS Resolution

DNS is critical for service discovery. Always allow DNS traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: team-a-production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

## Step 6: Allow Ingress Controller Traffic

External traffic typically enters through an ingress controller in a shared namespace. Allow it:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: team-a-production
spec:
  podSelector:
    matchLabels:
      app: web-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: shared-platform
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
```

## Step 7: Allow Monitoring and Logging

Monitoring systems need to scrape metrics from all projects:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: team-a-production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              app: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
        - protocol: TCP
          port: 8080
```

## Step 8: Allow Specific Cross-Project Communication

Sometimes two projects need limited communication. Allow specific service access:

```yaml
# Allow team-b to access team-a's API service on port 8080 only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-team-b-api-access
  namespace: team-a-production
spec:
  podSelector:
    matchLabels:
      app: api-gateway
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: team-b
      ports:
        - protocol: TCP
          port: 8080
```

This allows team-b pods to reach team-a's API gateway on port 8080 but nothing else.

## Step 9: Allow External Traffic

Allow pods to reach external services (outside the cluster):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
  namespace: team-a-production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # Allow traffic to external IPs (outside cluster CIDRs)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
```

Adjust the except list to match your cluster's pod and service CIDRs.

## Step 10: Apply Network Policies Across All Namespaces

Use a script to apply consistent network policies across a project:

```bash
#!/bin/bash
# apply-network-isolation.sh

PROJECT_LABEL=$1  # e.g., "team-a"

NAMESPACES=$(kubectl get namespaces -l project=$PROJECT_LABEL -o jsonpath='{.items[*].metadata.name}')

for ns in $NAMESPACES; do
  echo "Applying network policies to: $ns"

  cat <<EOF | kubectl apply -f -
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: $ns
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# Allow intra-project
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-intra-project
  namespace: $ns
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: $PROJECT_LABEL
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              project: $PROJECT_LABEL
---
# Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: $ns
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - {protocol: UDP, port: 53}
        - {protocol: TCP, port: 53}
---
# Allow external egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external
  namespace: $ns
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except: [10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16]
EOF

  echo "  Done."
done
```

## Step 11: Verify Network Isolation

Test that isolation is working:

```bash
# Deploy a test pod in team-a namespace
kubectl run nettest-a -n team-a-production --image=busybox --restart=Never -- sleep 3600

# Deploy a test service in team-b namespace
kubectl run nettest-b -n team-b-production --image=nginx --restart=Never
kubectl expose pod nettest-b -n team-b-production --port=80

# Wait for pods to be ready
kubectl wait --for=condition=Ready pod/nettest-a -n team-a-production --timeout=30s
kubectl wait --for=condition=Ready pod/nettest-b -n team-b-production --timeout=30s

# Test cross-project communication (should fail)
kubectl exec nettest-a -n team-a-production -- wget -qO- --timeout=3 \
  http://nettest-b.team-b-production.svc.cluster.local 2>&1
echo "Exit code: $?"

# Clean up
kubectl delete pod nettest-a -n team-a-production
kubectl delete pod nettest-b -n team-b-production
kubectl delete service nettest-b -n team-b-production
```

## Troubleshooting

**Pods cannot communicate within the same project:**

```bash
# Check that namespace labels match the NetworkPolicy selectors
kubectl get namespace <ns> --show-labels

# Verify the NetworkPolicy is applied
kubectl get networkpolicies -n <ns>
kubectl describe networkpolicy allow-intra-project -n <ns>
```

**DNS resolution is failing:**

```bash
# Verify the DNS allow policy exists
kubectl get networkpolicy allow-dns -n <ns>

# Check that kube-dns pods have the expected labels
kubectl get pods -n kube-system -l k8s-app=kube-dns --show-labels
```

**External traffic is blocked:**

```bash
# Check the external egress policy
kubectl describe networkpolicy allow-external -n <ns>

# Verify the except list does not accidentally block the destination
```

## Best Practices

- **Start with audit**: Before enforcing, monitor traffic patterns to understand what communication is needed.
- **Always allow DNS**: Forgetting DNS is the most common network isolation mistake.
- **Use namespace labels**: Label namespaces with project identifiers for clean policy selectors.
- **Be specific with cross-project access**: When allowing cross-project traffic, limit it to specific pods and ports.
- **Test after changes**: Verify network connectivity after every policy change.
- **Version control policies**: Store NetworkPolicy manifests in Git for auditability.
- **Use Rancher's built-in isolation as a starting point**: Then layer custom policies on top for specific needs.

## Conclusion

Network isolation between Rancher projects is a critical security measure that prevents unauthorized communication between teams' workloads. Start with Rancher's built-in project network isolation, then add custom NetworkPolicy resources for fine-grained control. Always verify your policies with actual connectivity tests and maintain them as code in version control.
