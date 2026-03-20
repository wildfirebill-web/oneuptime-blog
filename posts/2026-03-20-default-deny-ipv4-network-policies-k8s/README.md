# How to Implement Default Deny-All IPv4 Network Policies in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, NetworkPolicy, IPv4, Zero Trust, Security, Default Deny

Description: Implement default deny-all NetworkPolicy for both ingress and egress in a Kubernetes namespace to enforce zero-trust pod networking.

A default-deny policy blocks all traffic to and from pods in a namespace unless explicitly allowed by another NetworkPolicy. This zero-trust approach ensures only approved communication paths exist.

## Default Deny All Ingress

```yaml
# default-deny-ingress.yaml
# Applies to ALL pods in the namespace (empty podSelector = all pods)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  # Selects all pods in the namespace
  podSelector: {}
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress
```

## Default Deny All Egress

```yaml
# default-deny-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny all egress
```

## Combined Deny All (Ingress + Egress)

```yaml
# default-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## Apply to Multiple Namespaces

```bash
# Apply default deny to all application namespaces
for ns in production staging development; do
  kubectl apply -f default-deny-all.yaml -n $ns
  echo "Applied default-deny to namespace: $ns"
done
```

## Allow DNS After Default Deny

After applying egress deny-all, DNS will stop working. Allow it explicitly:

```yaml
# allow-dns-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS to CoreDNS in kube-system
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

## Example: Allow Only Specific App Communication

After default-deny, explicitly allow necessary paths:

```yaml
# allow-api-to-db.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api
    ports:
    - protocol: TCP
      port: 5432
```

## Verifying Default Deny Works

```bash
# Deploy a test pod
kubectl run test-deny --image=alpine -n production --restart=Never -- sleep 3600

# Try to connect to another pod (should fail after default-deny)
kubectl exec -it test-deny -n production -- wget -qO- --timeout=3 http://my-service
# Expected: Connection timed out

# Verify the policies are applied
kubectl get networkpolicy -n production
# NAME                  POD-SELECTOR   AGE
# default-deny-all      <none>         5m
# allow-dns-egress      <none>         4m
```

Default-deny policies are the foundation of a zero-trust Kubernetes networking model. Apply them first, then progressively add allow rules for verified communication paths.
