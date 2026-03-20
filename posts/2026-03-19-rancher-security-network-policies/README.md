# How to Set Up Network Policies for Security in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, Network Policies

Description: Learn how to implement Kubernetes Network Policies in Rancher-managed clusters to control pod-to-pod and external traffic for improved security.

By default, Kubernetes allows all pods to communicate with each other without restrictions. Network Policies let you define rules that control ingress and egress traffic at the pod level, implementing micro-segmentation for your cluster. This guide covers creating and managing Network Policies in Rancher.

## Prerequisites

- Rancher v2.5 or later
- A CNI plugin that supports Network Policies (Calico, Canal, Cilium, or Weave Net)
- kubectl access with admin privileges
- Basic understanding of Kubernetes networking

## Step 1: Verify CNI Support

Not all CNI plugins support Network Policies. Verify your cluster uses a compatible CNI:

```bash
kubectl get pods -n kube-system | grep -E "calico|canal|cilium|weave"
```

For RKE2 clusters, Canal (Calico + Flannel) is the default CNI and supports Network Policies.

When creating a new cluster in Rancher, select a CNI that supports Network Policies:

1. Go to **Cluster Management** > **Create**.
2. Under **Network Provider**, select **Canal** or **Calico**.

## Step 2: Create a Default Deny Policy

Start with a default deny policy to block all traffic, then selectively allow what is needed:

```yaml
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

Apply to all application namespaces:

```bash
kubectl apply -f default-deny.yaml
```

After applying this, no pods in the namespace can send or receive traffic unless explicitly allowed.

## Step 3: Allow DNS Resolution

Pods need DNS access to resolve service names. Allow egress to the kube-dns service:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

## Step 4: Allow Application-Specific Traffic

Create policies for your applications. Here is an example for a typical web application:

### Allow frontend to receive external traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
```

### Allow frontend to talk to the API:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 8080
```

### Allow API to talk to the database:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Allow database to receive connections from API only:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-from-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
```

## Step 5: Isolate Namespaces

Prevent pods from communicating across namespaces except where explicitly allowed:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
```

This allows all traffic within the namespace but blocks traffic from other namespaces.

## Step 6: Allow Rancher System Traffic

Ensure Rancher system components can still communicate with your pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-rancher-system
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: cattle-system
    - namespaceSelector:
        matchLabels:
          name: cattle-monitoring-system
```

## Step 7: Manage Network Policies via Rancher UI

Rancher provides a UI for managing Network Policies:

1. Navigate to the cluster.
2. Go to **Policy** > **Network Policies**.
3. Click **Create** to add a new policy.
4. Use the form to define selectors and rules.
5. Review the generated YAML and save.

## Step 8: Block External Egress

Prevent pods from reaching the internet unless explicitly needed:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
    - ipBlock:
        cidr: 172.16.0.0/12
    - ipBlock:
        cidr: 192.168.0.0/16
```

This allows traffic only to private IP ranges, effectively blocking internet access.

## Step 9: Test Network Policies

Verify your policies are working correctly:

```bash
# Test connectivity between pods

kubectl run test-client --rm -it --image=busybox --restart=Never -n production -- \
  wget -qO- --timeout=3 http://api-service:8080/health

# Test blocked connectivity
kubectl run test-client --rm -it --image=busybox --restart=Never -n staging -- \
  wget -qO- --timeout=3 http://api-service.production:8080/health
```

The second command should time out if cross-namespace traffic is blocked.

## Step 10: Monitor Network Policy Impact

Use Calico or Cilium flow logs to see which connections are being blocked:

```bash
# For Calico
kubectl logs -n kube-system -l k8s-app=calico-node | grep -i denied

# For Cilium
kubectl exec -n kube-system -l k8s-app=cilium -- cilium monitor --type drop
```

## Best Practices

- Start with default deny and add allow rules incrementally.
- Always allow DNS egress in every namespace with Network Policies.
- Test policies in a staging environment before applying to production.
- Use labels consistently to make policies readable and maintainable.
- Allow Rancher system traffic to avoid breaking cluster management.
- Document all Network Policies and their intended purpose.
- Regularly review and audit policies for accuracy.

## Conclusion

Network Policies are a fundamental security layer for Kubernetes clusters managed by Rancher. By implementing default deny policies and selectively allowing only required traffic, you create micro-segmentation that limits the blast radius of any security breach. Start with basic namespace isolation and progressively add granular pod-level rules as your security requirements mature.
